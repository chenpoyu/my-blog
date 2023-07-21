---
layout: post
title: "Trace Context Propagation：串聯分散式追蹤"
date: 2023-07-21 09:00:00 +0800
categories: [Observability, Tracing]
tags: [Context Propagation, W3C Trace Context, Distributed Systems, Baggage]
---

在微服務架構中，一個請求會經過多個服務。**如何讓這些服務知道它們處理的是同一個請求？**

答案是 **Context Propagation**（上下文傳遞）。

## 為什麼需要 Context Propagation？

沒有上下文傳遞時：

```
User → Service A (trace_id: abc123)
       ↓
       Service B (trace_id: def456)  ← 新的 trace_id，無法關聯
       ↓
       Service C (trace_id: ghi789)  ← 又是新的 trace_id
```

結果：Jaeger 中看到 3 個獨立的 Trace，無法串聯。

有上下文傳遞時：

```
User → Service A (trace_id: abc123)
       ↓ (透過 HTTP Header 傳遞 trace_id)
       Service B (trace_id: abc123, parent_span_id: span-A)
       ↓
       Service C (trace_id: abc123, parent_span_id: span-B)
```

結果：Jaeger 中看到 1 個完整的 Trace。

## W3C Trace Context 標準

W3C 定義了一套標準的 HTTP Header 來傳遞追蹤資訊。

### traceparent

格式：

```
traceparent: 00-{trace-id}-{parent-id}-{trace-flags}
```

範例：

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
             │  │                                │                │
             │  └─ trace-id (32 hex chars)      │                └─ flags
             │                                   └─ parent-span-id (16 hex chars)
             └─ version
```

**欄位說明**：

| 欄位 | 長度 | 說明 |
|------|------|------|
| version | 2 chars | 版本號（目前是 `00`） |
| trace-id | 32 chars | Trace 的唯一識別碼 |
| parent-id | 16 chars | 父 Span 的唯一識別碼 |
| trace-flags | 2 chars | 旗標（`01` = sampled, `00` = not sampled） |

### tracestate

用於傳遞供應商特定的資訊。

```
tracestate: congo=t61rcWkgMzE,rojo=00f067aa0ba902b7
```

## OpenTelemetry 自動傳遞

使用 OpenTelemetry SDK 時，**不需要手動處理 Header**，SDK 會自動：

1. **發送請求時**：自動加入 `traceparent` Header
2. **接收請求時**：自動讀取 `traceparent` Header

### Java 範例

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

**實際發送的 HTTP 請求**：

```http
GET http://inventory-service/api/inventory/123
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

### .NET 範例

```csharp
app.MapGet("/api/orders/{id}", async (long id, HttpClient httpClient) =>
{
    // OpenTelemetry 會自動加入 traceparent header
    var inventory = await httpClient.GetStringAsync($"http://inventory-service/api/inventory/{id}");
    
    return await orderService.GetByIdAsync(id);
});
```

## 手動傳遞（如果沒有自動 Instrumentation）

如果你使用的 HTTP Client 沒有 OpenTelemetry 支援，需要手動加入 Header。

### Java 手動傳遞

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.context.propagation.TextMapSetter;
import io.opentelemetry.context.propagation.TextMapPropagator;

@Service
public class OrderService {
    private final TextMapPropagator propagator;
    
    public String callInventoryService(Long productId) {
        HttpHeaders headers = new HttpHeaders();
        
        // 手動注入 trace context
        propagator.inject(Context.current(), headers, new TextMapSetter<HttpHeaders>() {
            @Override
            public void set(HttpHeaders carrier, String key, String value) {
                carrier.set(key, value);
            }
        });
        
        // 發送請求
        HttpEntity<Void> entity = new HttpEntity<>(headers);
        return restTemplate.exchange(
            "http://inventory-service/api/inventory/" + productId,
            HttpMethod.GET,
            entity,
            String.class
        ).getBody();
    }
}
```

### .NET 手動傳遞

```csharp
using System.Diagnostics;

public class OrderService
{
    public async Task<string> CallInventoryServiceAsync(long productId)
    {
        using var activity = new ActivitySource("OrderService").StartActivity("call_inventory");
        
        using var httpClient = new HttpClient();
        
        // 手動注入 trace context
        if (Activity.Current != null)
        {
            httpClient.DefaultRequestHeaders.Add("traceparent", Activity.Current.Id);
        }
        
        return await httpClient.GetStringAsync($"http://inventory-service/api/inventory/{productId}");
    }
}
```

## 跨訊息佇列的傳遞

### Kafka

**Producer**：

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.propagation.TextMapSetter;

@Service
public class OrderProducer {
    private final KafkaTemplate<String, Order> kafkaTemplate;
    
    public void sendOrder(Order order) {
        ProducerRecord<String, Order> record = new ProducerRecord<>("orders", order);
        
        // 注入 trace context 到 Kafka Headers
        GlobalOpenTelemetry.getPropagators().getTextMapPropagator()
            .inject(Context.current(), record.headers(), new TextMapSetter<Headers>() {
                @Override
                public void set(Headers carrier, String key, String value) {
                    carrier.add(key, value.getBytes());
                }
            });
        
        kafkaTemplate.send(record);
    }
}
```

