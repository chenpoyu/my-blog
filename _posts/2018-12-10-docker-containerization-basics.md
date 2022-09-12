---
layout: post
title: "Docker 容器化入門實戰"
date: 2018-12-10 11:20:00 +0800
categories: [DevOps, Container]
tags: [Docker, Container, Virtualization]
---

前幾週建立了 Jenkins CI/CD 環境，但還是會遇到環境不一致的問題。「在我機器上可以跑」這句話已經聽膩了。這週來研究 Docker，徹底解決環境一致性的問題。

> 使用版本：Docker CE 18.09（2018 年底的穩定版）

## 環境不一致的惡夢

上個月發生的慘案：

**開發環境（我的 Mac）**
- macOS 10.14
- Java 8u191
- MySQL 5.7.24
- Redis 4.0.11

**測試環境（Linux 伺服器）**
- Ubuntu 16.04
- Java 8u181（舊版本）
- MySQL 5.6.42（舊版本！）
- Redis 3.2.12（超舊！）

結果：
1. 本機測試完全正常
2. 部署到測試環境，應用程式啟動失敗
3. 查半天發現是 MySQL 版本不同，SQL 語法有差異
4. 改完 SQL 又發現 Redis 的某個指令在舊版不支援
5. 最後花了一整天處理環境問題

主管問：「這個功能不是測試過了嗎？」
我：「在我本機測試過了...」（又是這句話）

## Docker 是什麼

Docker 是一種容器技術，可以把應用程式和它需要的所有東西（執行環境、函式庫、設定）打包成一個獨立的容器。

**傳統虛擬機 vs Docker 容器**

虛擬機：
```
[應用程式] [應用程式]
[Guest OS] [Guest OS]
[Hypervisor]
[Host OS]
[硬體]
```
- 每個 VM 都要完整的 OS
- 啟動慢（分鐘級）
- 資源消耗大（GB 級）

Docker 容器：
```
[容器1] [容器2] [容器3]
[Docker Engine]
[Host OS]
[硬體]
```
- 共享 Host OS 的 kernel
- 啟動快（秒級）
- 資源消耗小（MB 級）

## 安裝 Docker

### macOS 安裝

下載 Docker Desktop for Mac：
```bash
brew cask install docker
```

或從官網下載 dmg 檔案安裝。

安裝後，在系統列會看到 Docker 圖示。點擊後選擇「Preferences」→「Resources」，調整記憶體配置（建議 4GB）。

驗證安裝：
```bash
docker version
# Client: Docker Engine - Community
#  Version:           18.09.0
#  API version:       1.39

docker run hello-world
# Hello from Docker!
```

## Docker 核心概念

### Image（映像檔）

映像檔就像是一個模板，包含執行應用程式需要的所有東西。

查看本機的映像檔：
```bash
docker images
```

下載映像檔：
```bash
docker pull nginx
docker pull mysql:5.7
docker pull redis:4.0
```

### Container（容器）

容器是映像檔的執行實例。一個映像檔可以啟動多個容器。

執行容器：
```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

參數說明：
- `-d`：背景執行
- `--name`：容器名稱
- `-p 8080:80`：port mapping（本機 8080 → 容器 80）

現在訪問 `http://localhost:8080` 就可以看到 Nginx 的歡迎頁面！

查看運行中的容器：
```bash
docker ps

# CONTAINER ID   IMAGE   COMMAND                  STATUS
# a1b2c3d4e5f6   nginx   "nginx -g 'daemon ..."   Up 2 minutes
```

停止容器：
```bash
docker stop my-nginx
```

刪除容器：
```bash
docker rm my-nginx
```

## 容器化 Spring Boot 應用

現在來把我們的 Spring Boot API 容器化。

### 建立 Dockerfile

在專案根目錄建立 `Dockerfile`：

```dockerfile
FROM openjdk:8-jre-alpine

# 設定工作目錄
WORKDIR /app

# 複製 jar 檔
COPY target/my-shop-api-1.0.0.jar app.jar

# 設定環境變數
ENV JAVA_OPTS="-Xmx512m -Xms256m"

# 暴露 port
EXPOSE 8080

# 啟動指令
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### 建置映像檔

```bash
# 先用 Maven 打包
mvn clean package

# 建置 Docker 映像檔
docker build -t my-shop-api:1.0.0 .

# 查看建立的映像檔
docker images | grep my-shop-api
```

### 執行容器

```bash
docker run -d \
  --name my-shop-api \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=dev \
  my-shop-api:1.0.0
