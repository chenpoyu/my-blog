---
layout: post
title: "分散式追蹤 - Jaeger 深入實踐"
date: 2019-04-08 10:00:00 +0800
categories: [DevOps, Observability]
tags: [Jaeger, Distributed Tracing, OpenTracing, Microservices, Observability]
---

上週學了 API Gateway（參考 [API Gateway 與微服務架構](/posts/2019/04/01/api-gateway-microservices/)），這週深入研究分散式追蹤，解決微服務架構最頭痛的問題：如何追蹤一個請求經過了哪些服務？

我們遇到的問題：

使用者反應訂單頁面載入很慢（5 秒），但我們不知道瓶頸在哪：
```
前端 --> API Gateway --> Order Service --> ?
```

Order Service 可能呼叫：
- User Service（取使用者資訊）
- Product Service（取商品資訊）
- Inventory Service（取庫存）
- Payment Service（取支付狀態）
- Shipping Service（取物流資訊）

到底是哪個服務慢？每個服務都說「我這邊很快啊」。

看 log？30 個服務的 log，怎麼找？而且沒有 correlation ID，根本不知道哪些 log 屬於同一個請求。

需要分散式追蹤！

> 使用版本：Jaeger 1.9.0（2019 年初）、OpenTracing 0.31

## 分散式追蹤是什麼

分散式追蹤記錄一個請求的完整路徑：

```
Trace: 訂單查詢 (5000ms)
├─ Span: API Gateway (50ms)
├─ Span: Order Service (4800ms)
│  ├─ Span: Query Database (100ms)
│  ├─ Span: Call User Service (200ms)
│  ├─ Span: Call Product Service (4000ms)  ← 瓶頸！
│  │  ├─ Span: Query Database (3500ms)     ← 真正的問題
│  │  └─ Span: Call Cache (500ms)
│  └─ Span: Call Shipping Service (500ms)
└─ Span: Response (150ms)
```

一目了然！Product Service 的資料庫查詢很慢。

### OpenTracing 標準

OpenTracing 是分散式追蹤的標準 API，主要概念：

- **Trace**：完整的請求鏈路
- **Span**：單一操作（一個服務處理、一次資料庫查詢）
- **SpanContext**：跨服務傳遞的追蹤資訊（Trace ID、Span ID）
- **Tags**：標籤（http.method=GET, db.type=mysql）
- **Logs**：事件（cache hit, retry attempt）

### Jaeger 架構

Jaeger 是 Uber 開發的分散式追蹤系統，CNCF 專案。

架構：
```
應用 --> Jaeger Agent --> Jaeger Collector --> Storage (Elasticsearch)
                                             --> Jaeger Query UI
```

- **Agent**：每台機器上的代理，收集 spans
- **Collector**：聚合 spans，寫入儲存
- **Storage**：Elasticsearch、Cassandra 或記憶體
- **Query**：查詢和視覺化介面

## 安裝 Jaeger

### All-in-One（開發用）

```bash
docker run -d --name jaeger \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.9
```

開啟 UI：`http://localhost:16686`

### Production 部署

使用 Kubernetes Operator：

```bash
kubectl create namespace observability

kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/v1.9.0/deploy/crds/jaegertracing_v1_jaeger_crd.yaml

kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/v1.9.0/deploy/service_account.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/v1.9.0/deploy/role.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/v1.9.0/deploy/role_binding.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/v1.9.0/deploy/operator.yaml
```

建立 Jaeger instance：

`jaeger.yaml`：
```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        index-prefix: jaeger
  collector:
    maxReplicas: 5
    resources:
      limits:
        cpu: 1000m
        memory: 1Gi
  query:
    replicas: 2
```

```bash
kubectl apply -f jaeger.yaml
```

## Spring Boot 整合

### 加入依賴

`pom.xml`：
```xml
<dependency>
    <groupId>io.opentracing.contrib</groupId>
    <artifactId>opentracing-spring-jaeger-cloud-starter</artifactId>
    <version>2.0.0</version>
</dependency>
```

### 設定

`application.yml`：
```yaml
opentracing:
  jaeger:
    service-name: ${spring.application.name}
    udp-sender:
      host: jaeger-agent
      port: 6831
    probabilistic-sampler:
      sampling-rate: 1.0  # 100% 採樣（開發環境）
    log-spans: true

spring:
  application:
    name: order-service
```

