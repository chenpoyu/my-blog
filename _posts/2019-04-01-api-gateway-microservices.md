---
layout: post
title: "API Gateway 與微服務架構"
date: 2019-04-01 10:00:00 +0800
categories: [DevOps, Microservices]
tags: [API Gateway, Kong, Microservices, Rate Limiting, Authentication]
---

上週研究了部署策略（參考 [金絲雀部署和藍綠部署](/posts/2019/03/25/canary-blue-green-deployment/)），這週來解決微服務架構的入口問題。

我們的痛點：

目前有 30 個微服務，前端要呼叫不同的服務：
```javascript
// 前端要記住所有服務的位址
fetch('http://user-service:8080/api/users')
fetch('http://order-service:8080/api/orders')
fetch('http://payment-service:8080/api/payments')
```

問題：
- **前端要知道所有服務的位址**：服務搬家要改前端
- **認證邏輯重複**：每個服務都要驗證 JWT
- **CORS 設定麻煩**：30 個服務都要設定 CORS
- **Rate Limiting 難統一**：每個服務自己限流
- **監控分散**：難以統一監控所有 API 呼叫
- **版本管理混亂**：每個服務各自版本

需要一個統一的入口：API Gateway！

> 使用版本：Kong 1.0.3（2019 年初）、Spring Cloud Gateway 2.1.0

## API Gateway 是什麼

API Gateway 是微服務架構的唯一入口，所有外部請求都先經過 Gateway，再路由到後端服務。

```
前端 --> API Gateway --> 服務 A
                     \--> 服務 B
                     \--> 服務 C
```

API Gateway 負責：
- **路由**：根據 URL 轉發到對應服務
- **認證授權**：統一驗證 JWT、OAuth2
- **Rate Limiting**：限制請求頻率
- **負載平衡**：分散流量
- **快取**：減少後端負擔
- **日誌監控**：統一記錄所有請求
- **協議轉換**：HTTP → gRPC
- **請求/回應轉換**：統一格式

## Kong API Gateway

Kong 是基於 Nginx 的高效能 API Gateway，使用 Lua 擴展。

### 安裝 Kong

使用 Docker Compose：

`docker-compose.yml`：
```yaml
version: '3.7'

services:
  kong-database:
    image: postgres:11-alpine
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kong
    volumes:
      - kong-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - kong-net

  kong-migration:
    image: kong:1.0.3-alpine
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: kong
    depends_on:
      - kong-database
    networks:
      - kong-net

  kong:
    image: kong:1.0.3-alpine
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    ports:
      - "8000:8000"  # Proxy
      - "8443:8443"  # Proxy SSL
      - "8001:8001"  # Admin API
      - "8444:8444"  # Admin API SSL
    depends_on:
      - kong-database
      - kong-migration
    networks:
      - kong-net

  konga:
    image: pantsel/konga:0.14.1
    environment:
      NODE_ENV: production
    ports:
      - "1337:1337"
    depends_on:
      - kong
    networks:
      - kong-net

volumes:
  kong-data:

networks:
  kong-net:
```

啟動：
```bash
docker-compose up -d

# 驗證
curl -i http://localhost:8001/

# HTTP/1.1 200 OK
# ...
# {"version":"1.0.3",...}
```

開啟 Konga（Kong 的管理介面）：`http://localhost:1337`

### 註冊服務

假設我們有三個服務：
- User Service: `http://user-service:8080`
- Order Service: `http://order-service:8080`
- Payment Service: `http://payment-service:8080`

#### 新增 Service

```bash
# User Service
curl -i -X POST http://localhost:8001/services/ \
  --data "name=user-service" \
  --data "url=http://user-service:8080"

# Order Service
curl -i -X POST http://localhost:8001/services/ \
  --data "name=order-service" \
  --data "url=http://order-service:8080"

# Payment Service
curl -i -X POST http://localhost:8001/services/ \
  --data "name=payment-service" \
  --data "url=http://payment-service:8080"
```

#### 新增 Route

```bash
# User Service 路由
curl -i -X POST http://localhost:8001/services/user-service/routes \
  --data "paths[]=/api/users"

# Order Service 路由
curl -i -X POST http://localhost:8001/services/order-service/routes \
  --data "paths[]=/api/orders"

# Payment Service 路由
curl -i -X POST http://localhost:8001/services/payment-service/routes \
  --data "paths[]=/api/payments"
```

現在可以透過 Kong 訪問服務：
```bash
curl http://localhost:8000/api/users
curl http://localhost:8000/api/orders
curl http://localhost:8000/api/payments
```

前端只需要知道 Kong 的位址！

### 認證 - JWT Plugin

讓所有 API 都需要 JWT 驗證。

#### 建立 Consumer

```bash
curl -i -X POST http://localhost:8001/consumers/ \
  --data "username=mobile-app"
```

#### 產生 JWT Credential

