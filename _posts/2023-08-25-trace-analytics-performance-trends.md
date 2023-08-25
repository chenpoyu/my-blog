---
layout: post
title: "Trace Analytics 與效能趨勢分析"
date: 2023-08-25 09:00:00 +0800
categories: [Observability, Analytics]
tags: [Tracing, Analytics, Performance, Trends, OpenTelemetry]
---

單個 Trace 可以幫你找到問題，但如果你想知道**系統的整體趨勢**，就需要 Trace Analytics。

今天我們來談談如何從大量的 Trace 中，提取有價值的資訊。

## 從 Trace 到 Metrics

### 問題

Jaeger 適合查看單個 Trace，但不適合回答這些問題：
- 上週的 P95 延遲是多少？
- 哪個服務的錯誤率最高？
- 延遲的趨勢是上升還是下降？

### 解決方案：RED Metrics

從 Trace 中提取 **RED Metrics**：
- **R**ate（請求率）
- **E**rrors（錯誤率）
- **D**uration（延遲）

## Jaeger 的 Service Performance Monitoring (SPM)

Jaeger 1.31+ 支援 SPM，可以從 Trace 中自動產生 Metrics。

### 啟用 SPM

**1. 配置 Jaeger Collector**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-collector-config
data:
  collector.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
    
    processors:
      batch:
      
      # 從 Trace 產生 Metrics
      spanmetrics:
        metrics_exporter: prometheus
        dimensions:
          - name: http.method
          - name: http.status_code
    
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
      
      prometheus:
        endpoint: 0.0.0.0:8889
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [spanmetrics, batch]
          exporters: [jaeger]
        
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [prometheus]
```

**2. Prometheus 抓取 Metrics**：

```yaml
scrape_configs:
  - job_name: 'jaeger-collector'
    static_configs:
      - targets: ['jaeger-collector:8889']
```

**3. 查詢 Metrics**：

```promql
# 請求率
rate(calls_total{service_name="order-service"}[5m])

# 錯誤率
rate(calls_total{service_name="order-service",status_code="STATUS_CODE_ERROR"}[5m]) 
  / 
rate(calls_total{service_name="order-service"}[5m])

# P95 延遲
histogram_quantile(0.95, rate(duration_milliseconds_bucket{service_name="order-service"}[5m]))
```

### Grafana Dashboard

```json
{
  "panels": [
    {
      "title": "Request Rate by Service",
      "targets": [
        {
          "expr": "sum by (service_name) (rate(calls_total[5m]))"
        }
      ]
    },
    {
      "title": "Error Rate by Service",
      "targets": [
        {
          "expr": "sum by (service_name) (rate(calls_total{status_code=\"STATUS_CODE_ERROR\"}[5m])) / sum by (service_name) (rate(calls_total[5m]))"
        }
      ]
    },
    {
      "title": "P95 Latency by Service",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum by (service_name, le) (rate(duration_milliseconds_bucket[5m])))"
        }
      ]
    }
  ]
}
```

## OpenTelemetry Collector 的 Span Metrics Processor

### 配置

```yaml
processors:
  spanmetrics:
    metrics_exporter: prometheus
    latency_histogram_buckets: [2ms, 8ms, 50ms, 100ms, 200ms, 500ms, 1s, 2s, 5s, 10s]
    dimensions:
      - name: http.method
      - name: http.status_code
    dimensions_cache_size: 1000
    aggregation_temporality: "AGGREGATION_TEMPORALITY_CUMULATIVE"

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: traces_spanmetrics

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [spanmetrics, batch]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp, spanmetrics]  # spanmetrics 也是一個 receiver
      processors: [batch]
      exporters: [prometheus]
```

### 產生的 Metrics

```promql
# 請求數
traces_spanmetrics_calls_total{
  service_name="order-service",
  span_name="GET /api/orders",
  http_method="GET",
  http_status_code="200"
}

