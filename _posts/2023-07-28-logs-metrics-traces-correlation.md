---
layout: post
title: "Logs-Metrics-Traces 三位一體：關聯分析"
date: 2023-07-28 09:00:00 +0800
categories: [Observability, Integration]
tags: [Logs, Metrics, Traces, Correlation, Unified Observability]
---

到目前為止，我們分別學習了 Metrics、Logs、Traces。但它們不應該是孤立的。

今天我們來談談如何**關聯這三種資料**，實現真正的可觀測性。

## 三者的關係

```
Metrics: 告訴你「發生了什麼」（What）
         ↓
Traces:  告訴你「在哪裡發生」（Where）
         ↓
Logs:    告訴你「為什麼發生」（Why）
```

### 實際案例

**Metrics 告訴你**：
```
http_requests_error_rate{service="payment-service"} = 5%  ← 異常！
```

**Traces 告訴你**：
```
payment-service → stripe-api (30s timeout)  ← 問題在這裡
```

**Logs 告訴你**：
```
ERROR: Stripe API returned 503 Service Unavailable
Message: "Rate limit exceeded"  ← 根因
```

## 關聯的關鍵：trace_id

所有資料都應該包含 `trace_id`，這樣就能串聯起來。

### Logs 中加入 trace_id

> **詳細說明**：關於如何在 Logs 中加入 trace_id 的完整實作，請參考《Week 29: Trace Context Propagation》文章。

**簡單範例（Java）**：

```java
import io.opentelemetry.api.trace.Span;
import org.slf4j.MDC;

@RestController
public class OrderController {
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // OpenTelemetry 會自動設定 MDC
        String traceId = Span.current().getSpanContext().getTraceId();
        MDC.put("trace_id", traceId);
        
        log.info("Fetching order: {}", id);
        return orderService.findById(id);
    }
}
```
{
    var activity = Activity.Current;
    
    using (LogContext.PushProperty("trace_id", activity?.TraceId.ToString()))
    using (LogContext.PushProperty("span_id", activity?.SpanId.ToString()))
    {
        logger.LogInformation("Fetching order: {OrderId}", id);
        return orderService.GetById(id);
    }
});
```

### Metrics 中加入 trace_id（Exemplars）

Prometheus 2.26+ 支援 **Exemplars**，可以在 Metrics 中加入 `trace_id`。

**Java 範例**：

```java
import io.prometheus.client.exemplars.tracer.otel.OpenTelemetryTracer;

Counter errorCounter = Counter.build()
    .name("http_requests_errors_total")
    .help("Total number of errors")
    .labelNames("service", "endpoint")
    .register();

// 增加計數時，自動加入 trace_id
errorCounter.labels("order-service", "/api/orders").inc();
```

**Prometheus 查詢**：

```promql
http_requests_errors_total{service="order-service"}
```

**結果**：

```
http_requests_errors_total{service="order-service"} 123
  exemplar: trace_id="0af7651916cd43dd8448eb211c80319c" timestamp=1627483200
```

點選 Exemplar，可以直接跳到 Jaeger 看 Trace！

## 實戰：從 Metrics 到 Traces 到 Logs

### 場景

你收到告警：

```
[CRITICAL] payment-service 錯誤率 > 5%
```

### 步驟 1：查看 Grafana Dashboard

打開 Grafana，看到：

```
http_requests_error_rate{service="payment-service"} = 5.2%
```

時間：10:15-10:30

### 步驟 2：點選 Exemplar 跳到 Jaeger

在 Grafana 的圖表上，你會看到一些點（Exemplars），點選其中一個。

自動跳轉到 Jaeger，看到一個錯誤的 Trace：

```
payment-service: POST /api/payment (30.5s) [ERROR]
├── validate_payment (45ms)
├── stripe-api: POST /v1/charges (30.2s) [ERROR]
└── db: UPDATE payments (23ms)
```

### 步驟 3：從 Trace 跳到 Logs

在 Jaeger 的 Span 詳細資訊中，你看到：

```
trace_id: 0af7651916cd43dd8448eb211c80319c
span_id: b7ad6b7169203331
error: true
error.message: Stripe API timeout
```

複製 `trace_id`，打開 Kibana：

```
trace_id:"0af7651916cd43dd8448eb211c80319c"
```

### 步驟 4：找到根因

在 Kibana 中看到詳細的錯誤日誌：

```json
{
  "timestamp": "2023-07-28T10:15:23.123Z",
  "level": "ERROR",
  "message": "Stripe API request failed",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "span_id": "b7ad6b7169203331",
  "error": {
    "type": "StripeRateLimitException",
    "message": "Rate limit exceeded. Please retry after 60 seconds.",
    "retry_after": 60
  }
}
```

**根因**：Stripe API 達到速率限制。

### 步驟 5：解決問題

加入 Retry 機制：

```java
@Service
public class PaymentService {
    @Retryable(
        value = StripeRateLimitException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 60000)  // 等 60 秒後重試
    )
    public Payment charge(ChargeRequest request) {
        return stripeClient.charge(request);
    }
}
```

## Grafana 整合 Jaeger 和 Loki

### 設定 Grafana Data Sources

#### 1. Prometheus

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    jsonData:
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: jaeger  # 連結到 Jaeger
```

