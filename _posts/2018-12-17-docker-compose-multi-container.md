---
layout: post
title: "Docker Compose 多容器編排"
date: 2018-12-17 10:30:00 +0800
categories: [DevOps, Container]
tags: [Docker, Docker Compose, Orchestration]
---

上週學會了 Docker 容器化（參考 [Docker 容器化入門實戰](/posts/2018/12/10/docker-containerization-basics/)），但手動管理多個容器實在太麻煩了。每次啟動都要執行一堆 `docker run` 指令，還要記得正確的順序和參數。

這週來研究 Docker Compose，用一個 YAML 檔案管理所有容器。

> 使用版本：Docker Compose 1.23.2

## 手動管理容器的痛苦

每次要啟動完整環境，都要執行：

```bash
# 1. 建立網路
docker network create my-shop-network

# 2. 啟動 MySQL
docker run -d --name mysql --network my-shop-network \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=myshop \
  -p 3306:3306 mysql:5.7

# 3. 等 MySQL 啟動完成（30 秒）

# 4. 啟動 Redis
docker run -d --name redis --network my-shop-network \
  -p 6379:6379 redis:4.0

# 5. 建置應用程式映像檔
mvn clean package
docker build -t my-shop-api:1.0.0 .

# 6. 啟動應用程式
docker run -d --name my-shop-api --network my-shop-network \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=dev \
  my-shop-api:1.0.0
```

總共 6 個步驟，一個不小心就會出錯。上週新來的同事按照我寫的文件操作，卡在第 3 步 — 他沒等 MySQL 啟動完成就執行第 6 步，應用程式起不來。

而且要停止環境也很麻煩：
```bash
docker stop my-shop-api redis mysql
docker rm my-shop-api redis mysql
docker network rm my-shop-network
```

## Docker Compose 是什麼

Docker Compose 是一個定義和執行多容器 Docker 應用程式的工具。

特色：
- **單一檔案定義**：所有服務都寫在 `docker-compose.yml`
- **一鍵啟動**：`docker-compose up` 搞定
- **依賴管理**：自動處理服務啟動順序
- **環境變數管理**：統一管理設定

## 安裝 Docker Compose

Docker Desktop for Mac 已經內建了，檢查版本：

```bash
docker-compose version
# docker-compose version 1.23.2, build 1110ad01
```

Linux 使用者需要另外安裝：
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## 第一個 docker-compose.yml

在專案根目錄建立 `docker-compose.yml`：

```yaml
version: '3.7'

services:
  mysql:
    image: mysql:5.7
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: myshop
      MYSQL_USER: shopuser
      MYSQL_PASSWORD: shoppass
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:4.0
    container_name: redis
    ports:
      - "6379:6379"

  app:
    build: .
    container_name: my-shop-api
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/myshop
      SPRING_DATASOURCE_USERNAME: shopuser
      SPRING_DATASOURCE_PASSWORD: shoppass
      SPRING_REDIS_HOST: redis
    depends_on:
      - mysql
      - redis

volumes:
  mysql-data:
```

### 檔案結構說明

- `version: '3.7'`：使用 Compose file format 3.7 版
- `services`：定義所有服務
- `volumes`：定義持久化儲存

### 啟動所有服務

```bash
docker-compose up -d
```

就這樣！Docker Compose 會自動：
1. 建立網路
2. 啟動 MySQL 容器
3. 啟動 Redis 容器
4. 建置應用程式映像檔
5. 啟動應用程式容器
6. 處理服務間的連接

查看運行狀態：
```bash
docker-compose ps

#       Name                    State           Ports
# --------------------------------------------------------
# my-shop-api         Up      0.0.0.0:8080->8080/tcp
# mysql               Up      0.0.0.0:3306->3306/tcp
# redis               Up      0.0.0.0:6379->6379/tcp
```

查看日誌：
```bash
# 所有服務的日誌
docker-compose logs

# 特定服務的日誌
docker-compose logs app

# 即時追蹤日誌
docker-compose logs -f app
```

停止所有服務：
```bash
docker-compose down
```

這會停止並移除所有容器，但保留 volumes（資料不會遺失）。

## docker-compose.yml 詳解

### 服務定義

每個服務可以設定：

```yaml
services:
  app:
    # 使用現成的映像檔
    image: nginx:latest
    
    # 或從 Dockerfile 建置
    build:
      context: .
      dockerfile: Dockerfile
    
    # 容器名稱
    container_name: my-app
    
    # Port mapping
    ports:
      - "8080:80"      # host:container
      - "8443:443"
    
    # 環境變數
    environment:
      ENV: production
      DEBUG: "false"
    
    # 或從檔案讀取
    env_file:
      - .env
    
    # Volume mapping
    volumes:
      - ./data:/app/data           # 綁定目錄
      - app-logs:/app/logs         # 命名 volume
      - /etc/timezone:/etc/timezone:ro  # 唯讀
    
    # 依賴關係
    depends_on:
      - mysql
      - redis
    
    # 重啟策略
    restart: always
    
    # 健康檢查
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### 網路設定

預設情況下，Compose 會建立一個網路，所有服務都連接到這個網路。

自訂網路：

```yaml
services:
  app:
    networks:
      - frontend
      - backend
  
  mysql:
    networks:
      - backend

networks:
  frontend:
  backend:
```

### Volumes

持久化資料儲存：

```yaml
services:
  mysql:
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  mysql-data:
    driver: local
```

Volume 的資料在容器刪除後仍然保留。

## 實際專案的 docker-compose.yml

我們完整專案的配置：

```yaml
version: '3.7'

