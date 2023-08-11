---
layout: post
title: "錯誤追蹤與 Root Cause Analysis"
date: 2023-08-11 09:00:00 +0800
categories: [Observability, Debugging]
tags: [Error Tracking, RCA, Debugging, Sentry, OpenTelemetry]
---

在分散式系統中，錯誤很難追蹤。一個請求可能經過 5 個服務，但只有第 3 個服務出錯。

今天我們來談談如何**系統化地追蹤錯誤**，並找到根本原因（Root Cause）。

## 錯誤追蹤的挑戰

### 問題 1：錯誤散落在各處

```
order-service 的日誌：
  2023-08-11 10:15:23 ERROR Failed to create order

payment-service 的日誌：
  2023-08-11 10:15:22 ERROR Stripe API returned 500

database 的日誌：
  2023-08-11 10:15:21 ERROR Connection timeout
```

這三個錯誤是同一個問題嗎？還是三個不同的問題？

### 問題 2：缺少上下文

```
2023-08-11 10:15:23 ERROR NullPointerException
  at com.example.OrderService.createOrder(OrderService.java:45)
```

這個錯誤是怎麼觸發的？使用者做了什麼操作？

### 問題 3：重複的錯誤

同一個錯誤在 10 秒內發生了 1000 次，你要看 1000 遍嗎？

## 錯誤追蹤系統：Sentry

Sentry 是最流行的錯誤追蹤系統，它可以：
- 自動聚合相同的錯誤
- 提供完整的堆疊追蹤和上下文
- 整合到 Slack、Jira 等工具

### 安裝 Sentry

**自建 Sentry**（使用 Docker）：

```bash
git clone https://github.com/getsentry/self-hosted.git
cd self-hosted
./install.sh
docker-compose up -d
```

或使用 Sentry 的雲端服務：https://sentry.io

### Java 整合 Sentry

**1. 加入依賴**：

```xml
<dependency>
    <groupId>io.sentry</groupId>
    <artifactId>sentry-spring-boot-starter</artifactId>
    <version>6.28.0</version>
</dependency>
```

**2. 配置**：

```yaml
sentry:
  dsn: https://examplePublicKey@o0.ingest.sentry.io/0
  traces-sample-rate: 1.0  # 100% 的請求會被追蹤
  environment: production
  release: order-service@1.2.3
```

**3. 自動捕捉錯誤**：

```java
@RestController
public class OrderController {
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);  // 如果拋出異常，Sentry 會自動捕捉
    }
}
```

**4. 手動捕捉錯誤**：

```java
import io.sentry.Sentry;

try {
    order = orderService.findById(id);
} catch (Exception e) {
    Sentry.captureException(e);
    throw e;
}
```

**5. 加入上下文**：

```java
Sentry.configureScope(scope -> {
    scope.setUser(new User().setId("123").setEmail("user@example.com"));
    scope.setTag("order_id", "456");
    scope.setExtra("payment_method", "stripe");
});

try {
    orderService.createOrder(request);
} catch (Exception e) {
    Sentry.captureException(e);
}
```

### .NET 整合 Sentry

**1. 安裝套件**：

```bash
dotnet add package Sentry.AspNetCore
```

**2. 配置**：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.UseSentry(options =>
{
    options.Dsn = "https://examplePublicKey@o0.ingest.sentry.io/0";
    options.TracesSampleRate = 1.0;
    options.Environment = "production";
    options.Release = "order-service@1.2.3";
});

var app = builder.Build();
```

**3. 手動捕捉錯誤**：

```csharp
using Sentry;

try
{
    var order = await orderService.GetByIdAsync(id);
}
catch (Exception ex)
{
    SentrySdk.CaptureException(ex);
    throw;
}
```

**4. 加入上下文**：

```csharp
SentrySdk.ConfigureScope(scope =>
{
    scope.User = new User { Id = "123", Email = "user@example.com" };
    scope.SetTag("order_id", "456");
    scope.SetExtra("payment_method", "stripe");
});
```

## Sentry 的功能

### 1. 錯誤聚合（Grouping）

Sentry 會自動把相同的錯誤聚合在一起：

```
NullPointerException: Cannot invoke "String.length()" because "name" is null
  at OrderService.createOrder(OrderService.java:45)
  
  發生次數: 1,234
  影響的使用者: 567
  首次出現: 2023-08-11 10:00:00
  最後出現: 2023-08-11 15:30:00