```

查看日誌：
```bash
docker logs -f my-shop-api
```

測試 API：
```bash
curl http://localhost:8080/api/health
# {"status":"UP"}
```

成功！

## 多容器應用：加入 MySQL 和 Redis

我們的應用需要 MySQL 和 Redis，可以用 Docker 一起管理。

### 建立 Docker network

讓容器之間可以互相通訊：

```bash
docker network create my-shop-network
```

### 啟動 MySQL

```bash
docker run -d \
  --name mysql \
  --network my-shop-network \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=myshop \
  -e MYSQL_USER=shopuser \
  -e MYSQL_PASSWORD=shoppass \
  -p 3306:3306 \
  mysql:5.7
```

等待 MySQL 初始化完成（約 30 秒）：
```bash
docker logs -f mysql
# mysqld: ready for connections
```

### 啟動 Redis

```bash
docker run -d \
  --name redis \
  --network my-shop-network \
  -p 6379:6379 \
  redis:4.0
```

### 重新啟動應用連接資料庫

修改 `application-dev.properties`：
```properties
spring.datasource.url=jdbc:mysql://mysql:3306/myshop
spring.datasource.username=shopuser
spring.datasource.password=shoppass

spring.redis.host=redis
spring.redis.port=6379
```

重新建置並啟動：
```bash
mvn clean package
docker build -t my-shop-api:1.0.0 .

docker run -d \
  --name my-shop-api \
  --network my-shop-network \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=dev \
  my-shop-api:1.0.0
```

測試連接：
```bash
curl http://localhost:8080/api/products
# 成功取得商品列表！
```

## 常用 Docker 指令

### 容器管理

```bash
# 列出所有容器（包含停止的）
docker ps -a

# 進入容器內部
docker exec -it my-shop-api sh

# 查看容器詳細資訊
docker inspect my-shop-api

# 查看容器資源使用
docker stats my-shop-api

# 複製檔案到容器
docker cp config.yml my-shop-api:/app/

# 從容器複製檔案
docker cp my-shop-api:/app/logs/app.log ./
```

### 映像檔管理

```bash
# 刪除映像檔
docker rmi my-shop-api:1.0.0

# 清理未使用的映像檔
docker image prune

# 查看映像檔層級
docker history my-shop-api:1.0.0

# 匯出映像檔
docker save -o my-shop-api.tar my-shop-api:1.0.0

# 匯入映像檔
docker load -i my-shop-api.tar
```

### 清理指令

```bash
# 停止所有容器
docker stop $(docker ps -aq)

# 刪除所有停止的容器
docker container prune

# 刪除所有未使用的資源（容器、網路、映像檔）
docker system prune -a
```

## Dockerfile 最佳實踐

### 使用 .dockerignore

建立 `.dockerignore` 檔案：
```
target/
.git
.idea
*.log
node_modules/
```

### 多階段建置

把編譯和執行分開，減少最終映像檔大小：

```dockerfile
# 建置階段
FROM maven:3.5.4-jdk-8 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# 執行階段
FROM openjdk:8-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

這樣最終映像檔只包含 JRE 和 JAR 檔，不包含 Maven 和原始碼。

映像檔大小比較：
- 原本：600+ MB
- 優化後：120 MB

## 遇到的問題

### 問題一：容器無法連接 MySQL

錯誤訊息：`Connection refused`

原因：容器啟動太快，MySQL 還沒準備好。

解決方法：加入健康檢查和重試機制。

```yaml
# 在 application.yml 中
spring:
  datasource:
    hikari:
      connection-timeout: 60000
      initialization-fail-timeout: 0
```

或使用 `wait-for-it.sh` 腳本等待 MySQL 啟動。

### 問題二：映像檔太大

原始映像檔 600MB，太大了。

解決方法：
1. 使用 alpine 基底映像檔（`openjdk:8-jre-alpine`）
2. 使用多階段建置
3. 清理不必要的檔案

### 問題三：時區錯誤

容器內的時區是 UTC，日誌時間不對。

解決方法：

```dockerfile
FROM openjdk:8-jre-alpine

# 設定時區
RUN apk add --no-cache tzdata
ENV TZ=Asia/Taipei

# 其他設定...
```

## 心得

Docker 真的解決了「在我機器上可以跑」的問題。現在：

1. 開發環境：每個人都用同樣的 Docker 映像檔
2. CI 環境：Jenkins 也用同樣的映像檔建置
3. 測試/生產環境：部署同樣的映像檔

環境完全一致！而且啟動速度很快，不用再手動安裝 MySQL、Redis 了。

但是手動管理多個容器還是有點麻煩，下週要研究 Docker Compose，可以用一個檔案定義整個應用的所有服務。
