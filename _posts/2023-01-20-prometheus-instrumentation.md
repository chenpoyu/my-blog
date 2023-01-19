---
layout: post
title: "讓你的應用程式開口說話：Instrumentation 實戰"
date: 2023-01-20 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Prometheus, Instrumentation, Metrics, Java, .NET Core]
---

上週我們安裝了 Prometheus，並且用 Node Exporter 監控了系統資源。但說實話，**系統資源只能告訴你『機器』的狀態，無法告訴你『應用程式』發生了什麼事。**

今天我們要讓應用程式「開口說話」，這個過程叫做 **Instrumentation（埋點）**。

> 本文範例使用：**Java 11 + Spring Boot 2.7** 和 **.NET 6.0 (ASP.NET Core)**

## 什麼是 Instrumentation？

簡單來說，就是在你的程式碼中加入「監控點」，讓它主動暴露指標。

舉個例子，假設你有一個 API：

```java
@GetMapping("/api/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderService.findById(id);
}
```

你想知道：
- 這個 API 被呼叫了幾次？
- 平均回應時間多久？
- 有多少次失敗？

沒有 Instrumentation，你只能靠「猜」。有了 Instrumentation，你就能「看見」。

## 方法一：使用 Prometheus Client Library（手動埋點）

### Java + Spring Boot 範例

首先加入依賴（`pom.xml`）：

```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient</artifactId>
    <version>0.16.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_hotspot</artifactId>
    <version>0.16.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_spring_boot</artifactId>
    <version>0.16.0</version>
</dependency>
```

建立一個 Configuration：

```java
@Configuration
public class PrometheusConfig {
    
    @Bean
    public CollectorRegistry collectorRegistry() {
        CollectorRegistry registry = new CollectorRegistry();
        DefaultExports.initialize();  // 註冊 JVM 指標
        return registry;
    }
}
```

加入 `/metrics` 端點：

```java
@RestController
public class MetricsController {
    
    @Autowired
    private CollectorRegistry collectorRegistry;
    
    @GetMapping(value = "/metrics", produces = "text/plain")
    public String metrics() throws IOException {
        StringWriter writer = new StringWriter();
        TextFormat.write004(writer, collectorRegistry.metricFamilySamples());
        return writer.toString();
    }
}
```

現在啟動應用程式，訪問 `http://localhost:8080/metrics`，你會看到一堆指標：

```
# HELP jvm_memory_bytes_used Used bytes of a given JVM memory area.
# TYPE jvm_memory_bytes_used gauge
jvm_memory_bytes_used{area="heap",} 1.234567E8
jvm_memory_bytes_used{area="nonheap",} 5.678901E7
```

### .NET Core + ASP.NET Core 範例

安裝 NuGet 套件：

```bash
dotnet add package prometheus-net.AspNetCore
```

建立應用程式 (`Program.cs`)：

```csharp
using Prometheus;

var builder = WebApplication.CreateBuilder(args);

// 定義指標
var requestCounter = Metrics.CreateCounter(
    "http_requests_total", 
    "Total HTTP requests",
    new CounterConfiguration
    {
        LabelNames = new[] { "method", "endpoint", "status" }
    });

var requestLatency = Metrics.CreateHistogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    new HistogramConfiguration
    {
        LabelNames = new[] { "method", "endpoint" }
    });

var app = builder.Build();

// 啟用 Prometheus metrics 端點
app.UseMetricServer();  // 預設在 /metrics
app.UseHttpMetrics();   // 自動收集 HTTP 指標

app.MapGet("/api/orders/{id}", (int id) =>
{
    using (requestLatency.WithLabels("GET", "/api/orders").NewTimer())
    {
        // 業務邏輯
        var result = new { id, status = "shipped" };
        
        // 記錄指標
        requestCounter.WithLabels("GET", "/api/orders", "200").Inc();
        
        return Results.Ok(result);
    }
});

app.Run();
```

訪問 `http://localhost:5000/metrics`，你會看到：

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/orders",status="200"} 42

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",endpoint="/api/orders",le="0.005"} 10
http_request_duration_seconds_bucket{method="GET",endpoint="/api/orders",le="0.01"} 20
```

**.NET Core 的優勢**在於 `prometheus-net` 套件整合度極高，只需要兩行程式碼（`UseMetricServer()` 和 `UseHttpMetrics()`）就能自動收集所有 HTTP 請求的指標。

訪問 `http://localhost:5000/metrics`，你會看到：