**Consumer**：

```java
@Service
public class OrderConsumer {
    @KafkaListener(topics = "orders")
    public void consume(ConsumerRecord<String, Order> record) {
        // 提取 trace context 從 Kafka Headers
        Context context = GlobalOpenTelemetry.getPropagators().getTextMapPropagator()
            .extract(Context.current(), record.headers(), new TextMapGetter<Headers>() {
                @Override
                public Iterable<String> keys(Headers carrier) {
                    return carrier.stream().map(Header::key).collect(Collectors.toList());
                }
                
                @Override
                public String get(Headers carrier, String key) {
                    Header header = carrier.lastHeader(key);
                    return header == null ? null : new String(header.value());
                }
            });
        
        try (Scope scope = context.makeCurrent()) {
            // 處理訊息
            orderService.process(record.value());
        }
    }
}
```

## Baggage：傳遞額外資訊

除了 `trace_id` 和 `span_id`，你可能想傳遞其他資訊（如 `user_id`, `tenant_id`）。

**Baggage** 就是用來傳遞這些資訊的。

### 設定 Baggage

```java
import io.opentelemetry.api.baggage.Baggage;

Baggage baggage = Baggage.current()
    .toBuilder()
    .put("user.id", "john@example.com")
    .put("tenant.id", "tenant-123")
    .build();

try (Scope scope = baggage.makeCurrent()) {
    // 在這個 scope 內，所有的 HTTP 請求都會自動帶上這些資訊
    restTemplate.getForObject("http://inventory-service/api/inventory/123", String.class);
}
```

### 讀取 Baggage

```java
String userId = Baggage.current().getEntryValue("user.id");
String tenantId = Baggage.current().getEntryValue("tenant.id");

log.info("Processing request for user: {}, tenant: {}", userId, tenantId);
```

### Baggage 的傳輸

Baggage 會透過 `baggage` HTTP Header 傳遞：

```http
baggage: user.id=john@example.com,tenant.id=tenant-123
```

### 注意事項

1. **不要放太多資料**：Baggage 會在每個請求中傳遞，太大會影響效能
2. **不要放敏感資訊**：Baggage 是明文傳輸
3. **建議大小**：< 1KB

## 常見問題

### Q1：如果中間某個服務沒有傳遞 Context 怎麼辦？

Trace 會斷掉，變成兩個獨立的 Trace。

**解決方案**：確保所有服務都使用 OpenTelemetry 或手動傳遞 Header。

### Q2：如果服務 A 呼叫服務 B 兩次，會建立兩個 Span 嗎？

是的，每次呼叫都會建立一個新的 Span。

```
Service A
├── Call Service B (1st)
└── Call Service B (2nd)
```

### Q3：非 HTTP 的通訊（如 gRPC）如何傳遞？

gRPC 也使用 Metadata 傳遞：

```java
Metadata metadata = new Metadata();
propagator.inject(Context.current(), metadata, metadataSetter);

stub.withInterceptors(MetadataUtils.newAttachHeadersInterceptor(metadata))
    .getOrder(request);
```

### Q4：如果兩個服務使用不同的 Tracing 系統怎麼辦？

W3C Trace Context 是標準，所以：
- Service A 用 OpenTelemetry + Jaeger
- Service B 用 Zipkin

只要都支援 W3C Trace Context，就能串聯。

## 實戰：Debug Context 傳遞問題

### 症狀

Service A 的 Trace 無法串到 Service B。

### Debug 步驟

#### 1. 檢查 HTTP Header

在 Service A 加上日誌：

```java
@GetMapping("/api/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    // 檢查是否有 traceparent
    String traceparent = Span.current().getSpanContext().getTraceId();
    log.info("Current trace_id: {}", traceparent);
    
    // ...
}
```

#### 2. 檢查 HTTP 請求

用 Wireshark 或 tcpdump 抓包，確認 HTTP 請求有沒有 `traceparent` Header。

#### 3. 檢查 Service B 的日誌

在 Service B 加上日誌：

```java
@GetMapping("/api/inventory/{id}")
public Inventory getInventory(@PathVariable Long id, @RequestHeader(value = "traceparent", required = false) String traceparent) {
    log.info("Received traceparent: {}", traceparent);
    // ...
}
```

#### 4. 確認 OpenTelemetry 版本

確保所有服務使用相容的 OpenTelemetry 版本。

---

**Context Propagation 是分散式追蹤的基礎，沒有它，Tracing 就無法運作。**

好消息是：如果你用 OpenTelemetry，99% 的情況下它會自動處理。