#### 2. Jaeger

```yaml
  - name: Jaeger
    type: jaeger
    uid: jaeger
    url: http://jaeger-query:16686
```

#### 3. Loki

```yaml
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: jaeger
          matcherRegex: "trace_id=(\\w+)"
          name: trace_id
          url: "$${__value.raw}"
```

### 在 Grafana 中跳轉

#### Prometheus → Jaeger

在 Grafana 的 Prometheus 圖表中，點選 Exemplar（小點），自動跳到 Jaeger。

#### Loki → Jaeger

在 Loki 的日誌中，`trace_id` 會變成超連結，點選後跳到 Jaeger。

#### Jaeger → Loki

在 Jaeger 的 Span 中，加入一個連結：

```java
span.setAttribute("log.query", "trace_id:" + traceId);
```

在 Grafana 中配置：

```yaml
traceQuery:
  datasourceUid: loki
  tags:
    - key: log.query
      value: 'trace_id="${__trace.traceId}"'
```

## OpenTelemetry Collector 統一收集

### 架構

```
應用程式 → OpenTelemetry Collector → Jaeger (Traces)
                                    → Prometheus (Metrics)
                                    → Loki (Logs)
```

### Collector 配置

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 10s
  
  # 從 Logs 提取 Metrics
  metricstransform:
    transforms:
      - include: ^logs_.*
        match_type: regexp
        action: update
        operations:
          - action: add_label
            new_label: extracted_from
            new_value: logs

exporters:
  jaeger:
    endpoint: jaeger-collector:14250
  
  prometheus:
    endpoint: "0.0.0.0:8889"
  
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
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

## 統一的 Dashboard

### Grafana Dashboard 範例

**Panel 1：錯誤率（Metrics）**

```promql
rate(http_requests_errors_total{service="payment-service"}[5m])
```

**Panel 2：P95 延遲（Traces）**

```promql
histogram_quantile(0.95, rate(jaeger_traces_duration_seconds_bucket[5m]))
```

**Panel 3：錯誤日誌（Logs）**

```logql
{service="payment-service"} |= "ERROR"
```

**Panel 4：Trace 列表（Traces）**

用 Jaeger Data Source 顯示最近的 Traces。

## 實戰技巧

### 1. 在告警中加入跳轉連結

```yaml
groups:
  - name: payment-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_errors_total[5m]) > 0.05
        annotations:
          summary: "High error rate in payment-service"
          description: "Error rate: {{ $value }}"
          jaeger_link: "http://jaeger:16686/search?service=payment-service&start={{ $value | timestamp }}000000&end={{ $value | timestamp }}000000"
          kibana_link: "http://kibana:5601/app/discover#/?_g=(time:(from:now-15m,to:now))&_a=(query:(match:(service:(query:payment-service,type:phrase))))"
```

### 2. 在 Logs 中加入 Trace 連結

```json
{
  "message": "Payment failed",
  "trace_url": "http://jaeger:16686/trace/0af7651916cd43dd8448eb211c80319c"
}
```

### 3. 在 Traces 中加入 Logs 連結

```java
span.setAttribute("log.url", 
    "http://kibana:5601/app/discover#/?_a=(query:(match:(trace_id:(query:" + traceId + ",type:phrase))))"
);
```

---

**Logs、Metrics、Traces 不是三個孤立的系統，而是一個整體。**

當你能在它們之間自由跳轉，排查問題的速度會提升 10 倍。
