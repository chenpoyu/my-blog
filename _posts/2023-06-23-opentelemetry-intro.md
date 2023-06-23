---
layout: post
title: "OpenTelemetry 入門：可觀測性的新標準"
date: 2023-06-23 09:00:00 +0800
categories: [Observability, OpenTelemetry]
tags: [OpenTelemetry, OTLP, Tracing, Metrics, Logs, Standards]
---

如果你用過 Prometheus、Jaeger、ELK Stack，你會發現一個問題：**它們都有自己的格式與協定。**

- Prometheus 用 Pull 模式收集 Metrics
- Jaeger 用 Thrift 或 gRPC 收集 Traces
- ELK 用 JSON 收集 Logs

當你想切換工具時（如從 Jaeger 換到 Zipkin），你需要重寫所有的 instrumentation code。

**OpenTelemetry（OTEL）** 的目標就是解決這個問題：**一套標準,支援所有可觀測性工具。**

## OpenTelemetry 是什麼？

OpenTelemetry 是一個開源專案，由 CNCF（Cloud Native Computing Foundation）託管，合併了：

- **OpenTracing**（Tracing 標準）
- **OpenCensus**（Metrics 標準）

它提供：

### 1. 統一的 API 與 SDK

你只需要寫一次 instrumentation code，就能：
- 把 Traces 送到 Jaeger / Zipkin / Tempo
- 把 Metrics 送到 Prometheus / DataDog / New Relic
- 把 Logs 送到 ELK / Loki / Splunk

### 2. 自動 Instrumentation

支援主流框架的自動埋點：
- Java: Spring Boot, JDBC, HTTP Client
- .NET: ASP.NET Core, HttpClient, Entity Framework
- Node.js: Express, HTTP, MySQL
- Python: Flask, Django, Requests

### 3. 統一的協定（OTLP）

**OpenTelemetry Protocol (OTLP)** 是一個標準的傳輸協定，支援：
- HTTP/2 + Protobuf
- gRPC
- JSON（人類可讀）

## OpenTelemetry 的三大支柱

### 1. Traces（追蹤）

記錄請求在分散式系統中的完整路徑。

```
User → API Gateway → Order Service → Payment Service → Database
```

每個步驟都是一個 **Span**，所有 Span 組成一個 **Trace**。

### 2. Metrics（指標）

記錄系統的量化數據。

```
http_requests_total{service="order-service", status="200"} 12345
```

### 3. Logs（日誌）

記錄系統的事件。

```json
{
  "timestamp": "2023-06-23T10:15:23Z",
  "level": "ERROR",
  "message": "Payment timeout",
  "trace_id": "abc123",
  "span_id": "def456"
}
```

**關鍵**：Logs 可以透過 `trace_id` 和 `span_id` **關聯到 Traces**！

## OpenTelemetry 架構

```
應用程式 → OpenTelemetry SDK → OpenTelemetry Collector → 後端
                                                         ↓
                                         (Jaeger / Prometheus / ELK)
```

### OpenTelemetry SDK

在應用程式中埋點，收集 Telemetry 資料。

### OpenTelemetry Collector

集中處理 Telemetry 資料：
- 接收（Receiver）：從應用程式接收資料
- 處理（Processor）：過濾、轉換、批次處理
- 匯出（Exporter）：送到不同的後端

### 後端

儲存與查詢 Telemetry 資料：
- Traces: Jaeger, Zipkin, Tempo
- Metrics: Prometheus, Victoria Metrics
- Logs: ELK, Loki

## Java + Spring Boot 範例

### 1. 加入依賴

```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.27.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>1.27.0-alpha</version>
</dependency>
```

### 2. 設定

```yaml
# application.yml
otel:
  service:
    name: order-service
  traces:
    exporter: otlp
  metrics:
    exporter: otlp
  exporter:
    otlp:
      endpoint: http://localhost:4317
```

### 3. 自動 Instrumentation

**不需要改任何程式碼！**Spring Boot 會自動埋點：

```java
@RestController
public class OrderController {
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // OpenTelemetry 會自動建立 Span
        return orderService.findById(id);
    }
}
```

### 4. 手動 Instrumentation

如果你想加入自定義的 Span：

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

@Service
public class OrderService {
    private final Tracer tracer;
    
