---
layout: post
title: "Trace Sampling 策略與成本優化"
date: 2023-08-18 09:00:00 +0800
categories: [Observability, Cost Optimization]
tags: [Tracing, Sampling, Cost, Performance, OpenTelemetry]
---

當你的系統每秒處理 10,000 個請求，如果 100% 採樣，會產生大量的 Trace 資料。

這不僅會增加成本，還會影響效能。

今天我們來談談如何**智慧地採樣**，在成本和可觀測性之間取得平衡。

## 採樣的必要性

### 不採樣的成本

假設一個系統：
- 每秒 10,000 個請求
- 每個 Trace 平均 5 個 Span
- 每個 Span 平均 2 KB

**每天的資料量**：

```
10,000 req/s × 5 span × 2 KB × 86,400 s/day = 8.64 TB/day
```

**每月的儲存成本**（以 AWS S3 為例）：

```
8.64 TB × 30 days × $0.023/GB = $5,961/month
```

這還不包括 Jaeger、Elasticsearch 的運算成本。

### 採樣的效益

如果採樣率是 1%：

```
8.64 TB/day × 1% = 86.4 GB/day
86.4 GB × 30 days × $0.023/GB = $59.6/month
```

成本降低了 **100 倍**。

## 採樣策略

### 1. Head-Based Sampling（頭部採樣）

在請求**開始時**決定是否採樣。

#### 優點
- 實作簡單
- 效能開銷小

#### 缺點
- 可能會遺漏錯誤的請求

#### 實作

**OpenTelemetry Java**：

```java
import io.opentelemetry.sdk.trace.samplers.Sampler;

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setSampler(Sampler.traceIdRatioBased(0.01))  // 1% 採樣
    .build();
```

**OpenTelemetry .NET**：

```csharp
using OpenTelemetry.Trace;

var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .SetSampler(new TraceIdRatioBasedSampler(0.01))  // 1% 採樣
    .Build();
```

### 2. Tail-Based Sampling（尾部採樣）

在請求**結束後**決定是否採樣。

#### 優點
- 可以保留所有錯誤的請求
- 可以保留延遲異常的請求

#### 缺點
- 需要暫存所有 Trace
- 效能開銷較大

#### 實作

Tail-Based Sampling 通常在 **OpenTelemetry Collector** 中實作。

```yaml
processors:
  tail_sampling:
    decision_wait: 10s  # 等待 10 秒後決定
    num_traces: 100000  # 暫存 100,000 條 Trace
    expected_new_traces_per_sec: 1000
    policies:
      # 保留所有錯誤的 Trace
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      
      # 保留延遲 > 1s 的 Trace
      - name: slow
        type: latency
        latency:
          threshold_ms: 1000
      
      # 其他的 Trace 只保留 1%
      - name: default
        type: probabilistic
        probabilistic:
          sampling_percentage: 1

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, batch]
      exporters: [jaeger]
```

### 3. Adaptive Sampling（自適應採樣）

根據**當前的流量和錯誤率**動態調整採樣率。

#### Jaeger 的 Adaptive Sampling

Jaeger 會根據每個服務的流量，動態調整採樣率：

```json
{
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.01
  },
  "per_operation_strategies": [
    {
      "operation": "POST /api/orders",
      "type": "probabilistic",
      "param": 0.1  // 重要的 API，採樣率較高
    },
    {
      "operation": "GET /health",
      "type": "probabilistic",
      "param": 0.001  // Health check，採樣率較低
    }
  ]
}
```

Jaeger 會自動計算每個服務的流量，並調整採樣率，確保：
- 每個服務每秒至少有 1 個 Trace
- 總體採樣率不超過設定的上限

### 4. Priority Sampling（優先級採樣）

根據請求的**重要性**決定採樣率。

#### 範例

```java
import io.opentelemetry.api.trace.Span;

@RestController
public class OrderController {
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        Span span = Span.current();
        
        // 如果是 VIP 使用者，設定高優先級
        if (user.isVip()) {
            span.setAttribute("sampling.priority", 1);  // 一定採樣
        } else {
            span.setAttribute("sampling.priority", 0);  // 使用預設採樣率
        }
        
        return orderService.findById(id);
    }
}
```

在 OpenTelemetry Collector 中：

```yaml
processors:
  tail_sampling:
    policies:
      - name: vip
        type: string_attribute
        string_attribute:
          key: sampling.priority
          values: ["1"]
          enabled_regex_matching: false
          invert_match: false
```

## 實戰：多層採樣策略

在實際應用中，我們會結合多種採樣策略。

### 架構

```
應用程式 (Head Sampling 10%) → OpenTelemetry Collector (Tail Sampling) → Jaeger
```

### 步驟 1：應用程式中使用 Head Sampling

```java
SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setSampler(Sampler.traceIdRatioBased(0.1))  // 10% 採樣
    .build();
```

這樣可以減少應用程式的效能開銷。

### 步驟 2：Collector 中使用 Tail Sampling

```yaml
processors:
  tail_sampling:
    policies:
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      
      - name: slow
        type: latency
        latency:
          threshold_ms: 1000
      
      - name: default
        type: probabilistic
        probabilistic:
          sampling_percentage: 10  # 10% 的正常請求
```