```

### 2. 堆疊追蹤（Stack Trace）

```
java.lang.NullPointerException: Cannot invoke "String.length()" because "name" is null
  at com.example.OrderService.createOrder(OrderService.java:45)
  at com.example.OrderController.createOrder(OrderController.java:23)
  at jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
  ...
```

Sentry 會顯示完整的堆疊追蹤，並且可以點選檔案名稱，直接跳到 GitHub 上的程式碼。

### 3. Breadcrumbs（麵包屑）

Breadcrumbs 記錄了錯誤發生前的操作：

```
10:15:18 [INFO] User logged in (user_id=123)
10:15:19 [INFO] User clicked "Create Order" button
10:15:20 [HTTP] POST /api/orders (status=200)
10:15:21 [HTTP] POST /api/payment (status=200)
10:15:22 [HTTP] POST /api/stripe (status=500) ← 錯誤發生
10:15:23 [ERROR] NullPointerException
```

這樣你就知道錯誤是怎麼被觸發的。

### 4. 使用者資訊

```
User ID: 123
Email: user@example.com
IP: 192.168.1.1
Browser: Chrome 115.0.0.0
OS: macOS 13.4
```

### 5. 自訂標籤和上下文

```
Tags:
  order_id: 456
  payment_method: stripe
  environment: production

Extra:
  cart_items: [{"id": 1, "name": "iPhone", "price": 999}]
  total_amount: 999.00
  discount_code: SUMMER2023
```

## 整合 Sentry 和 Jaeger

Sentry 可以和 Jaeger 整合，這樣你就能從錯誤跳到 Trace。

> **關於 trace_id 的詳細說明**：請參考《Week 29: Trace Context Propagation》。

### 配置

```java
import io.sentry.Sentry;
import io.opentelemetry.api.trace.Span;

Sentry.configureScope(scope -> {
    String traceId = Span.current().getSpanContext().getTraceId();
    scope.setTag("trace_id", traceId);
});
```

在 Sentry 的錯誤詳情中，你會看到：

```
Tags:
  trace_id: 0af7651916cd43dd8448eb211c80319c
```

點選 `trace_id`，可以配置一個連結，自動跳到 Jaeger：

```
http://jaeger:16686/trace/0af7651916cd43dd8448eb211c80319c
```

## OpenTelemetry 的錯誤追蹤

OpenTelemetry 也支援錯誤追蹤，但比較基礎。

### 記錄錯誤到 Span

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;

Span span = tracer.spanBuilder("create-order").startSpan();
try {
    orderService.createOrder(request);
} catch (Exception e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR, e.getMessage());
    throw e;
} finally {
    span.end();
}
```

在 Jaeger 中，你會看到：

```
Span: create-order (ERROR)
  Tags:
    error: true
    exception.type: java.lang.NullPointerException
    exception.message: Cannot invoke "String.length()" because "name" is null
    exception.stacktrace: ...
```

### OpenTelemetry vs Sentry

| 功能                  | OpenTelemetry          | Sentry               |
|-----------------------|------------------------|----------------------|
| 自動捕捉錯誤          | ❌                     | ✅                   |
| 錯誤聚合              | ❌                     | ✅                   |
| Breadcrumbs           | ❌                     | ✅                   |
| 使用者資訊            | ❌                     | ✅                   |
| Release 追蹤          | ❌                     | ✅                   |
| 整合 Jira、Slack      | ❌                     | ✅                   |
| 開源                  | ✅                     | ✅（有雲端版本）     |

**建議**：用 OpenTelemetry 做 Tracing，用 Sentry 做錯誤追蹤。

## Root Cause Analysis（RCA）流程

當錯誤發生時，如何找到根本原因？

### 步驟 1：從 Sentry 開始

打開 Sentry，看到一個錯誤：

```
NullPointerException: Cannot invoke "String.length()" because "name" is null
  at OrderService.createOrder(OrderService.java:45)
  
  發生次數: 1,234
  影響的使用者: 567
```

### 步驟 2：查看 Breadcrumbs

```
10:15:19 [HTTP] POST /api/orders (status=200)
10:15:20 [HTTP] POST /api/payment (status=200)
10:15:21 [HTTP] POST /api/stripe (status=500) ← 關鍵
10:15:22 [ERROR] NullPointerException
```

看起來是 Stripe API 返回了 500，導致後續的錯誤。

### 步驟 3：查看 Trace

從 Sentry 複製 `trace_id`，打開 Jaeger：

