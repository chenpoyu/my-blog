---
layout: post
title: "結構化日誌：讓日誌變成可查詢的數據"
date: 2023-04-14 09:00:00 +0800
categories: [Observability, Logging]
tags: [Structured Logging, JSON Logging, Log Parsing, Best Practices]
---

上週我們搭建了 ELK Stack，並且成功收集了日誌。但你很快會發現一個問題：**原始日誌很難查詢。**

看看這兩種日誌格式：

**非結構化日誌（純文字）**：
```
2023-04-14 10:15:23 ERROR Failed to process order 12345 for user john@example.com due to payment timeout
```

**結構化日誌（JSON）**：
```json
{
  "timestamp": "2023-04-14T10:15:23Z",
  "level": "ERROR",
  "message": "Failed to process order",
  "orderId": 12345,
  "userId": "john@example.com",
  "errorType": "PaymentTimeout",
  "duration": 30000
}
```

哪個更容易查詢？答案顯而易見。

## 非結構化日誌的痛點

假設你要找「所有 user_id 為 12345 的錯誤」。

### 用純文字日誌

```
message:*user 12345* AND level:ERROR
```

問題：
- 可能誤匹配到「order 12345」
- 可能漏掉「user=12345」或「userId:12345」
- 查詢效率低（全文搜尋）

### 用結構化日誌

```
userId:12345 AND level:ERROR
```

精準、快速、不會誤判。

## 什麼是結構化日誌？

結構化日誌的核心概念：**每個欄位都有明確的名稱和類型。**

```json
{
  "timestamp": "2023-04-14T10:15:23Z",    // 時間（ISO 8601）
  "level": "ERROR",                        // 日誌級別（字串）
  "message": "Failed to process order",   // 訊息（字串）
  "orderId": 12345,                       // 訂單 ID（數字）
  "userId": "john@example.com",           // 使用者 ID（字串）
  "duration": 30000,                      // 執行時間（毫秒）
  "tags": ["payment", "timeout"]          // 標籤（陣列）
}
```

這樣 Elasticsearch 就能建立正確的索引：
- `timestamp` → Date 類型
- `orderId` → Integer 類型
- `duration` → Long 類型
- `tags` → Keyword 陣列

## Java 實作：Logback + JSON

我們在上週已經用了 `logstash-logback-encoder`，它預設就輸出 JSON。

### 進階設定：加入自定義欄位

```xml
<configuration>
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>localhost:5000</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- 靜態欄位 -->
            <customFields>{"app":"order-service","environment":"production","version":"1.2.3"}</customFields>
            
            <!-- 包含 MDC -->
            <includeMdcKeyName>requestId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>orderId</includeMdcKeyName>
            
            <!-- 包含 Exception stack trace -->
            <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                <maxDepthPerThrowable>30</maxDepthPerThrowable>
            </throwableConverter>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="LOGSTASH" />
    </root>
</configuration>
```

### 在程式碼中使用 MDC（Mapped Diagnostic Context）

```java
import org.slf4j.MDC;

@RestController
public class OrderController {
    private static final Logger log = LoggerFactory.getLogger(OrderController.class);
    
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 設定 MDC（會自動加入日誌）
        MDC.put("userId", request.getUserId());
        MDC.put("orderId", UUID.randomUUID().toString());
        MDC.put("requestId", request.getRequestId());
        
        try {
            log.info("Creating order");
            
            Order order = orderService.create(request);
            
            log.info("Order created successfully");
            return order;
            
        } catch (PaymentTimeoutException e) {
            log.error("Payment timeout", e);
            throw e;
        } finally {
            // 清除 MDC（避免記憶體洩漏）
            MDC.clear();
        }
    }
}
```

輸出的 JSON：