```bash
curl -i -X POST http://localhost:8001/consumers/mobile-app/jwt \
  --data "algorithm=HS256" \
  --data "secret=my-secret-key"

# 回傳：
# {
#   "key": "abc123def456",
#   "secret": "my-secret-key",
#   "algorithm": "HS256"
# }
```

#### 啟用 JWT Plugin

```bash
# 全域啟用（所有服務）
curl -X POST http://localhost:8001/plugins/ \
  --data "name=jwt"

# 或針對特定服務
curl -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=jwt"
```

#### 測試

沒有 Token：
```bash
curl http://localhost:8000/api/users

# {"message":"Unauthorized"}
```

使用 JWT：
```bash
# 產生 JWT (使用 jwt.io 或程式)
TOKEN="eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."

curl http://localhost:8000/api/users \
  -H "Authorization: Bearer $TOKEN"

# 成功！
```

### Rate Limiting

限制每個使用者每分鐘只能呼叫 100 次。

```bash
curl -X POST http://localhost:8001/plugins/ \
  --data "name=rate-limiting" \
  --data "config.minute=100" \
  --data "config.policy=local"
```

測試：
```bash
# 快速呼叫
for i in {1..150}; do
  curl -s http://localhost:8000/api/users -H "Authorization: Bearer $TOKEN"
done

# 第 101 次會收到：
# {"message":"API rate limit exceeded"}
```

Header 會顯示剩餘次數：
```
X-RateLimit-Limit-Minute: 100
X-RateLimit-Remaining-Minute: 99
```

### CORS

統一設定 CORS，不用每個服務各自設定。

```bash
curl -X POST http://localhost:8001/plugins/ \
  --data "name=cors" \
  --data "config.origins=*" \
  --data "config.methods=GET,POST,PUT,DELETE" \
  --data "config.headers=Accept,Authorization,Content-Type" \
  --data "config.exposed_headers=X-Auth-Token" \
  --data "config.credentials=true" \
  --data "config.max_age=3600"
```

### 快取

快取 GET 請求，減少後端負擔。

```bash
curl -X POST http://localhost:8001/plugins/ \
  --data "name=proxy-cache" \
  --data "config.request_method=GET" \
  --data "config.response_code=200" \
  --data "config.content_type=application/json" \
  --data "config.cache_ttl=300" \
  --data "config.strategy=memory"
```

第一次請求會打到後端，後續請求直接從 Kong 回傳。

### Request/Response Transformer

修改請求和回應。

```bash
# 新增 Header
curl -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=request-transformer" \
  --data "config.add.headers=X-Service-Version:v1" \
  --data "config.add.headers=X-Request-ID:$(uuidgen)"

# 移除敏感資訊
curl -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=response-transformer" \
  --data "config.remove.json=password" \
  --data "config.remove.json=ssn"
```

### 監控

Kong 整合 Prometheus：

```bash
curl -X POST http://localhost:8001/plugins/ \
  --data "name=prometheus"
```

Prometheus 抓取指標：
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kong'
    static_configs:
      - targets: ['kong:8001']
```

Grafana 顯示：
- 每個 Route 的請求數
- 延遲分佈
- 錯誤率
- Rate limit 觸發次數

## Spring Cloud Gateway

如果是 Spring Boot 生態系，可以用 Spring Cloud Gateway。

### 建立專案

`pom.xml`：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
    <version>2.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 設定路由

`application.yml`：
```yaml
spring:
  cloud:
    gateway:
      routes:
        # User Service
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
        
        # Order Service
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Request-From, Gateway
        
        # Payment Service
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
            - Method=GET,POST
          filters:
            - StripPrefix=1

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

`lb://` 表示從 Eureka 取得服務位址，自動負載平衡。

### 自訂 Filter

驗證 JWT：

`JwtAuthenticationFilter.java`：
```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // 跳過登入 API
        if (request.getPath().value().equals("/api/auth/login")) {
            return chain.filter(exchange);
        }
        
        // 檢查 Authorization Header
        String token = extractToken(request);
        if (token == null || !tokenProvider.validateToken(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // 從 Token 取得使用者資訊
        String userId = tokenProvider.getUserIdFromToken(token);
        
        // 加入 Header 傳給後端服務
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-User-Id", userId)
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
    
    private String extractToken(ServerHttpRequest request) {
        List<String> headers = request.getHeaders().get("Authorization");
        if (headers != null && !headers.isEmpty()) {
            String bearer = headers.get(0);
            if (bearer.startsWith("Bearer ")) {
                return bearer.substring(7);
            }
        }
        return null;
    }
    
    @Override
    public int getOrder() {
        return -100;  // 優先執行
    }
}
```

### 動態路由

從資料庫載入路由設定：