**就這樣！** Spring Boot 自動配置完成。

### 自動追蹤

Spring Boot 自動追蹤：
- **HTTP 請求**：所有 RestController
- **RestTemplate**：呼叫其他服務
- **Feign Client**：聲明式 HTTP 客戶端
- **資料庫查詢**：JDBC、JPA
- **Redis**：如果有使用

### 手動追蹤

有時需要手動加入 Span：

```java
@Service
public class OrderService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private UserServiceClient userClient;
    
    public Order getOrder(String orderId) {
        // 建立自訂 Span
        Span span = tracer.buildSpan("process-order").start();
        try {
            span.setTag("order.id", orderId);
            
            // 從資料庫查詢
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new NotFoundException("Order not found"));
            
            // 呼叫 User Service（自動產生子 Span）
            User user = userClient.getUser(order.getUserId());
            order.setUser(user);
            
            // 記錄事件
            span.log("Order loaded successfully");
            
            // 複雜運算
            Span calculationSpan = tracer.buildSpan("calculate-total")
                .asChildOf(span)
                .start();
            try {
                BigDecimal total = calculateTotal(order);
                order.setTotal(total);
                calculationSpan.setTag("total", total.toString());
            } finally {
                calculationSpan.finish();
            }
            
            return order;
        } catch (Exception e) {
            span.setTag(Tags.ERROR, true);
            span.log(ImmutableMap.of(
                "event", "error",
                "error.object", e,
                "message", e.getMessage()
            ));
            throw e;
        } finally {
            span.finish();
        }
    }
}
```

### 傳遞 Trace Context

呼叫其他服務時，自動傳遞 Trace Context：

```java
@Bean
public RestTemplate restTemplate() {
    return new RestTemplateBuilder()
        .interceptors(new TracingRestTemplateInterceptor(tracer))
        .build();
}
```

RestTemplate 會自動加入 Header：
```
uber-trace-id: {trace-id}:{span-id}:{parent-span-id}:{flags}
```

下游服務收到後，繼續同一個 Trace。

## 查看追蹤資料

### Jaeger UI

開啟 `http://localhost:16686`。

#### 搜尋 Traces

選擇：
- **Service**：order-service
- **Operation**：GET /api/orders/{id}
- **Tags**：http.status_code=200

點 "Find Traces"，看到所有符合的 Traces。

#### 查看 Trace 詳情

點進去一個 Trace：

```
Timeline View:
────────────────────────────────────────────────────
api-gateway: GET /api/orders/123               50ms
├─ order-service: GET /orders/123            4800ms
│  ├─ DB: SELECT * FROM orders                100ms
│  ├─ user-service: GET /users/456            200ms
│  │  └─ DB: SELECT * FROM users               50ms
│  ├─ product-service: GET /products/789     4000ms  ← 紅色！
│  │  ├─ DB: SELECT * FROM products (slow!)  3500ms
│  │  └─ cache: GET product:789               500ms
│  └─ shipping-service: GET /shipments/...    500ms
└─ Response Processing                         150ms
```

立刻看出 product-service 的資料庫查詢很慢！

#### Trace 統計

看一段時間的統計：
- **P50/P90/P99 延遲**
- **錯誤率**
- **吞吐量**

例如：
```
product-service: GET /products/{id}
- Calls: 10,000
- P50: 100ms
- P95: 500ms
- P99: 4000ms  ← 有些請求特別慢
- Errors: 1.2%
```

### 依賴關係圖

System Architecture 頁面顯示服務依賴：

```
              ┌─> user-service
api-gateway ──┤
              ├─> order-service ──┬─> product-service
              │                   ├─> inventory-service
              │                   └─> shipping-service
              └─> payment-service
```

可以看出：
- 哪些服務呼叫哪些服務
- 呼叫次數
- 平均延遲
- 錯誤率

## 進階功能

### Baggage

Baggage 是在整個 Trace 傳遞的 Key-Value，所有 Spans 都能讀取。

```java
tracer.activeSpan().setBaggageItem("user-id", "123");
tracer.activeSpan().setBaggageItem("request-region", "asia");

// 在任何子 Span 都能讀取
String userId = tracer.activeSpan().getBaggageItem("user-id");
```

用途：
- 傳遞使用者 ID（用於過濾 Traces）
- 傳遞 feature flag
- 傳遞請求來源（mobile/web）