這樣可以確保所有錯誤和慢請求都被保留。

### 步驟 3：Jaeger 中使用 Adaptive Sampling

Jaeger 會根據流量動態調整採樣率。

### 最終效果

```
10,000 req/s
  ↓ Head Sampling 10%
1,000 req/s (傳到 Collector)
  ↓ Tail Sampling (保留錯誤、慢請求、10% 正常請求)
100 req/s (儲存到 Jaeger)
```

**最終採樣率**：1%

但是：
- **100% 的錯誤請求**都被保留
- **100% 的慢請求**都被保留

## 成本優化技巧

### 1. 分層儲存

不是所有 Trace 都需要保留 90 天。

#### Jaeger 的 Storage Policy

```yaml
# Hot Storage (最近 7 天)
storage:
  type: elasticsearch
  options:
    servers: http://elasticsearch:9200
    index_prefix: jaeger
    
# Warm Storage (7-30 天)
storage_archive:
  type: elasticsearch
  options:
    servers: http://elasticsearch-warm:9200
    index_prefix: jaeger-archive

# Cold Storage (30-90 天)
storage_cold:
  type: s3
  options:
    bucket: jaeger-cold-storage
```

#### Elasticsearch Index Lifecycle Management (ILM)

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### 2. 壓縮 Span 資料

減少 Span 中不必要的 Attribute。

#### 前

```java
span.setAttribute("http.url", "https://api.example.com/api/orders/123?user_id=456&token=xyz...");
span.setAttribute("http.user_agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...");
```

#### 後

```java
span.setAttribute("http.path", "/api/orders/123");
span.setAttribute("http.method", "GET");
```

### 3. 使用 Span Links 而非完整 Trace

對於一些非關鍵的異步操作，不需要完整的 Trace。

#### 範例

```java
// 主要操作
Span mainSpan = tracer.spanBuilder("create-order").startSpan();
try {
    Order order = orderService.create(request);
    
    // 發送郵件（異步）
    Span emailSpan = tracer.spanBuilder("send-email")
        .addLink(mainSpan.getSpanContext())  // 只保留連結，不是完整的 Span
        .startSpan();
    
    emailService.sendAsync(order.getEmail());
    emailSpan.end();
    
} finally {
    mainSpan.end();
}
```

## 監控採樣效果

### 1. 採樣率 Dashboard

```promql
# 實際採樣率
sum(rate(traces_sampled_total[5m])) / sum(rate(traces_total[5m]))

# 每個服務的採樣率
sum by (service_name) (rate(traces_sampled_total[5m])) 
  / 
sum by (service_name) (rate(traces_total[5m]))
```

### 2. 成本 Dashboard

```promql
# 每天的 Span 數量
sum(increase(spans_total[1d]))

# 每天的資料量（假設每個 Span 2 KB）
sum(increase(spans_total[1d])) * 2048

# 每月的儲存成本（假設 $0.023/GB）
sum(increase(spans_total[30d])) * 2048 / 1024 / 1024 / 1024 * 0.023
```

### 3. 錯誤遺漏率

```promql
# 錯誤請求的採樣率（應該是 100%）
sum(rate(traces_sampled_total{status="error"}[5m])) 
  / 
sum(rate(traces_total{status="error"}[5m]))
```

如果這個值 < 1，表示有錯誤請求沒有被採樣，需要調整策略。

## 實戰案例

### 案例 1：電商網站

**需求**：
- 每秒 50,000 個請求
- 預算：$500/month

**策略**：

```yaml
# Head Sampling: 5%
sampler: traceIdRatioBased(0.05)

# Tail Sampling:
policies:
  - name: errors
    type: status_code
    status_code: [ERROR]
  
  - name: checkout
    type: string_attribute
    string_attribute:
      key: http.path
      values: ["/checkout"]  # 結帳流程一定採樣
  
  - name: slow
    type: latency
    latency:
      threshold_ms: 2000
  
  - name: default
    type: probabilistic
    probabilistic:
      sampling_percentage: 2
```

**結果**：
- 最終採樣率：0.1%
- 每天資料量：432 GB
- 每月成本：$298

### 案例 2：金融系統

**需求**：
- 每秒 1,000 個請求
- 所有交易都要保留 Trace（合規要求）

**策略**：

```yaml
# 交易相關的 API：100% 採樣
policies:
  - name: transactions
    type: string_attribute
    string_attribute:
      key: http.path
      values: ["/api/transfer", "/api/payment"]
      sampling_percentage: 100

# 其他 API：1% 採樣
  - name: default
    type: probabilistic
    probabilistic:
      sampling_percentage: 1
```

**結果**：
- 交易 API 採樣率：100%
- 其他 API 採樣率：1%
- 滿足合規要求

## 建議

### 開發環境
- Head Sampling：100%
- Tail Sampling：保留所有

### 測試環境
- Head Sampling：50%
- Tail Sampling：保留錯誤和慢請求

### 生產環境
- Head Sampling：1-5%
- Tail Sampling：保留錯誤、慢請求、VIP 使用者

---

**採樣不是簡單的「保留 1%」，而是要根據業務需求、成本預算、合規要求，設計出最適合的策略。**

好的採樣策略，可以讓你在保持可觀測性的同時，降低 90% 的成本。