```json
{
  "@timestamp": "2023-04-14T10:15:23.123Z",
  "level": "ERROR",
  "message": "Payment timeout",
  "app": "order-service",
  "environment": "production",
  "version": "1.2.3",
  "userId": "john@example.com",
  "orderId": "a1b2c3d4",
  "requestId": "req-12345",
  "exception": {
    "class": "PaymentTimeoutException",
    "message": "Timeout after 30s",
    "stacktrace": "..."
  }
}
```

## .NET Core 實作：Serilog + JSON

### 設定 Serilog

```csharp
using Serilog;
using Serilog.Formatting.Json;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()  // 啟用 LogContext
    .Enrich.WithProperty("app", "order-service")
    .Enrich.WithProperty("environment", "production")
    .Enrich.WithProperty("version", "1.2.3")
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.TCPSink("localhost", 5000, new JsonFormatter())
    .CreateLogger();
```

### 在程式碼中使用 LogContext

```csharp
using Serilog.Context;

app.MapPost("/api/orders", (CreateOrderRequest request, ILogger<Program> logger) =>
{
    // 使用 LogContext 加入欄位
    using (LogContext.PushProperty("userId", request.UserId))
    using (LogContext.PushProperty("orderId", Guid.NewGuid()))
    using (LogContext.PushProperty("requestId", request.RequestId))
    {
        logger.LogInformation("Creating order");
        
        try
        {
            var order = orderService.Create(request);
            logger.LogInformation("Order created successfully");
            return Results.Ok(order);
        }
        catch (PaymentTimeoutException ex)
        {
            logger.LogError(ex, "Payment timeout");
            throw;
        }
    }
});
```

## Logstash 解析非結構化日誌

如果你的系統已經在運行，無法立刻改成 JSON，怎麼辦？

可以用 Logstash 的 **Grok** 過濾器解析。

### 範例：解析 Nginx access log

原始日誌：
```
192.168.1.100 - - [14/Apr/2023:10:15:23 +0000] "GET /api/orders HTTP/1.1" 200 1234 "https://example.com" "Mozilla/5.0"
```

Logstash 設定：

```ruby
filter {
  grok {
    match => {
      "message" => '%{IPORHOST:clientip} - - \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:httpversion}" %{NUMBER:status:int} %{NUMBER:bytes:int} "%{DATA:referrer}" "%{DATA:useragent}"'
    }
  }
  
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }
  
  geoip {
    source => "clientip"
  }
}
```

解析後的 JSON：

```json
{
  "clientip": "192.168.1.100",
  "timestamp": "14/Apr/2023:10:15:23 +0000",
  "method": "GET",
  "request": "/api/orders",
  "httpversion": "1.1",
  "status": 200,
  "bytes": 1234,
  "referrer": "https://example.com",
  "useragent": "Mozilla/5.0",
  "geoip": {
    "country_name": "Taiwan",
    "city_name": "Taipei"
  }
}
```

## 日誌欄位設計的最佳實踐

### 1. 必須有的欄位

每筆日誌都應該包含：

```json
{
  "timestamp": "2023-04-14T10:15:23.123Z",  // ISO 8601 格式
  "level": "ERROR",                          // DEBUG, INFO, WARN, ERROR, FATAL
  "message": "Human readable message",       // 人類可讀的訊息
  "app": "order-service",                    // 服務名稱
  "environment": "production",               // 環境（dev, staging, prod）
  "version": "1.2.3",                        // 應用程式版本
  "hostname": "server-01"                    // 主機名稱
}
```

### 2. 業務相關欄位

```json
{
  "userId": "john@example.com",
  "orderId": 12345,
  "action": "create_order",
  "result": "success"
}
```

### 3. 效能相關欄位

```json
{
  "duration": 1234,           // 執行時間（毫秒）
  "dbQueryTime": 456,         // 資料庫查詢時間
  "externalApiTime": 789      // 外部 API 呼叫時間
}
```

### 4. 錯誤相關欄位

```json
{
  "errorType": "PaymentTimeoutException",
  "errorMessage": "Timeout after 30s",
  "stackTrace": "...",
  "errorCode": "E1001"
}
```