**注意**：Baggage 會增加網路傳輸量，不要放太多資料。

### Sampling 策略

不是所有請求都要追蹤（太多資料），使用 Sampling：

#### Probabilistic Sampling

隨機採樣 10%：
```yaml
opentracing:
  jaeger:
    probabilistic-sampler:
      sampling-rate: 0.1
```

#### Rate Limiting Sampling

每秒最多採樣 10 個：
```yaml
opentracing:
  jaeger:
    rate-limiting-sampler:
      max-traces-per-second: 10
```

#### Adaptive Sampling

Jaeger Collector 動態調整採樣率：
```yaml
opentracing:
  jaeger:
    remote-sampler:
      host-port: jaeger-agent:5778
```

Production 建議：
- **重要服務**：100% 採樣
- **一般服務**：10% 採樣
- **高流量服務**：1% 採樣
- **錯誤請求**：永遠採樣

### 錯誤追蹤

自動標記錯誤的 Spans：

```java
try {
    // 業務邏輯
} catch (Exception e) {
    Span span = tracer.activeSpan();
    Tags.ERROR.set(span, true);
    span.log(Map.of(
        Fields.EVENT, "error",
        Fields.ERROR_OBJECT, e,
        Fields.MESSAGE, e.getMessage(),
        Fields.STACK, ExceptionUtils.getStackTrace(e)
    ));
    throw e;
}
```

Jaeger UI 會紅色顯示錯誤的 Spans，並顯示 stack trace。

### 與 Prometheus 整合

匯出追蹤指標到 Prometheus：

```xml
<dependency>
    <groupId>io.opentracing.contrib</groupId>
    <artifactId>opentracing-prometheus-spring-autoconfigure</artifactId>
    <version>0.1.0</version>
</dependency>
```

自動產生指標：
```
# Span 數量
opentracing_spans_total{operation="GET /orders/{id}",error="false"} 1000

# Span 延遲
opentracing_span_duration_seconds_sum{operation="GET /orders/{id}"} 50.5
opentracing_span_duration_seconds_count{operation="GET /orders/{id}"} 1000
```

Grafana 顯示：
- 每個操作的延遲趨勢
- 錯誤率變化
- 吞吐量

## 最佳實踐

### 1. 有意義的 Span 名稱

❌ 不好：
```java
Span span = tracer.buildSpan("process").start();
```

好：
```java
Span span = tracer.buildSpan("calculate-order-total").start();
```

### 2. 加入關鍵 Tags

```java
span.setTag("order.id", orderId);
span.setTag("user.id", userId);
span.setTag("order.status", order.getStatus());
span.setTag("order.total", order.getTotal().toString());
span.setTag(Tags.DB_TYPE, "mysql");
span.setTag(Tags.DB_STATEMENT, sql);
```

可以用 Tags 搜尋 Traces：
```
http.status_code=500
order.status=failed
db.type=mysql
```

### 3. 記錄重要事件

```java
span.log("Cache miss, querying database");
span.log("Retry attempt 1");
span.log("Email sent successfully");
```

### 4. 追蹤非同步操作

```java
Span parentSpan = tracer.activeSpan();

CompletableFuture.supplyAsync(() -> {
    Span asyncSpan = tracer.buildSpan("async-operation")
        .asChildOf(parentSpan.context())
        .start();
    try (Scope scope = tracer.scopeManager().activate(asyncSpan)) {
        // 非同步處理
        return processAsync();
    } finally {
        asyncSpan.finish();
    }
});
```

### 5. 追蹤訊息佇列

發送訊息時注入 Trace Context：

```java
@Autowired
private Tracer tracer;

public void sendMessage(String message) {
    Span span = tracer.buildSpan("send-message").start();
    try {
        Map<String, String> headers = new HashMap<>();
        tracer.inject(span.context(), Format.Builtin.TEXT_MAP, 
            new TextMapAdapter(headers));
        
        // 將 headers 加入訊息
        rabbitTemplate.convertAndSend("exchange", "routing-key", message,
            msg -> {
                headers.forEach((k, v) -> 
                    msg.getMessageProperties().setHeader(k, v));
                return msg;
            });
    } finally {
        span.finish();
    }
}
```

接收訊息時提取 Trace Context：

