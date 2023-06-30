---
layout: post
title: "分散式追蹤：理解 Trace 與 Span"
date: 2023-06-30 09:00:00 +0800
categories: [Observability, Tracing]
tags: [Distributed Tracing, Trace, Span, Context Propagation, Jaeger]
---

在單體應用中，一個請求的完整路徑很容易追蹤。

但在微服務架構中，一個請求可能經過 10 個不同的服務：

```
User → API Gateway → Order Service → Inventory Service → Payment Service
                   → Notification Service → Email Service
```

**如何追蹤這個請求的完整路徑？如何知道哪個環節最慢？**

這就是**分散式追蹤（Distributed Tracing）**要解決的問題。

## 核心概念

### Trace（追蹤）

**一個完整的請求路徑**，由多個 Span 組成。

每個 Trace 有唯一的 `trace_id`。

### Span（跨度）

**請求路徑中的一個步驟**。

每個 Span 有：
- `span_id`：唯一識別碼
- `parent_span_id`：父 Span 的 ID
- `operation_name`：操作名稱（如 `GET /api/orders`）
- `start_time`：開始時間
- `duration`：執行時間
- `tags`：自定義屬性（如 `http.status_code=200`）
- `logs`：事件記錄（如 `"cache miss"`）

### 範例：建立訂單的 Trace

```
Trace ID: abc123

├── Span: API Gateway (200ms)
│   ├── Span: Order Service (150ms)
│   │   ├── Span: Validate User (20ms)
│   │   ├── Span: Check Inventory (50ms)
│   │   └── Span: Create Order in DB (80ms)
│   └── Span: Payment Service (30ms)
│       └── Span: Call Stripe API (25ms)
└── Span: Send Notification (15ms)
```

總執行時間：200ms
最慢的環節：Create Order in DB（80ms）

## Context Propagation（上下文傳遞）

分散式追蹤的關鍵：**如何在服務之間傳遞 `trace_id` 和 `span_id`？**

### W3C Trace Context 標準

OpenTelemetry 使用 **W3C Trace Context**，透過 HTTP Header 傳遞：

```
traceparent: 00-abc123def456-012345678901-01
             │  │            │            │
             │  └─ trace_id  │            └─ flags
             │               └─ parent_span_id
             └─ version
```

### 範例：服務 A 呼叫服務 B

**服務 A（Order Service）**：

```java
@RestController
public class OrderController {
    private final RestTemplate restTemplate;
    
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // OpenTelemetry 會自動加入 traceparent header
        String inventory = restTemplate.getForObject(
            "http://inventory-service/api/inventory/" + id, 
            String.class
        );
        
        return orderService.findById(id);
    }
}
```

**HTTP 請求**：

```
GET http://inventory-service/api/inventory/123
traceparent: 00-abc123def456-789012345678-01
```

**服務 B（Inventory Service）**：

```java
@RestController
public class InventoryController {
    @GetMapping("/api/inventory/{id}")
    public Inventory getInventory(@PathVariable Long id) {
        // OpenTelemetry 會自動讀取 traceparent header
        // 並建立一個新的 Span（parent_span_id = 789012345678）
        return inventoryService.findById(id);
    }
}
```

**不需要手動傳遞，OpenTelemetry 會自動處理！**

## Span 的類型

### 1. Server Span（服務端 Span）

接收 HTTP 請求時建立。

```java
// 自動建立（OpenTelemetry auto-instrumentation）
@GetMapping("/api/orders")
public List<Order> getOrders() {
    return orderService.findAll();
}
```

**Span 屬性**：
```
span.kind: SERVER
http.method: GET
http.url: /api/orders
http.status_code: 200
```

### 2. Client Span（客戶端 Span）

發送 HTTP 請求時建立。

```java
// 自動建立（OpenTelemetry auto-instrumentation）
restTemplate.getForObject("http://payment-service/api/payment", String.class);
```

**Span 屬性**：
```
span.kind: CLIENT
http.method: GET
http.url: http://payment-service/api/payment
http.status_code: 200
```

### 3. Internal Span（內部 Span）

內部邏輯（如資料庫查詢）。

```java
Span span = tracer.spanBuilder("query_database").startSpan();
try (Scope scope = span.makeCurrent()) {
    span.setAttribute("db.statement", "SELECT * FROM orders WHERE id = ?");
    return jdbcTemplate.query("SELECT * FROM orders WHERE id = ?", id);
} finally {
    span.end();
}
```

**Span 屬性**：
```
span.kind: INTERNAL
db.system: postgresql
db.statement: SELECT * FROM orders WHERE id = ?
```

### 4. Producer / Consumer Span（訊息佇列）

用於 Kafka、RabbitMQ 等訊息系統。

**Producer**：

```java
kafkaTemplate.send("orders", order);
```

**Span 屬性**：
```
span.kind: PRODUCER
messaging.system: kafka
messaging.destination: orders
```

**Consumer**：

```java
@KafkaListener(topics = "orders")
public void consume(Order order) {
    // ...
}
```

