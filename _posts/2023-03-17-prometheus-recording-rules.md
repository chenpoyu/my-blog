---
layout: post
title: "Recording Rules：預先計算，加速查詢"
date: 2023-03-17 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Prometheus, Recording Rules, Performance, Optimization]
---

過去幾週我們寫了很多複雜的 PromQL 查詢。但你有沒有想過：**每次查詢都重新計算，Prometheus 不會累死嗎？**

今天我們要學一個效能優化的絕招：**Recording Rules**——讓 Prometheus 預先計算好結果。

> **前置知識**：這裡使用的 CPU 使用率、錯誤率、P95 延遲等查詢，我們在前面的文章已經介紹過（Week 4: Grafana、Week 5: PromQL 進階、Week 8: 監控方法論）。這裡專注於如何透過 Recording Rules 優化效能。

## 為什麼需要 Recording Rules？

### 問題場景

你有一個複雜的查詢：

```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service, method)
) > 1
```

每次在 Grafana Dashboard 重新整理，這個查詢都要：
1. 掃描所有 `http_request_duration_seconds_bucket` 時間序列
2. 計算 `rate()`
3. 聚合 `sum()`
4. 計算 `histogram_quantile()`

如果你有 100 個服務、50 個 method，這就是 5000 個時間序列。**查詢可能要幾秒鐘。**

### 解決方案：Recording Rules

把這個複雜查詢「預先計算」，每 15 秒存一次結果：

```yaml
groups:
  - name: api_latency
    interval: 15s
    rules:
      - record: job:http_request_duration_seconds:p95
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service, method)
          )
```

然後在 Dashboard 直接用：

```promql
job:http_request_duration_seconds:p95 > 1
```

**查詢從 3 秒變成 0.1 秒。**

## Recording Rules 的命名規範

這不是隨便取名字，Prometheus 社群有標準：

**格式**：`level:metric:operations`

- **level**：聚合層級（job, instance, cluster）
- **metric**：原始指標名稱
- **operations**：做了什麼運算（sum, rate, avg, p95）

### 範例

```yaml
# Job 層級，HTTP 請求的 5 分鐘速率
- record: job:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m])) by (job)

# Instance 層級，CPU 使用率
- record: instance:node_cpu:utilization
  expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Cluster 層級，總請求數
- record: cluster:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m]))
```

### 不好的命名

❌ `my_custom_metric`（看不出來是預計算的）  
❌ `api_p95`（缺少原始指標名稱）  
❌ `http_requests_5m_sum`（格式錯誤）

## 實戰：常用的 Recording Rules

### 1. API 效能指標

```yaml
groups:
  - name: api_performance
    interval: 30s
    rules:
      # 每秒請求數
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job, method, endpoint)
      
      # 錯誤率
      - record: job:http_requests:error_rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
      
      # P50 延遲
      - record: job:http_request_duration:p50
        expr: |
          histogram_quantile(0.50,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )
      
      # P95 延遲
      - record: job:http_request_duration:p95
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )
      
      # P99 延遲
      - record: job:http_request_duration:p99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )
```

### 2. 資料庫連線池

```yaml
groups:
  - name: database_pool
    interval: 30s
    rules:
      # 連線池使用率
      - record: job:db_connections:utilization
        expr: |
          (
            db_connections_active / db_connections_max
          ) * 100
      
      # 連線等待數
      - record: job:db_connections:waiting
        expr: db_connections_waiting
      
      # 平均查詢時間
      - record: job:db_query_duration:avg
        expr: |
          rate(db_query_duration_seconds_sum[5m])
          /
          rate(db_query_duration_seconds_count[5m])
```

### 3. 節點資源使用

```yaml
groups:
  - name: node_resources
    interval: 30s
    rules:
      # CPU 使用率
      - record: instance:node_cpu:utilization
        expr: |
          100 - (
            avg by (instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100
          )
      
      # 記憶體使用率
      - record: instance:node_memory:utilization
        expr: |
          (
            1 - (
              node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
            )
          ) * 100
      
      # 磁碟使用率
      - record: instance:node_disk:utilization
        expr: |
          (
            node_filesystem_size_bytes - node_filesystem_free_bytes
          ) / node_filesystem_size_bytes * 100
      
      # 網路流量（Mbps）
      - record: instance:node_network:receive_mbps
        expr: rate(node_network_receive_bytes_total[5m]) * 8 / 1024 / 1024
      
      - record: instance:node_network:transmit_mbps
        expr: rate(node_network_transmit_bytes_total[5m]) * 8 / 1024 / 1024
```

