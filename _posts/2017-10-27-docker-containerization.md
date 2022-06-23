---
layout: post
title: "Docker 容器化筆記"
date: 2017-10-27 16:15:00 +0800
categories: [設計模式, Java]
tags: [Docker, 容器化, DevOps, 部署]
---

上週我們學習了微服務架構(請參考 [Spring Cloud 微服務](/posts/2017/10/20/spring-cloud-microservices/)),但如何快速部署這麼多服務?Docker 提供了有效的解決方案。

## 為什麼需要 Docker?

### 傳統部署的問題
```
開發環境: Windows + JDK 8 + MySQL 5.7 (成功) 正常
測試環境: Linux + JDK 8 + MySQL 5.7 (失敗) 出錯
生產環境: Linux + JDK 8 + MySQL 5.7 (不確定) 不確定
```

「在我機器上可以跑啊!」

### Docker 的優勢
- **環境一致**:開發、測試、生產使用相同映像檔
- **快速部署**:秒級啟動,不像虛擬機要幾分鐘
- **資源隔離**:每個容器獨立執行
- **版本管理**:映像檔可版本控制和回滾

## Docker 基礎概念

```
Docker Image (映像檔) → 靜態模板
     ↓
Docker Container (容器) → 執行中的實例
```

就像「類別」和「物件」的關係!

## Spring Boot 應用容器化

### 1. 建立 Dockerfile

```dockerfile
# 基礎映像檔
FROM openjdk:8-jre-slim

# 設定工作目錄
WORKDIR /app

# 複製 jar 檔
COPY target/product-service-1.0.0.jar app.jar

# 暴露端口
EXPOSE 8080

# 啟動命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2. 建構映像檔

```bash
# 先打包 Spring Boot 應用
mvn clean package

# 建構 Docker 映像檔
docker build -t product-service:1.0.0 .

# 查看映像檔
docker images
```

### 3. 執行容器

```bash
# 執行容器
docker run -d \
  --name product-service \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e MYSQL_HOST=mysql \
  product-service:1.0.0

# 查看執行中的容器
docker ps

# 查看日誌
docker logs -f product-service

# 進入容器
docker exec -it product-service /bin/bash

# 停止容器
docker stop product-service

# 刪除容器
docker rm product-service
```

## 實戰案例:電商系統容器化

### 優化的 Dockerfile

```dockerfile
# 多階段建構
FROM maven:3.8-openjdk-11 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# 執行階段
FROM openjdk:8-jre-slim
WORKDIR /app

# 建立非 root 使用者
RUN groupadd -r spring && useradd -r -g spring spring
USER spring:spring