### 5. 追蹤相關欄位

```json
{
  "requestId": "req-12345",    // 請求 ID（整個請求鏈路唯一）
  "spanId": "span-67890",      // Span ID（分散式追蹤）
  "traceId": "trace-abcde"     // Trace ID
}
```

## 欄位命名規範

不同的團隊可能用不同的命名：

❌ **不一致的命名**：
```
user_id, userId, UserID, uid, user
```

✅ **統一的命名**：
```
userId  （統一用 camelCase）
```

我的建議：
- **使用 camelCase**：`userId`, `orderId`, `errorType`
- **避免縮寫**：用 `userId` 而不是 `uid`
- **保持一致**：全公司統一命名規範

## 敏感資訊處理

千萬不要在日誌中記錄：

❌ **密碼、信用卡號、身分證字號**

```java
// 錯誤示範
log.info("User login: username={}, password={}", username, password);  // 危險！
```

✅ **遮罩處理**

```java
// 正確示範
log.info("User login: username={}", username);

// 如果一定要記錄，至少遮罩
String maskedCard = cardNumber.replaceAll("\\d(?=\\d{4})", "*");
log.info("Payment: card={}", maskedCard);  // ****-****-****-1234
```

## Kibana 中的結構化查詢

有了結構化日誌，查詢變得超級簡單。

### 範例 1：找出特定使用者的所有操作

```
userId:"john@example.com"
```

### 範例 2：找出慢查詢（超過 1 秒）

```
duration:>1000 AND action:*query*
```

### 範例 3：找出付款失敗的訂單

```
action:"create_order" AND result:"failure" AND errorType:"PaymentException"
```

### 範例 4：聚合分析

在 Kibana 的 **Visualize** 中：

**圖表類型**：Bar chart  
**X-axis**：`errorType.keyword`（Top 10）  
**Y-axis**：Count

這會顯示「最常見的 10 種錯誤類型」。

## JSON vs 純文字：效能考量

有人會問：「JSON 日誌不是更佔空間嗎？」

### 儲存空間比較

**純文字日誌**：
```
2023-04-14 10:15:23 ERROR Failed to process order 12345 for user john@example.com
```
**大小**：83 bytes

**JSON 日誌**：
```json
{"timestamp":"2023-04-14T10:15:23Z","level":"ERROR","message":"Failed to process order","orderId":12345,"userId":"john@example.com"}
```
**大小**：144 bytes

**差異**：多了 73%

但考慮到：
- Elasticsearch 會壓縮儲存（實際差異約 20-30%）
- 查詢效率提升 10 倍以上
- 不需要 Grok 解析（節省 CPU）

**結論：值得。**

## 實戰案例：從日誌發現 N+1 查詢問題

我們在日誌中加入了資料庫查詢次數：

```java
int queryCount = // 計算查詢次數
log.info("Request completed", 
    kv("requestId", requestId),
    kv("endpoint", "/api/orders"),
    kv("duration", duration),
    kv("dbQueryCount", queryCount)  // 關鍵欄位
);
```

在 Kibana 查詢：

```
endpoint:"/api/orders" AND dbQueryCount:>10
```

發現某些請求執行了 50+ 次資料庫查詢！

進一步調查後發現是經典的 **N+1 問題**：

```java
// 問題程式碼
List<Order> orders = orderRepository.findAll();  // 1 次查詢
for (Order order : orders) {
    User user = userRepository.findById(order.getUserId());  // N 次查詢
}
```

修正後：

```java
// 使用 JOIN 或 Batch Fetch
List<Order> orders = orderRepository.findAllWithUsers();  // 1 次查詢
```

**效能提升**：從 5 秒降到 0.5 秒。

---

**結構化日誌不只是格式問題，而是可觀測性的基礎。**

當每個欄位都有明確的意義，你就能用數據驅動決策，而不是靠猜測。