**Span 屬性**：
```
span.kind: CONSUMER
messaging.system: kafka
messaging.destination: orders
```

## 手動建立 Span

### Java 範例

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

@Service
public class OrderService {
    private final Tracer tracer;
    
    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("create_order")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 加入屬性
            span.setAttribute("order.id", request.getOrderId());
            span.setAttribute("order.amount", request.getAmount());
            span.setAttribute("user.id", request.getUserId());
            
            // 記錄事件
            span.addEvent("Validating order");
            validateOrder(request);
            
            span.addEvent("Creating order in database");
            Order order = orderRepository.save(request);
            
            span.addEvent("Order created successfully");
            span.setStatus(StatusCode.OK);
            return order;
            
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### .NET 範例

```csharp
using System.Diagnostics;

public class OrderService
{
    private readonly ActivitySource _activitySource = new("OrderService");
    
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        using var activity = _activitySource.StartActivity("create_order", ActivityKind.Internal);
        
        activity?.SetTag("order.id", request.OrderId);
        activity?.SetTag("order.amount", request.Amount);
        activity?.SetTag("user.id", request.UserId);
        
        try
        {
            activity?.AddEvent(new ActivityEvent("Validating order"));
            await ValidateOrderAsync(request);
            
            activity?.AddEvent(new ActivityEvent("Creating order in database"));
            var order = await _orderRepository.SaveAsync(request);
            
            activity?.AddEvent(new ActivityEvent("Order created successfully"));
            activity?.SetStatus(ActivityStatusCode.Ok);
            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }
}
```

## Span 屬性的最佳實踐

### 1. 使用標準屬性

OpenTelemetry 定義了一套標準屬性：

**HTTP**：
```
http.method: GET
http.url: /api/orders
http.status_code: 200
http.user_agent: Mozilla/5.0...
```

**Database**：
```
db.system: postgresql
db.name: orders_db
db.statement: SELECT * FROM orders
db.operation: SELECT
```

**RPC**：
```
rpc.system: grpc
rpc.service: OrderService
rpc.method: CreateOrder
```

完整列表：https://opentelemetry.io/docs/reference/specification/trace/semantic_conventions/

### 2. 加入業務屬性

```java
span.setAttribute("order.id", orderId);
span.setAttribute("user.id", userId);
span.setAttribute("order.amount", amount);
span.setAttribute("payment.method", "credit_card");
```

### 3. 不要記錄敏感資訊

❌ **錯誤**：
```java
span.setAttribute("user.password", password);
span.setAttribute("credit_card.number", cardNumber);
```

✅ **正確**：
```java
span.setAttribute("user.id", userId);  // 只記錄 ID
span.setAttribute("payment.method", "credit_card");  // 不記錄卡號
```

## Sampling（取樣）

如果你每秒有 10,000 個請求，全部追蹤會產生大量資料。

**Sampling** 的作用：只追蹤一部分請求。

### 1. Always On（全部追蹤）

```java
SdkTracerProvider.builder()
    .setSampler(Sampler.alwaysOn())
    .build();
```

**適用場景**：開發環境、低流量系統

### 2. Always Off（全部不追蹤）

```java
SdkTracerProvider.builder()
    .setSampler(Sampler.alwaysOff())
    .build();
```

**適用場景**：測試用

### 3. Trace ID Ratio（按比例取樣）

```java
SdkTracerProvider.builder()
    .setSampler(Sampler.traceIdRatioBased(0.1))  // 追蹤 10%
    .build();
```

**適用場景**：高流量系統

### 4. Parent Based（基於父 Span）

如果父 Span 被追蹤，子 Span 也會被追蹤。

```java
SdkTracerProvider.builder()
    .setSampler(Sampler.parentBased(Sampler.traceIdRatioBased(0.1)))
    .build();
```

**適用場景**：大部分情況（預設值）

## 在 Jaeger 中查看 Trace

### 啟動 Jaeger

```bash
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
  jaegertracing/all-in-one:latest
```

### 存取 Jaeger UI

```
http://localhost:16686
```

### 搜尋 Trace

1. **Service**：選擇 `order-service`
2. **Operation**：選擇 `GET /api/orders`
3. **Lookback**：Last 1 hour
4. **Find Traces**

### Trace 詳細資訊

點選任一個 Trace，你會看到：

```
Timeline View:
├── API Gateway (200ms) ████████████████████████
│   ├── Order Service (150ms) ████████████████
│   │   ├── Validate User (20ms) ██
│   │   ├── Check Inventory (50ms) ██████
│   │   └── DB Query (80ms) ████████████
│   └── Payment Service (30ms) ████
```

點選任一個 Span，可以看到：
- **Tags**：所有屬性
- **Logs**：事件記錄
- **Process**：服務資訊（如主機名稱、版本）

---

**分散式追蹤讓你看到「請求的完整生命週期」，而不是「單一服務的片段」。**

這是理解微服務效能瓶頸的關鍵。