services:
  # MySQL 資料庫
  mysql:
    image: mysql:5.7
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-rootpass}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-myshop}
      MYSQL_USER: ${MYSQL_USER:-shopuser}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-shoppass}
      TZ: Asia/Taipei
    volumes:
      - mysql-data:/var/lib/mysql
      - ./docker/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  # Redis 快取
  redis:
    image: redis:4.0-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  # Spring Boot 應用程式
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: my-shop-api
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: ${ENV:-dev}
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/${MYSQL_DATABASE:-myshop}?useSSL=false&serverTimezone=Asia/Taipei
      SPRING_DATASOURCE_USERNAME: ${MYSQL_USER:-shopuser}
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_PASSWORD:-shoppass}
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
      JAVA_OPTS: -Xmx512m -Xms256m
    volumes:
      - ./logs:/app/logs
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    networks:
      - backend
    restart: unless-stopped

networks:
  backend:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

### 使用 .env 檔案

建立 `.env` 檔案管理環境變數：

```bash
# .env
ENV=dev
MYSQL_ROOT_PASSWORD=my_secure_password
MYSQL_DATABASE=myshop
MYSQL_USER=shopuser
MYSQL_PASSWORD=shop_secure_pass
```

記得把 `.env` 加入 `.gitignore`！

不同環境可以用不同的 env 檔案：
```bash
# 開發環境
docker-compose --env-file .env.dev up

# 生產環境
docker-compose --env-file .env.prod up
```

## 常用 Docker Compose 指令

### 啟動和停止

```bash
# 啟動所有服務（背景執行）
docker-compose up -d

# 啟動特定服務
docker-compose up -d mysql redis

# 前景執行（看日誌）
docker-compose up

# 停止所有服務
docker-compose stop

# 停止並移除容器、網路
docker-compose down

# 停止並移除容器、網路、volumes（會刪除資料！）
docker-compose down -v
```

### 檢視狀態

```bash
# 列出所有服務
docker-compose ps

# 查看服務日誌
docker-compose logs
docker-compose logs -f app

# 查看特定服務的最近 100 行日誌
docker-compose logs --tail=100 app

# 執行指令
docker-compose exec app sh
docker-compose exec mysql mysql -u root -p
```

### 建置和更新

```bash
# 重新建置映像檔
docker-compose build

# 重新建置特定服務
docker-compose build app

# 不使用快取建置
docker-compose build --no-cache

# 重新建置並啟動
docker-compose up -d --build
```

### 擴展服務

```bash
# 啟動 3 個 app 實例
docker-compose up -d --scale app=3

# 查看
docker-compose ps
#       Name          State     Ports
# ------------------------------------
# myshop_app_1       Up        0.0.0.0:8080->8080/tcp
# myshop_app_2       Up        0.0.0.0:8081->8080/tcp
# myshop_app_3       Up        0.0.0.0:8082->8080/tcp
```

## 開發環境最佳化

### 熱重載（Hot Reload）

開發時不想每次改 code 都要重新建置映像檔。

```yaml
services:
  app:
    build: .
    volumes:
      # 將原始碼掛載進容器
      - ./src:/app/src
      # Spring Boot DevTools 會偵測變更並重啟
    environment:
      SPRING_DEVTOOLS_RESTART_ENABLED: "true"
```

### 多個 Compose 檔案

基礎配置：`docker-compose.yml`
```yaml
version: '3.7'
services:
  mysql:
    image: mysql:5.7
    # 基本設定...
```

開發環境覆蓋：`docker-compose.dev.yml`
```yaml
version: '3.7'
services:
  mysql:
    ports:
      - "3306:3306"  # 開發時暴露 port
  app:
    volumes:
      - ./src:/app/src  # 開發時掛載原始碼
    environment:
      SPRING_DEVTOOLS_RESTART_ENABLED: "true"
```

使用：
```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
```

## 遇到的問題

### 問題一：depends_on 不等待服務就緒

`depends_on` 只確保容器啟動順序，不等待服務真的準備好。

MySQL 可能還在初始化，應用程式就開始連接了，結果失敗。

解決方法一：使用 `healthcheck`（較新版本）
```yaml
app:
  depends_on:
    mysql:
      condition: service_healthy
```

解決方法二：應用程式加入重試機制
```yaml
app:
  restart: on-failure
```

### 問題二：Volume 權限問題

在 Linux 上，容器內的檔案是 root 擁有的，本機無法修改。

解決方法：指定 user
```yaml
app:
  user: "${UID}:${GID}"
```

啟動時：
```bash
UID=$(id -u) GID=$(id -g) docker-compose up
```

### 問題三：Port 衝突

本機已經有服務在使用 3306 port。

```
Error: Bind for 0.0.0.0:3306 failed: port is already allocated
```

解決方法：換 port
```yaml
mysql:
  ports:
    - "3307:3306"  # 本機用 3307
```

應用程式連接改成 `mysql:3306`（容器內部還是 3306）。

## 實際效果

現在新同事要啟動開發環境：

```bash
git clone https://github.com/mycompany/my-shop-api.git
cd my-shop-api
cp .env.example .env  # 複製環境變數範本
docker-compose up -d
```

三個指令，搞定！不用安裝 Java、MySQL、Redis，不用煩惱版本問題。

需要清空資料庫重來：
```bash
docker-compose down -v  # 刪除 volumes
docker-compose up -d    # 重新啟動，資料庫會重新初始化
```

## 心得

Docker Compose 大幅簡化了多容器管理。以前每次要啟動完整環境都很痛苦，現在一個指令就搞定。

而且把所有設定都放在 `docker-compose.yml`，新人 onboarding 變得超簡單。不用再寫一大篇「環境安裝指南」，直接給他 `docker-compose up` 就好。

下週要研究 Docker Registry，把映像檔推到私有倉庫，這樣 CI/CD 和生產環境都能使用同一份映像檔。
