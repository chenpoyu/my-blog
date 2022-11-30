---
layout: post
title: "配置中心 - Spring Cloud Config"
date: 2019-04-15 10:00:00 +0800
categories: [DevOps, Configuration Management]
tags: [Spring Cloud Config, Configuration, Microservices, Git]
---

上週研究了分散式追蹤（參考 [分散式追蹤 - Jaeger](/posts/2019/04/08/distributed-tracing-jaeger/)），這週來解決微服務的配置管理問題。

我們的痛點：

30 個微服務，每個都有 `application.yml`：
```yaml
# order-service/application.yml
database:
  url: jdbc:mysql://db-host:3306/orders
  username: orderuser
  password: SecretPassword123

redis:
  host: redis-host
  port: 6379

feature:
  newCheckout: false
```

問題：
- **配置散落各處**：要改資料庫密碼，要改 30 個檔案
- **無法動態更新**：改配置要重新部署
- **環境管理混亂**：dev/staging/prod 配置不同，容易搞混
- **敏感資訊外洩**：密碼寫在程式碼裡，push 到 Git
- **無版本控制**：不知道誰改了什麼配置
- **回滾困難**：配置出問題，怎麼回滾？

需要集中管理配置！

> 使用版本：Spring Cloud Config 2.1.0.RELEASE（配合 Spring Boot 2.1.x）

## Spring Cloud Config 是什麼

Spring Cloud Config 提供集中化的配置管理：

```
Git Repository ──> Config Server ──> 各個微服務
```

- **Config Server**：配置伺服器，從 Git 讀取配置
- **Config Client**：微服務，啟動時從 Config Server 取得配置
- **Git Repository**：存放所有配置檔案

優點：
- 集中管理
- 版本控制（Git）
- 環境隔離
- 動態刷新
- 加密支援

## 建立 Config Server

### 建立專案

`pom.xml`：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

`ConfigServerApplication.java`：
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### 設定

`application.yml`：
```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mycompany/config-repo
          search-paths: '{application}'
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
          clone-on-start: true
          default-label: master
  security:
    user:
      name: config
      password: ${CONFIG_SERVER_PASSWORD}
```

### 準備 Git Repository

建立 `config-repo` repository，結構：
```
config-repo/
├── order-service/
│   ├── application.yml           # 所有環境共用
│   ├── application-dev.yml       # 開發環境
│   ├── application-staging.yml   # 測試環境
│   └── application-prod.yml      # 正式環境
├── user-service/
│   ├── application.yml
│   ├── application-dev.yml
│   └── application-prod.yml
└── payment-service/
    ├── application.yml
    └── application-prod.yml
```

`order-service/application.yml`（共用配置）：
```yaml
server:
  port: 8080

spring:
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

logging:
  level:
    root: INFO
```

`order-service/application-dev.yml`（開發環境）：
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/orders_dev
    username: root
    password: dev123
  jpa:
    show-sql: true

logging:
  level:
    com.mycompany: DEBUG
```

`order-service/application-prod.yml`（正式環境）：
```yaml
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/orders
    username: orderuser
    password: '{cipher}AQA...'  # 加密後的密碼

logging:
  level:
    com.mycompany: INFO
```

### 啟動 Config Server

```bash
export GIT_USERNAME=myusername
export GIT_PASSWORD=mytoken
export CONFIG_SERVER_PASSWORD=configSecret123

mvn spring-boot:run
```

### 測試

```bash
# 取得 order-service 的 dev 配置
curl http://config:configSecret123@localhost:8888/order-service/dev

# 回傳
{
  "name": "order-service",
  "profiles": ["dev"],
  "label": null,
  "version": "abc123def456",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/mycompany/config-repo/order-service/application-dev.yml",
      "source": {
        "spring.datasource.url": "jdbc:mysql://localhost:3306/orders_dev",
        "spring.datasource.username": "root",
        "spring.datasource.password": "dev123",
        ...
      }
    },
    {
      "name": "https://github.com/mycompany/config-repo/order-service/application.yml",
      "source": {
        "server.port": 8080,
        ...
      }
    }
  ]
}
```

配置會合併：application.yml + application-dev.yml。

## 微服務整合 Config Client

### 加入依賴

`pom.xml`：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 設定

`bootstrap.yml`（比 application.yml 更早載入）：
```yaml
spring:
  application:
    name: order-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  cloud:
    config:
      uri: http://config-server:8888
      username: config
      password: configSecret123
      fail-fast: true
      retry:
        max-attempts: 6
        max-interval: 2000