```
order-service: POST /api/orders (150ms) [ERROR]
├── payment-service: POST /api/payment (120ms) [ERROR]
│   └── stripe-api: POST /v1/charges (100ms) [ERROR]
└── notification-service: POST /api/notify (20ms) [OK]
```

確認是 `stripe-api` 出錯。

### 步驟 4：查看 Logs

從 Jaeger 複製 `trace_id`，打開 Kibana：

```json
{
  "timestamp": "2023-08-11T10:15:21.123Z",
  "level": "ERROR",
  "message": "Stripe API returned 500",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "error": {
    "type": "StripeAPIException",
    "message": "Internal Server Error",
    "response_body": "{\"error\": \"rate_limit_exceeded\"}"
  }
}
```

**根本原因**：Stripe API 達到速率限制。

### 步驟 5：查看 Metrics

打開 Grafana，查看 Stripe API 的呼叫次數：

```promql
rate(stripe_api_requests_total[5m])
```

看到在 10:15 的時候，呼叫次數突然飆升到 500 req/s。

**原因**：有個排程任務在 10:15 執行，大量呼叫 Stripe API。

### 步驟 6：解決問題

加入速率限制：

```java
@Service
public class PaymentService {
    @RateLimiter(name = "stripe", fallbackMethod = "fallback")
    public Payment charge(ChargeRequest request) {
        return stripeClient.charge(request);
    }
    
    public Payment fallback(ChargeRequest request, RateLimitExceededException e) {
        log.warn("Rate limit exceeded, retrying in 60 seconds");
        throw new RetryableException("Rate limit exceeded", 60);
    }
}
```

## 建立 Runbook

當錯誤發生時，團隊應該有一套標準的處理流程（Runbook）。

### Runbook 範例

**錯誤**：`StripeRateLimitException`

**影響**：使用者無法完成付款

**處理步驟**：

1. 確認錯誤頻率（Sentry Dashboard）
2. 查看 Stripe API 呼叫次數（Grafana）
3. 如果呼叫次數 > 400 req/s，暫停排程任務
4. 聯絡 Stripe 支援，申請提高速率限制
5. 監控錯誤率是否下降

**預防措施**：

- 加入速率限制（Rate Limiter）
- 加入重試機制（Retry）
- 加入熔斷器（Circuit Breaker）

## 錯誤分級

不是所有錯誤都需要立即處理。

### 分級標準

| 等級     | 定義                         | 回應時間 | 範例                         |
|----------|------------------------------|----------|------------------------------|
| P0       | 系統完全無法使用             | 立即     | 資料庫無法連線               |
| P1       | 主要功能無法使用             | 1 小時   | 付款功能失敗                 |
| P2       | 次要功能無法使用             | 1 天     | 推薦功能失敗                 |
| P3       | 不影響功能的錯誤             | 1 週     | 日誌記錄失敗                 |

### Sentry 中設定優先級

```java
Sentry.configureScope(scope -> {
    scope.setLevel(SentryLevel.FATAL);  // P0
    scope.setTag("priority", "P0");
});
```

在 Sentry 的告警規則中，可以根據 `priority` 標籤設定不同的通知方式：

- P0：打電話給 on-call 工程師
- P1：發送 Slack 訊息
- P2：發送 Email
- P3：不通知

## 實戰建議

### 1. 設定 Release 追蹤

每次部署時，更新 `release` 版本：

```yaml
sentry:
  release: order-service@1.2.3
```

這樣你就能看到每個版本的錯誤率：

```
v1.2.2: 0.5% 錯誤率
v1.2.3: 2.3% 錯誤率 ← 這個版本有問題！
```

### 2. 設定錯誤率告警

當錯誤率超過閾值時，自動回滾：

```yaml
# Sentry Alert Rule
conditions:
  - type: event_frequency
    value: 100  # 100 次錯誤
    interval: 5m  # 5 分鐘內

actions:
  - type: slack
    workspace: engineering
    channel: #alerts
  - type: webhook
    url: https://your-ci-cd.com/rollback
```

### 3. 設定 Source Maps（前端）

如果是前端錯誤，上傳 Source Maps：

```bash
sentry-cli releases files order-service@1.2.3 upload-sourcemaps ./dist
```

這樣 Sentry 可以把壓縮後的程式碼還原成原始程式碼。

---

**錯誤追蹤不只是記錄錯誤，而是要能快速找到根本原因，並建立標準的處理流程。**

當你能在 5 分鐘內定位問題，你就贏了。