## 分層計算：從細到粗

Recording Rules 可以「疊加」，從細粒度算到粗粒度。

```yaml
groups:
  - name: http_requests_layered
    interval: 30s
    rules:
      # 第一層：instance 層級
      - record: instance:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (instance, job)
      
      # 第二層：job 層級（基於第一層）
      - record: job:http_requests:rate5m
        expr: sum(instance:http_requests:rate5m) by (job)
      
      # 第三層：cluster 層級（基於第二層）
      - record: cluster:http_requests:rate5m
        expr: sum(job:http_requests:rate5m)
```

**好處**：
- 每一層都可以單獨查詢
- 避免重複計算
- 查詢速度極快

## 搭配告警使用

Recording Rules 最大的用途之一，就是簡化告警規則。

### 改進前（慢）

```yaml
- alert: HighLatency
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
    ) > 1
  for: 5m
```

每 15 秒評估這個複雜查詢，消耗資源。

### 改進後（快）

```yaml
# Recording Rule
- record: job:http_request_duration:p95
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
    )

# 告警規則（簡單）
- alert: HighLatency
  expr: job:http_request_duration:p95 > 1
  for: 5m
```

**評估時間從 500ms 降到 5ms。**

## 效能優化技巧

### 1. 調整 interval

不是所有 Recording Rules 都需要高頻率。

```yaml
groups:
  # 即時監控（高頻）
  - name: realtime_metrics
    interval: 15s
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
  
  # 趨勢分析（低頻）
  - name: trend_metrics
    interval: 5m
    rules:
      - record: job:http_requests:rate1h
        expr: sum(rate(http_requests_total[1h])) by (job)
```

### 2. 避免過度聚合

錯誤：

```yaml
# 把所有 label 都保留
- record: job:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m])) by (job, method, endpoint, status, user_agent, ...)
```

這會產生幾千個時間序列，失去 Recording Rule 的意義。

正確：

```yaml
# 只保留關鍵 label
- record: job:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m])) by (job, method, status)
```

### 3. 定期清理無用的 Recording Rules

用 `promtool` 檢查哪些 Recording Rules 從未被查詢：

```bash
# 查詢 Recording Rule 的使用次數
promtool query instant http://prometheus:9090 \
  'count by (__name__) (your_recording_rule_name)'
```

如果結果是 0，代表這個 Recording Rule 沒人用，可以刪掉。

## 實戰案例：Dashboard 效能優化

### 改進前

某個 Dashboard 有 20 個 Panel，每個都查詢原始指標：

```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
sum(rate(http_requests_total[5m])) by (service)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
# ...還有 17 個
```

**載入時間：12 秒**

### 改進後

建立 Recording Rules：

```yaml
groups:
  - name: dashboard_metrics
    interval: 30s
    rules:
      - record: service:http_request_duration:p95
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
      
      - record: service:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (service)
      
      - record: service:http_requests:error_rate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
```

Dashboard 改用 Recording Rules：

```promql
service:http_request_duration:p95
service:http_requests:rate5m
service:http_requests:error_rate
```

**載入時間：0.8 秒**

## 監控 Recording Rules 本身

Recording Rules 也可能出問題。

### 檢查評估時間

```promql
# 哪個 Recording Rule 最慢？
topk(10, prometheus_rule_evaluation_duration_seconds)

# 哪個 Recording Rule 失敗了？
prometheus_rule_evaluation_failures_total > 0
```

### 告警：Recording Rule 失敗

```yaml
- alert: RecordingRuleFailed
  expr: increase(prometheus_rule_evaluation_failures_total[5m]) > 0
  for: 5m
  annotations:
    summary: "Recording Rule {{ $labels.rule_group }} 評估失敗"
    description: "過去 5 分鐘失敗了 {{ $value }} 次"
```

---

**效能優化不是過早優化，而是在發現瓶頸後有系統地解決。**

當你的 Dashboard 從 10 秒載入變成 1 秒，團隊就會更願意使用監控，形成良性循環。