# 延遲分佈
traces_spanmetrics_latency_bucket{
  service_name="order-service",
  span_name="GET /api/orders",
  le="100"
}
```

## 實戰：效能趨勢分析

### 場景

你想知道過去一個月，系統的效能趨勢。

### 步驟 1：查詢 P95 延遲趨勢

```promql
histogram_quantile(0.95, 
  sum by (service_name, le) (
    rate(duration_milliseconds_bucket[1d])
  )
)
```

**結果**（以 `order-service` 為例）：

```
2023-07-25: 120ms
2023-07-26: 125ms
2023-07-27: 135ms
2023-07-28: 150ms
2023-07-29: 180ms  ← 延遲在上升
```

### 步驟 2：找出延遲上升的原因

用 **heatmap** 查看延遲分佈：

```promql
sum by (le) (
  increase(duration_milliseconds_bucket{service_name="order-service"}[1d])
)
```

**2023-07-25 的分佈**：

```
0-50ms:   10,000 次
50-100ms: 8,000 次
100-200ms: 2,000 次
200ms+:   100 次
```

**2023-07-29 的分佈**：

```
0-50ms:   8,000 次
50-100ms: 7,000 次
100-200ms: 4,000 次  ← 這個範圍增加了
200ms+:   1,000 次  ← 這個範圍也增加了
```

看起來有一部分請求變慢了。

### 步驟 3：分析慢請求的特徵

在 Jaeger 中，篩選延遲 > 200ms 的 Trace：

```
service=order-service
minDuration=200ms
```

發現這些慢請求都有一個共同點：都呼叫了 `recommendation-service`。

### 步驟 4：查看 recommendation-service 的變更

```bash
git log --since="2023-07-28" --until="2023-07-29" -- recommendation-service/
```

發現在 7/28 部署了一個新版本，加入了一個機器學習模型。

**根本原因**：新的機器學習模型拖慢了回應時間。

## Trace 的聚合查詢

### Elasticsearch 的 Aggregation

Jaeger 的後端是 Elasticsearch，可以用 Aggregation 做複雜的查詢。

#### 範例 1：每小時的請求數

```json
GET jaeger-span-*/_search
{
  "size": 0,
  "query": {
    "range": {
      "startTime": {
        "gte": "now-1d",
        "lte": "now"
      }
    }
  },
  "aggs": {
    "requests_per_hour": {
      "date_histogram": {
        "field": "startTime",
        "fixed_interval": "1h"
      }
    }
  }
}
```

#### 範例 2：最常呼叫的 Endpoint

```json
GET jaeger-span-*/_search
{
  "size": 0,
  "query": {
    "term": {
      "process.serviceName": "order-service"
    }
  },
  "aggs": {
    "top_endpoints": {
      "terms": {
        "field": "operationName",
        "size": 10
      }
    }
  }
}
```

#### 範例 3：P95 延遲

```json
GET jaeger-span-*/_search
{
  "size": 0,
  "query": {
    "term": {
      "process.serviceName": "order-service"
    }
  },
  "aggs": {
    "latency_percentiles": {
      "percentiles": {
        "field": "duration",
        "percents": [50, 95, 99]
      }
    }
  }
}
```

## Trace 的異常偵測

### 方法 1：基於統計的異常偵測

計算每個 Endpoint 的延遲分佈，找出超過 **3σ** 的請求。

```promql
# 平均延遲
avg_over_time(histogram_quantile(0.5, rate(duration_milliseconds_bucket[5m]))[7d:1h])

# 標準差
stddev_over_time(histogram_quantile(0.5, rate(duration_milliseconds_bucket[5m]))[7d:1h])

# 異常：延遲 > 平均 + 3 * 標準差
histogram_quantile(0.95, rate(duration_milliseconds_bucket[5m])) 
  > 
avg_over_time(histogram_quantile(0.5, rate(duration_milliseconds_bucket[5m]))[7d:1h]) 
  + 