    public OrderService(Tracer tracer) {
        this.tracer = tracer;
    }
    
    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("create_order").startSpan();
        try (Scope scope = span.makeCurrent()) {
            // 加入自定義屬性
            span.setAttribute("order.id", request.getOrderId());
            span.setAttribute("user.id", request.getUserId());
            
            // 業務邏輯
            Order order = processOrder(request);
            
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

## .NET Core 範例

### 1. 安裝套件

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

### 2. 設定

```csharp
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProviderBuilder =>
        tracerProviderBuilder
            .AddSource("OrderService")
            .SetResourceBuilder(
                ResourceBuilder.CreateDefault()
                    .AddService("order-service"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri("http://localhost:4317");
            }));

var app = builder.Build();
```

### 3. 使用

```csharp
using System.Diagnostics;

app.MapPost("/api/orders", async (CreateOrderRequest request) =>
{
    var activitySource = new ActivitySource("OrderService");
    
    using var activity = activitySource.StartActivity("CreateOrder");
    activity?.SetTag("order.id", request.OrderId);
    activity?.SetTag("user.id", request.UserId);
    
    try
    {
        var order = await orderService.CreateAsync(request);
        return Results.Ok(order);
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
});
```

## OpenTelemetry Collector

### 安裝

```bash
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.80.0/otelcol_0.80.0_linux_amd64.tar.gz
tar -xvf otelcol_0.80.0_linux_amd64.tar.gz
```

### 設定

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  
  attributes:
    actions:
      - key: environment
        value: production
        action: insert

exporters:
  # Traces 送到 Jaeger
  jaeger:
    endpoint: localhost:14250
    tls:
      insecure: true
  
  # Metrics 送到 Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
  
  # Logs 送到 Loki
  loki:
    endpoint: http://localhost:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

### 啟動

```bash
./otelcol --config=otel-collector-config.yaml
```

## 為什麼要用 OpenTelemetry？

### 1. 供應商中立（Vendor Neutral）

你不會被綁定在某個商業產品上。

```
今天用 Jaeger → 明天換成 Tempo → 不需要改程式碼
```

### 2. 自動 Instrumentation

不需要手動埋點，框架會自動收集：
- HTTP 請求/回應
- 資料庫查詢
- 快取操作
- 訊息佇列

### 3. 統一的 Context Propagation

在分散式系統中，`trace_id` 會自動傳遞：

```
User → API Gateway (trace_id: abc123)
       ↓
       Order Service (trace_id: abc123)
       ↓
       Payment Service (trace_id: abc123)
```

### 4. 關聯 Logs / Metrics / Traces

```
看到錯誤日誌 → 點選 trace_id → 看到完整的請求路徑
看到 Metrics 異常 → 點選時間點 → 看到相關的 Traces
```

## OpenTelemetry vs 傳統方案

| 特性 | OpenTelemetry | Jaeger SDK | Prometheus Client |
|------|---------------|-----------|------------------|
| **標準化** | ✅ 統一標準 | ❌ Jaeger 專用 | ❌ Prometheus 專用 |
| **供應商中立** | ✅ 可切換後端 | ❌ 只能用 Jaeger | ❌ 只能用 Prometheus |
| **自動埋點** | ✅ 支援 | ❌ 需手動 | ❌ 需手動 |
| **Logs/Traces 關聯** | ✅ 原生支援 | ❌ 需自行實作 | ❌ 不支援 |

## 實戰：從 Jaeger SDK 遷移到 OpenTelemetry

### Before（Jaeger SDK）

```java
import io.jaegertracing.Configuration;

Configuration config = new Configuration("order-service");
Tracer tracer = config.getTracer();

Span span = tracer.buildSpan("create_order").start();
// ...
span.finish();
```

**問題**：
- 綁定 Jaeger
- 如果要換成 Zipkin，需要重寫

### After（OpenTelemetry）

```java
import io.opentelemetry.api.trace.Tracer;

Tracer tracer = openTelemetry.getTracer("order-service");

Span span = tracer.spanBuilder("create_order").startSpan();
// ...
span.end();
```

**好處**：
- 標準 API
- 要換後端，只需改 `otel-collector-config.yaml`

---

**OpenTelemetry 是可觀測性的未來。**

它不會取代 Prometheus、Jaeger、ELK，而是讓你更容易整合它們。