```
```

### 更進階的 .NET Core 監控

如果你使用的是 ASP.NET Core MVC 或 Web API，可以建立一個 Middleware：

```csharp
public class PrometheusMiddleware
{
    private readonly RequestDelegate _next;
    private static readonly Histogram _requestDuration = Metrics.CreateHistogram(
        "http_request_duration_seconds",
        "HTTP request duration in seconds",
        new HistogramConfiguration
        {
            LabelNames = new[] { "method", "endpoint", "status_code" },
            Buckets = Histogram.ExponentialBuckets(0.001, 2, 10)
        });

    public PrometheusMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var path = context.Request.Path.Value;
        var method = context.Request.Method;

        using (_requestDuration.WithLabels(method, path, "").NewTimer())
        {
            await _next(context);
        }
        
        _requestDuration.WithLabels(
            method, 
            path, 
            context.Response.StatusCode.ToString()
        ).Observe(0);
    }
}
```

在 `Program.cs` 中註冊：

```csharp
app.UseMiddleware<PrometheusMiddleware>();
```

## 方法二：使用 Spring Boot Actuator（自動化）

手動埋點很累？Spring Boot 提供了更優雅的方式。

加入依賴：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

設定 `application.yml`：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

就這樣！現在訪問 `http://localhost:8080/actuator/prometheus`，你會自動得到：

- JVM 記憶體、執行緒、GC
- HTTP 請求數量、延遲
- 資料庫連線池狀態
- ...等等

**這就是為什麼我喜歡 Spring Boot：它把最佳實踐內建了。**

## 設定 Prometheus 抓取指標

修改 `prometheus.yml`：

```yaml
scrape_configs:
  - job_name: 'my-java-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
  
  - job_name: 'my-dotnet-app'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['host.docker.internal:5000']
```

重啟 Prometheus，稍等片刻，你就能查詢到應用程式的指標了。

## 重點心法：不要過度埋點

我見過一個案例：某個團隊把每個 function 的執行時間都記錄下來，結果：

1. **效能下降**：埋點本身有開銷
2. **儲存爆炸**：時序資料庫塞滿了無用數據
3. **查詢變慢**：指標太多,PromQL 跑不動

我的建議是：**只監控對使用者有影響的行為。**

### 哪些該監控？

✅ **HTTP API 的請求數、延遲、錯誤率**
✅ **資料庫查詢延遲、連線池使用率**
✅ **外部服務呼叫的成功率、超時次數**
✅ **快取命中率**
✅ **訊息佇列的積壓量**

### 哪些不該監控？

❌ 內部 private function 的執行時間（除非是瓶頸）
❌ 每個使用者的個別行為（應該用 APM 工具，不是 Prometheus）
❌ 頻繁變化的 Labels（例如 user_id、session_id）

## 實戰案例：監控外部 API 呼叫

假設你的系統會呼叫第三方支付 API，你最關心的是：

1. 成功率
2. 延遲
3. 超時次數

```java
@Service
public class PaymentService {
    
    private static final Counter paymentRequests = Counter.build()
        .name("payment_requests_total")
        .help("Total payment requests")
        .labelNames("provider", "status")
        .register();
    
    private static final Histogram paymentLatency = Histogram.build()
        .name("payment_request_duration_seconds")
        .help("Payment request latency")
        .labelNames("provider")
        .register();
    
    public PaymentResult pay(PaymentRequest request) {
        Histogram.Timer timer = paymentLatency.labels("stripe").startTimer();
        
        try {
            PaymentResult result = stripeClient.charge(request);
            paymentRequests.labels("stripe", "success").inc();
            return result;
        } catch (PaymentException e) {
            paymentRequests.labels("stripe", "failure").inc();
            throw e;
        } finally {
            timer.observeDuration();
        }
    }
}
```

這樣一來，你可以在 Prometheus 查詢：

```promql
# 支付成功率
sum(rate(payment_requests_total{status="success"}[5m])) / 
sum(rate(payment_requests_total[5m]))

# P95 延遲
histogram_quantile(0.95, 
  rate(payment_request_duration_seconds_bucket[5m])
)
```

---

**Instrumentation 不是為了監控而監控，而是為了在出問題時,能快速定位根因。**

當你能說「支付 API 的 P95 延遲從 200ms 漲到 5 秒」，而不是「好像變慢了」，你就已經贏了一半。