```java
@RabbitListener(queues = "my-queue")
public void receiveMessage(Message message) {
    Map<String, String> headers = message.getMessageProperties()
        .getHeaders().entrySet().stream()
        .collect(Collectors.toMap(
            e -> e.getKey(),
            e -> e.getValue().toString()
        ));
    
    SpanContext parentContext = tracer.extract(
        Format.Builtin.TEXT_MAP, new TextMapAdapter(headers));
    
    Span span = tracer.buildSpan("process-message")
        .asChildOf(parentContext)
        .start();
    try (Scope scope = tracer.scopeManager().activate(span)) {
        // 處理訊息
    } finally {
        span.finish();
    }
}
```

## 實際案例

### 案例一：效能優化

使用者反應搜尋很慢。看 Jaeger Trace：

```
search-service: POST /search (5200ms)
├─ Parse Query (50ms)
├─ elasticsearch: search (4800ms)  ← 問題！
│  └─ HTTP POST /_search (4800ms)
└─ Format Results (350ms)
```

Elasticsearch 查詢很慢。展開看到：
```
POST /products/_search
{
  "query": { "match_all": {} },  ← 沒有任何過濾！
  "size": 10000                  ← 取太多資料
}
```

問題找到：查詢沒有限制條件，掃描整個索引。

修正後：
```
search-service: POST /search (200ms)
├─ Parse Query (50ms)
├─ elasticsearch: search (100ms)  ← 快多了！
└─ Format Results (50ms)
```

### 案例二：找出重複呼叫

看 Trace 發現：

```
order-service: GET /orders/123
├─ user-service: GET /users/456 (100ms)
├─ product-service: GET /products/789 (100ms)
├─ user-service: GET /users/456 (100ms)  ← 重複！
├─ inventory-service: GET /inventory/789 (100ms)
└─ user-service: GET /users/456 (100ms)  ← 又重複！
```

同一個請求中，呼叫了 3 次 user-service！

查程式碼，發現不同方法都會取使用者資訊，但沒有快取。

修正：加入 request-scoped cache。

### 案例三：找出級聯錯誤

Production 出現大量 500 錯誤。看 Jaeger，過濾 `error=true`：

```
order-service: GET /orders/123 (500ms) [ERROR]
├─ user-service: GET /users/456 (200ms) [OK]
├─ product-service: GET /products/789 (100ms) [ERROR] ← 源頭
│  └─ DB: SELECT ... (100ms) [ERROR]
│     Error: Connection timeout
└─ inventory-service: GET /inventory/789 [SKIPPED]
```

product-service 的資料庫連不上，導致整個請求失敗。

再看其他 Traces，發現所有呼叫 product-service 的請求都失敗。

找到根本原因：product-service 的資料庫掛了。

## 遇到的問題

### 問題一：儲存空間爆炸

100% 採樣，Elasticsearch 每天增加 500GB。

解決：
1. 降低採樣率（重要服務 100%，其他 10%）
2. 設定 retention policy，只保留 7 天
3. 錯誤請求永遠保留

### 問題二：效能影響

追蹤影響效能，延遲增加 5-10ms。

解決：
1. 使用非同步傳送（預設）
2. 批次傳送（減少網路呼叫）
3. 調整 Sampling rate

### 問題三：Trace 資料不完整

有時只看到部分 Spans，不完整。

原因：
1. 服務當機，Spans 沒送出
2. Agent 或 Collector 有問題
3. 網路問題

解決：
1. 檢查 Agent 和 Collector 狀態
2. 確保服務 graceful shutdown，傳送完 Spans 再關閉
3. 增加 Collector 容量

## 心得

Jaeger 真的是微服務架構的救星。以前追蹤問題要花好幾小時看 log、猜測，現在開 Jaeger 幾分鐘就找到。

特別是效能優化，以前只能靠猜，現在看 Trace 一目了然。有次優化，看 Trace 發現有個 N+1 查詢問題，修正後效能提升 10 倍。

而且對新人很友好，新來的工程師不熟悉系統架構，看 Jaeger 的依賴關係圖，立刻了解服務間的呼叫關係。

唯一要注意的是儲存成本，100% 採樣資料量很大。我們現在的策略：
- Production：10% 採樣
- 錯誤請求：100% 採樣
- 特定使用者（測試帳號）：100% 採樣

下週要研究配置中心（Spring Cloud Config），統一管理所有服務的設定。