3 * stddev_over_time(histogram_quantile(0.5, rate(duration_milliseconds_bucket[5m]))[7d:1h])
```

### 方法 2：機器學習異常偵測

用 Elasticsearch 的 Machine Learning 功能。

#### 建立異常偵測工作

```json
PUT _ml/anomaly_detectors/trace-latency-detector
{
  "analysis_config": {
    "bucket_span": "5m",
    "detectors": [
      {
        "function": "mean",
        "field_name": "duration",
        "by_field_name": "operationName"
      }
    ]
  },
  "data_description": {
    "time_field": "startTime"
  }
}
```

#### 啟動偵測

```json
POST _ml/anomaly_detectors/trace-latency-detector/_open
POST _ml/anomaly_detectors/trace-latency-detector/_start
```

Elasticsearch 會自動學習正常的延遲範圍，並標記異常的請求。

## 效能回歸偵測

### 問題

每次部署後，你想知道效能有沒有退步。

### 解決方案：A/B 測試

#### 步驟 1：部署新版本到一部分流量

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        user-agent:
          regex: ".*Chrome.*"
    route:
    - destination:
        host: order-service
        subset: v2  # 新版本
      weight: 10
    - destination:
        host: order-service
        subset: v1  # 舊版本
      weight: 90
```

#### 步驟 2：比較兩個版本的 P95 延遲

```promql
# v1 的 P95 延遲
histogram_quantile(0.95, 
  sum by (le) (
    rate(duration_milliseconds_bucket{service_name="order-service",version="v1"}[5m])
  )
)

# v2 的 P95 延遲
histogram_quantile(0.95, 
  sum by (le) (
    rate(duration_milliseconds_bucket{service_name="order-service",version="v2"}[5m])
  )
)
```

#### 步驟 3：統計檢定

用 **Mann-Whitney U test** 判斷兩個版本的延遲是否有顯著差異。

**Python 範例**：

```python
from scipy.stats import mannwhitneyu

v1_latencies = [120, 125, 130, 135, 140]  # 從 Prometheus 取得
v2_latencies = [150, 155, 160, 165, 170]

statistic, p_value = mannwhitneyu(v1_latencies, v2_latencies)

if p_value < 0.05:
    print("v2 的延遲顯著高於 v1，建議回滾")
else:
    print("v2 的延遲與 v1 沒有顯著差異")
```

## 成本分析

### 哪個服務最貴？

計算每個服務的「成本」：

```
成本 = 請求數 × 平均延遲
```

**Prometheus 查詢**：

```promql
sum by (service_name) (rate(calls_total[5m])) 
  * 
sum by (service_name) (rate(duration_milliseconds_sum[5m])) 
  / 
sum by (service_name) (rate(duration_milliseconds_count[5m]))
```

**結果**：

```
order-service:        1000 req/s × 100ms = 100,000 ms/s
payment-service:      500 req/s × 200ms = 100,000 ms/s
recommendation-service: 200 req/s × 1000ms = 200,000 ms/s  ← 最貴
```

雖然 `recommendation-service` 的請求數最少，但因為延遲很高，所以是最貴的服務。

### 優化方向

1. **快取**：減少延遲
2. **異步處理**：不要讓使用者等待
3. **降級**：在高負載時，暫時關閉推薦功能

## 實戰建議

### 1. 定期回顧效能趨勢

每週回顧一次 P95 延遲、錯誤率、請求數的趨勢。

### 2. 設定效能目標

**範例**：

| 服務          | P95 延遲 | 錯誤率 |
|---------------|----------|--------|
| order-service | < 200ms  | < 1%   |
| payment-service | < 500ms | < 0.1% |

### 3. 建立效能回歸檢測

在 CI/CD 中，加入效能測試：

```bash
# 部署到測試環境
kubectl apply -f deployment-v2.yaml

# 等待 5 分鐘
sleep 300

# 比較 v1 和 v2 的 P95 延遲
v1_p95=$(curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95, rate(duration_milliseconds_bucket{version=\"v1\"}[5m]))" | jq '.data.result[0].value[1]')
v2_p95=$(curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95, rate(duration_milliseconds_bucket{version=\"v2\"}[5m]))" | jq '.data.result[0].value[1]')

if (( $(echo "$v2_p95 > $v1_p95 * 1.1" | bc -l) )); then
  echo "v2 的延遲比 v1 高 10% 以上，建議回滾"
  exit 1
fi
```

---

**單個 Trace 可以幫你找到問題，Trace Analytics 可以幫你預防問題。**

當你能從趨勢中發現問題的徵兆，你就能在使用者發現之前解決它。