management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info
```

刪除 `application.yml`（配置都從 Config Server 來）。

### 啟動服務

```bash
export SPRING_PROFILES_ACTIVE=dev
mvn spring-boot:run
```

服務啟動時：
1. 讀取 `bootstrap.yml`
2. 連線到 Config Server
3. 請求 `order-service/dev` 配置
4. 下載配置
5. 啟動應用

Log 會顯示：
```
Located property source: [BootstrapPropertySource {name='bootstrapProperties-https://github.com/mycompany/config-repo/order-service/application-dev.yml'}, ...]
```

## 動態刷新配置

不用重啟服務，動態更新配置。

### @RefreshScope

```java
@RestController
@RefreshScope  // 重要！
public class FeatureController {
    
    @Value("${feature.newCheckout}")
    private boolean newCheckoutEnabled;
    
    @GetMapping("/feature/checkout")
    public Map<String, Object> getCheckoutFeature() {
        return Map.of(
            "newCheckout", newCheckoutEnabled
        );
    }
}
```

### 刷新配置

1. **修改 Git 配置**

編輯 `order-service/application-dev.yml`：
```yaml
feature:
  newCheckout: true  # 改成 true
```

Commit 並 push。

2. **通知服務刷新**

```bash
curl -X POST http://localhost:8080/actuator/refresh \
  -H "Content-Type: application/json"

# 回傳被刷新的配置
["feature.newCheckout"]
```

3. **驗證**

```bash
curl http://localhost:8080/feature/checkout

# {"newCheckout":true}  ← 已更新，不用重啟！
```

### 自動刷新 - Webhook

手動刷新太麻煩，使用 Git Webhook 自動觸發。

#### Spring Cloud Bus

使用訊息佇列廣播刷新事件。

`pom.xml`：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

`bootstrap.yml`：
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh  # 新增 endpoint
```

架構：
```
Git Push --> Webhook --> Config Server /bus-refresh
                         ↓
                      RabbitMQ
                    ↙    ↓    ↘
              Service1  Service2  Service3
                (自動刷新配置)
```

#### 設定 GitHub Webhook

GitHub Repository → Settings → Webhooks → Add webhook：
```
Payload URL: http://config-server:8888/actuator/bus-refresh
Content type: application/json
Secret: (optional)
```

現在：
1. Push 配置到 Git
2. GitHub 自動觸發 Webhook
3. Config Server 收到，發送刷新事件到 RabbitMQ
4. 所有服務訂閱 RabbitMQ，收到後自動刷新

完全自動化！

## 加密敏感資訊

密碼不能明文存在 Git。

### 設定加密 Key

`config-server/application.yml`：
```yaml
encrypt:
  key: MyVerySecretEncryptionKey123456789012
```

或使用 Key Store（更安全）：
```yaml
encrypt:
  key-store:
    location: classpath:/config-server.jks
    password: keystorePassword
    alias: config-server
    secret: keyPassword
```

### 加密

```bash
curl http://localhost:8888/encrypt \
  -H "Content-Type: text/plain" \
  -d 'MySecretPassword123'

# 回傳加密後的值
AQAJxPEp8VfPl4gF7L4qqNEJSVvwvpvYNpK1G1cE6Q...
```

### 使用加密值

在 Git 配置檔案中：
```yaml
spring:
  datasource:
    password: '{cipher}AQAJxPEp8VfPl4gF7L4qqNEJSVvwvpvYNpK1G1cE6Q...'
```

`{cipher}` 前綴告訴 Config Server 這是加密值。

### 自動解密

Config Server 自動解密後傳給服務，服務收到的是明文：
```yaml
spring:
  datasource:
    password: MySecretPassword123
```

安全！密碼加密存在 Git，只有 Config Server 能解密。

## 多環境管理

### Profile

使用 Spring Profile 區分環境：

```bash
# 開發環境
export SPRING_PROFILES_ACTIVE=dev
java -jar order-service.jar

# 正式環境
export SPRING_PROFILES_ACTIVE=prod
java -jar order-service.jar
```

服務會自動載入對應的配置。

### Label（分支）

不同環境使用不同 Git 分支：

```
Git Repository:
├── master (正式環境)
├── staging (測試環境)
└── develop (開發環境)
```

`bootstrap.yml`：
```yaml
spring:
  cloud:
    config:
      label: ${CONFIG_LABEL:master}
```

部署時指定：
```bash
# 開發環境
CONFIG_LABEL=develop java -jar order-service.jar

# 正式環境
CONFIG_LABEL=master java -jar order-service.jar
```

### 配置優先順序

載入順序（後面的覆蓋前面的）：
1. `application.yml` (repository)
2. `application-{profile}.yml` (repository)
3. `{application}/application.yml` (repository)
4. `{application}/application-{profile}.yml` (repository)
5. `application.yml` (local)
6. `application-{profile}.yml` (local)

例如 order-service 的 dev 環境：
1. `application.yml`
2. `application-dev.yml`
3. `order-service/application.yml`
4. `order-service/application-dev.yml`
5. 本地 `application.yml`（如果有）

## 高可用

Config Server 是關鍵服務，必須高可用。