# 複製 jar 檔
COPY --from=build /app/target/*.jar app.jar

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# JVM 參數優化
ENV JAVA_OPTS="-Xms512m -Xmx1024m -XX:+UseG1GC"

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Docker Compose 編排

```yaml
version: '3.8'

services:
  # MySQL 資料庫
  mysql:
    image: mysql:5.7
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: ecommerce
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network

  # Redis 快取
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - app-network

  # Eureka 註冊中心
  eureka-server:
    build: ./eureka-server
    container_name: eureka-server
    ports:
      - "8761:8761"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # 商品服務
  product-service:
    build: ./product-service
    container_name: product-service
    depends_on:
      - mysql
      - redis
      - eureka-server
    environment:
      SPRING_PROFILES_ACTIVE: docker
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/ecommerce
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root123
      SPRING_REDIS_HOST: redis
    ports:
      - "8081:8080"
    networks:
      - app-network
    restart: on-failure

  # 訂單服務
  order-service:
    build: ./order-service
    container_name: order-service
    depends_on:
      - mysql
      - redis
      - eureka-server
      - product-service
    environment:
      SPRING_PROFILES_ACTIVE: docker
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/ecommerce
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root123
      SPRING_REDIS_HOST: redis
    ports:
      - "8082:8080"
    networks:
      - app-network
    restart: on-failure

  # API 閘道
  gateway:
    build: ./gateway
    container_name: gateway
    depends_on:
      - eureka-server
    environment:
      SPRING_PROFILES_ACTIVE: docker
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
    ports:
      - "8080:8080"
    networks:
      - app-network
    restart: on-failure

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
```

### 啟動整個系統

```bash
# 建構並啟動所有服務
docker-compose up -d

# 查看執行狀態
docker-compose ps

# 查看日誌
docker-compose logs -f product-service

# 擴展服務實例
docker-compose up -d --scale product-service=3

# 停止所有服務
docker-compose down

# 停止並刪除 volumes
docker-compose down -v
```

## Docker 網路

```bash
# 建立自訂網路
docker network create my-network

# 將容器加入網路
docker run -d --name mysql --network my-network mysql:5.7

# 容器間可以透過服務名稱通訊
docker run -d --name app --network my-network \
  -e DB_HOST=mysql \
  product-service:1.0.0
```

## Docker Volume 資料持久化

```bash
# 建立 volume
docker volume create mysql-data

# 使用 volume
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  mysql:5.7

# 使用本機目錄
docker run -d \
  -v /local/path:/container/path \
  product-service:1.0.0
```

## 多階段建構最佳化

```dockerfile
# ===== 依賴下載階段 =====
FROM maven:3.8-openjdk-11 AS dependencies
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

# ===== 編譯階段 =====
FROM dependencies AS build
COPY src ./src
RUN mvn package -DskipTests

# ===== 執行階段 =====
FROM openjdk:8-jre-slim
WORKDIR /app

# 只複製需要的 jar 檔
COPY --from=build /app/target/*.jar app.jar

# 使用 exec 形式避免殭屍程序
ENTRYPOINT ["java", "-jar", "app.jar"]
```

優點:
- 最終映像檔只包含 JRE 和 jar,體積小
- Maven 依賴層可以被快取,建構更快

## 環境配置

### application-docker.yml

```yaml
spring:
  datasource:
    url: jdbc:mysql://${MYSQL_HOST:mysql}:3306/${MYSQL_DATABASE:ecommerce}
    username: ${MYSQL_USERNAME:root}
    password: ${MYSQL_PASSWORD:root123}
    
  redis:
    host: ${REDIS_HOST:redis}
    port: ${REDIS_PORT:6379}
    
eureka:
  client:
    service-url:
      defaultzone: ${EUREKA_SERVER:http://eureka-server:8761/eureka/}
  instance:
    prefer-ip-address: true
    ip-address: ${HOSTNAME}
```

## 健康檢查

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

Docker Compose:
```yaml
services:
  product-service:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

## 監控與日誌

### 查看容器資源使用

```bash
# 即時監控
docker stats

# 查看特定容器
docker stats product-service
```

### 日誌管理

```bash
# 查看日誌
docker logs product-service

# 持續跟蹤日誌
docker logs -f product-service

# 只看最後 100 行
docker logs --tail 100 product-service

# 查看特定時間的日誌
docker logs --since 2023-01-01T00:00:00 product-service
```

### 日誌驅動

```yaml
services:
  product-service:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## Docker 實踐經驗

1. **使用官方映像檔**:更安全、更新及時
2. **多階段建構**:減少映像檔大小
3. **.dockerignore**:排除不需要的檔案
4. **快取友善**:先複製 pom.xml,再複製原始碼
5. **非 root 使用者**:提高安全性
6. **健康檢查**:確保容器健康狀態
7. **版本標籤**:不要只用 latest
8. **環境變數**:配置外部化

### .dockerignore

```
target/
.git/
.idea/
*.log
*.tmp
```

## 小結

Docker 為微服務提供了:
- **一致的執行環境**:消除「在我機器上可以跑」
- **快速部署**:秒級啟動
- **資源隔離**:互不干擾
- **易於擴展**:輕鬆增加實例

Docker + Spring Boot + Spring Cloud = 現代微服務架構的黃金組合!

下週我們將學習 **Kubernetes**,看看如何在生產環境編排大量容器。