```java
@Component
public class DynamicRouteService implements ApplicationEventPublisherAware {
    
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    
    private ApplicationEventPublisher publisher;
    
    public void updateRoutes(List<RouteConfig> configs) {
        configs.forEach(config -> {
            RouteDefinition definition = new RouteDefinition();
            definition.setId(config.getId());
            definition.setUri(URI.create(config.getUri()));
            
            // Predicates
            PredicateDefinition predicate = new PredicateDefinition();
            predicate.setName("Path");
            predicate.addArg("pattern", config.getPath());
            definition.getPredicates().add(predicate);
            
            // Filters
            if (config.isStripPrefix()) {
                FilterDefinition filter = new FilterDefinition();
                filter.setName("StripPrefix");
                filter.addArg("parts", "1");
                definition.getFilters().add(filter);
            }
            
            routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        });
        
        // 刷新路由
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
    
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }
}
```

提供 API 動態更新路由，不用重啟！

## API Gateway 模式

### Backend for Frontend (BFF)

不同前端使用不同的 Gateway。

```
Mobile App  --> Mobile Gateway --> Services
Web App     --> Web Gateway    --> Services
Partner API --> Partner Gateway --> Services
```

Mobile Gateway 可以：
- 合併多個 API 呼叫
- 減少資料傳輸量
- 針對行動裝置最佳化

### GraphQL Gateway

提供 GraphQL 介面，讓前端彈性查詢。

```javascript
// 前端只要一個請求
query {
  user(id: "123") {
    name
    email
    orders {
      id
      totalAmount
      items {
        productName
        quantity
      }
    }
  }
}
```

Gateway 負責：
1. 呼叫 User Service 取得使用者
2. 呼叫 Order Service 取得訂單
3. 合併結果回傳

## 實際應用經驗

### 案例：統一認證

以前每個服務都要驗證 JWT：
```java
// 30 個服務都有這段程式碼
@Component
public class JwtAuthenticationFilter { ... }
```

很麻煩，而且：
- 重複程式碼
- 改動要更新所有服務
- 效能浪費（每個服務都驗證）

導入 Kong 後：
1. Gateway 統一驗證 JWT
2. 驗證通過後，加入 `X-User-Id` Header
3. 後端服務直接讀 Header，不用驗證

程式碼簡化，效能提升！

### 案例：版本管理

有些客戶還在用舊版 API（v1），新客戶用 v2。

在 Kong 設定：
```bash
# v1 路由
curl -X POST http://localhost:8001/services/user-service-v1/routes \
  --data "paths[]=/api/v1/users"

# v2 路由
curl -X POST http://localhost:8001/services/user-service-v2/routes \
  --data "paths[]=/api/v2/users"
```

前端呼叫：
```javascript
fetch('/api/v1/users')  // 打到舊服務
fetch('/api/v2/users')  // 打到新服務
```

可以慢慢遷移使用者到 v2，v1 和 v2 共存。

### 案例：災難演練

使用 Kong 做故障注入，測試前端容錯能力。

```bash
# 讓 10% 請求回傳 500
curl -X POST http://localhost:8001/services/order-service/plugins \
  --data "name=request-termination" \
  --data "config.status_code=500" \
  --data "config.message=Service Unavailable"

# 設定路由，只影響 10% 流量
# (使用 Kong 的 canary plugin)
```

前端會看到錯誤，可以測試：
- 錯誤處理是否正確
- Retry 機制
- 降級方案

## 遇到的問題

### 問題一：Gateway 變成單點故障

所有請求都經過 Gateway，Gateway 掛了全掛。

解決方法：
1. **高可用**：部署多個 Gateway instances
2. **負載平衡**：前面放 Nginx 或 AWS ALB
3. **監控告警**：Gateway 問題立刻通知

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kong
spec:
  replicas: 3  # 至少 3 個 instances
```

### 問題二：延遲增加

多一層 Gateway，延遲增加 5-10ms。

解決方法：
1. Gateway 盡量輕量，不做複雜處理
2. 啟用快取
3. 使用高效能 Gateway（Kong、Envoy）

### 問題三：設定複雜

30 個服務，每個有多個 endpoint，設定很多。

解決方法：
1. 使用程式碼管理設定（Infrastructure as Code）
2. 自動化：從 Swagger 自動產生 Kong 設定
3. 使用 Konga 視覺化管理

## 心得

導入 API Gateway 是我們微服務架構的重要里程碑。以前前端要知道所有服務的位址，現在只需要知道 Gateway。

特別喜歡統一認證和 Rate Limiting，以前要在每個服務重複實現，現在 Gateway 統一處理，省了很多工。

Kong 的 Plugin 生態系很豐富，大部分需求都有現成的 Plugin，不用自己寫。而且效能很好，單機可以處理幾萬 RPS。

不過要注意 Gateway 不要變成新的單體，所有邏輯都塞進去。Gateway 應該專注於路由、認證、限流這些橫切關注點，業務邏輯還是在後端服務。

下週要研究分散式追蹤（Jaeger），搭配 Gateway 可以追蹤整個請求鏈路。