### 部署多個 instances

```yaml
# Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-server
spec:
  replicas: 3  # 至少 3 個
  selector:
    matchLabels:
      app: config-server
  template:
    metadata:
      labels:
        app: config-server
    spec:
      containers:
      - name: config-server
        image: config-server:1.0
        ports:
        - containerPort: 8888
---
apiVersion: v1
kind: Service
metadata:
  name: config-server
spec:
  selector:
    app: config-server
  ports:
  - port: 8888
```

### 客戶端配置多個 URI

```yaml
spring:
  cloud:
    config:
      uri: http://config-server1:8888,http://config-server2:8888,http://config-server3:8888
      fail-fast: true
```

如果一個掛了，自動嘗試下一個。

### 本地快取

服務啟動時下載配置並快取到本地，Config Server 掛了還能啟動。

```yaml
spring:
  cloud:
    config:
      fail-fast: false  # Config Server 不可用時，使用本地快取
```

## 實際案例

### 案例一：Feature Toggle

透過配置控制功能開關，不用重新部署。

`config-repo/order-service/application.yml`：
```yaml
feature:
  newCheckout: false
  paymentV2: false
  recommendationEngine: true
```

程式碼：
```java
@Service
@RefreshScope
public class OrderService {
    
    @Value("${feature.newCheckout}")
    private boolean newCheckoutEnabled;
    
    public OrderResponse createOrder(OrderRequest request) {
        if (newCheckoutEnabled) {
            return newCheckoutFlow(request);
        } else {
            return oldCheckoutFlow(request);
        }
    }
}
```

上線新功能：
1. 部署程式碼（包含新舊兩套邏輯）
2. 先不開啟功能（`newCheckout: false`）
3. 測試環境開啟（`newCheckout: true`）
4. 測試通過，正式環境開啟
5. 出問題立刻關閉（改回 `false`，刷新配置）

不用重新部署！

### 案例二：A/B Testing

根據使用者 ID 決定功能開關。

`application.yml`：
```yaml
feature:
  newCheckout:
    enabled: true
    percentage: 10  # 10% 使用者
```

程式碼：
```java
@Service
@RefreshScope
public class FeatureToggle {
    
    @Value("${feature.newCheckout.enabled}")
    private boolean enabled;
    
    @Value("${feature.newCheckout.percentage}")
    private int percentage;
    
    public boolean isNewCheckoutEnabled(String userId) {
        if (!enabled) {
            return false;
        }
        
        // 根據 userId hash 決定
        int hash = Math.abs(userId.hashCode() % 100);
        return hash < percentage;
    }
}
```

逐步增加比例：10% → 50% → 100%，觀察指標。

### 案例三：緊急調參數

Production 發現 connection pool 太小，連線不夠用。

以前：改程式碼 → 重新部署 → 等待重啟（10 分鐘）

現在：
1. 改配置（1 分鐘）
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50  # 原本 20
```

2. Commit & Push
3. 自動刷新（30 秒）

總共 2 分鐘解決！

## 遇到的問題

### 問題一：Config Server 掛了，服務無法啟動

解決方法：
1. 高可用部署（多個 instances）
2. 啟用本地快取
3. 設定 `fail-fast: false`

### 問題二：Git Repository 太大

所有服務的配置都在一個 repo，越來越大。

解決方法：
1. 拆分 Repository（按團隊或業務領域）
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mycompany/config-{application}
          # order-service 會去 config-order-service repository
```

2. 使用 shallow clone
```yaml
spring:
  cloud:
    config:
      server:
        git:
          clone-on-start: true
          depth: 1  # 只 clone 最新一層
```

### 問題三：刷新配置後，某些 Bean 沒更新

只有 `@RefreshScope` 的 Bean 會刷新，其他不會。

解決：
1. 加 `@RefreshScope` 到需要刷新的 Bean
2. 或實作 `@ConfigurationProperties`
```java
@Component
@ConfigurationProperties(prefix = "feature")
@RefreshScope
public class FeatureConfig {
    private boolean newCheckout;
    private boolean paymentV2;
    // getters and setters
}
```

## 心得

Spring Cloud Config 真的讓配置管理變簡單了。以前要改個資料庫密碼，要改 30 個 `application.yml`，現在只要改 Git 裡的配置，所有服務自動更新。

特別喜歡動態刷新功能，Feature Toggle 可以隨時開關，不用重新部署。有次線上出問題，關掉有問題的功能，2 分鐘就恢復服務，如果要重新部署可能要 30 分鐘。

加密功能也很實用，密碼不用明文存在 Git，安全多了。

不過要注意：
- 不是所有配置都適合動態刷新（例如 server.port）
- 配置變更要謹慎，可能影響所有服務
- 要有回滾機制（Git revert）

下週要研究服務網格的流量管理，搭配 Istio 實現更精細的流量控制。
