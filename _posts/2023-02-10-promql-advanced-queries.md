---
layout: post
title: "PromQL 進階：讓數據開口說出問題在哪"
date: 2023-02-10 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Prometheus, PromQL, Query, Time Series]
---

前面幾週我們學會了基本的 PromQL 查詢，但如果只會 `up` 和 `rate()`，那就像只會加減法卻要解微積分。

今天我們要深入 PromQL 的強大功能，學會：**從海量數據中挖掘出隱藏的問題。**

> 本文涵蓋: **PromQL 聚合、子查詢、趨勢分析**

## PromQL 不只是查詢，是數據分析

很多人把 PromQL 當成「SQL for metrics」，這是誤解。

SQL 是針對「靜態數據」做查詢（查詢後數據不變），而 PromQL 是針對「時間序列」做分析（數據隨時間變化）。

舉個例子，你想知道「過去一小時 API 流量的成長率」，SQL 做不到，但 PromQL 可以：

```promql
rate(http_requests_total[1h]) / rate(http_requests_total[1h] offset 1h)
```

這就是時間序列資料庫的威力。

## 聚合運算：從細節到全局

### 場景 1：找出最慢的 API

你有 50 個 API endpoint，想知道哪個最慢：

```promql
topk(5, 
  avg by (endpoint) (
    rate(http_request_duration_seconds_sum[5m]) 
    / 
    rate(http_request_duration_seconds_count[5m])
  )
)
```

解析：
- `rate(..._sum) / rate(..._count)`：計算平均延遲
- `avg by (endpoint)`：按 endpoint 分組
- `topk(5, ...)`：取前 5 名

### 場景 2：找出流量突增的服務

```promql
sort_desc(
  sum by (job) (rate(http_requests_total[5m]))
)
```

### 場景 3：計算總體成功率

```promql
sum(rate(http_requests_total{status=~"2.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100
```

這會給你一個百分比，例如 `99.5`（代表 99.5% 成功率）。

## 時間偏移：比較「現在」與「過去」

### 場景 4：與昨天同時段比較

```promql
# 現在的流量
sum(rate(http_requests_total[5m]))

# 昨天同時段的流量
sum(rate(http_requests_total[5m] offset 24h))

# 成長率
(
  sum(rate(http_requests_total[5m]))
  /
  sum(rate(http_requests_total[5m] offset 24h))
  - 1
) * 100
```

如果結果是 `50`，代表流量比昨天多 50%。

### 場景 5：找出異常波動

```promql
abs(
  rate(http_requests_total[5m])
  -
  rate(http_requests_total[5m] offset 1h)
) > 100
```

如果流量在一小時內變化超過 100 req/s，就標記為異常。

## 子查詢：查詢的查詢

這是 PromQL 的高級功能，可以對查詢結果再做查詢。

### 場景 6：計算「過去一小時內，P95 延遲的最大值」

```promql
max_over_time(
  histogram_quantile(0.95,
    rate(http_request_duration_seconds_bucket[5m])
  )[1h:1m]
)
```

解析：
- 內層：計算 P95 延遲（每 5 分鐘的 rate）
- `[1h:1m]`：對過去 1 小時的數據，每 1 分鐘取一個點
- `max_over_time`：找出這 60 個點的最大值

這對容量規劃很有用：「在流量高峰時，我們的 P95 最慢到多少？」

## 預測未來：線性回歸

Prometheus 內建線性回歸函數，可以預測趨勢。

### 場景 7：預測磁碟何時會滿

```promql
predict_linear(
  node_filesystem_free_bytes{mountpoint="/"}[1h], 
  4 * 3600
)
```

這會基於過去 1 小時的趨勢，預測 4 小時後磁碟剩餘空間。

如果結果是負數，代表「如果照這個速度，4 小時後磁碟會滿」。

### 場景 8：預測記憶體洩漏

```promql
predict_linear(
  process_resident_memory_bytes[6h],
  24 * 3600
) > 8 * 1024 * 1024 * 1024  # 8GB
```

如果預測 24 小時後記憶體會超過 8GB，就該檢查是否有記憶體洩漏。

## 告警的進階寫法

結合上面的技巧，我們可以寫出更智慧的告警。

### 告警 1：與歷史基準比較

```yaml
- alert: TrafficAnomalyHigh
  expr: |
    (
      sum(rate(http_requests_total[5m]))
      /
      avg_over_time(sum(rate(http_requests_total[5m]))[7d:1h])
    ) > 3
  for: 10m
  annotations:
    summary: "流量異常飆高"
    description: "當前流量是過去 7 天平均值的 {{ $value }} 倍"
```

這會計算「過去 7 天同時段的平均流量」，如果當前流量是平均值的 3 倍以上，就告警。

這比固定閾值（例如 > 1000 req/s）更聰明，因為它會隨著業務成長自動調整。

### 告警 2：多維度綜合判斷

```yaml
- alert: ServiceDegraded
  expr: |
    (
      rate(http_requests_total{status=~"5.."}[5m]) > 10
    ) and (
      histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
    )
  for: 5m
  annotations:
    summary: "服務降級"
    description: "同時出現高錯誤率和高延遲"
```

只有當「錯誤率高」**且**「延遲高」時才告警，避免誤報。

## 效能優化：PromQL 的陷阱

PromQL 很強大,但也很容易寫出慢查詢。

### ❌ 陷阱 1：過大的時間範圍

錯誤：
```promql
rate(http_requests_total[24h])  # 太大了！
```

`rate()` 應該用短時間範圍（5m、10m），長時間範圍用 `increase()`：
```promql
increase(http_requests_total[24h])
```

### ❌ 陷阱 2：沒有過濾 Label

錯誤：
```promql
sum(rate(http_requests_total[5m]))  # 沒有 by
```

這會掃描所有時間序列。應該加上 `by` 或 `without`：
```promql
sum by (job, instance) (rate(http_requests_total[5m]))
```

### ❌ 陷阱 3：在 Dashboard 用複雜的子查詢

子查詢很消耗資源，不要在高頻刷新的 Dashboard 上使用。

應該用 Recording Rules 預先計算：

```yaml
groups:
  - name: precomputed
    interval: 1m
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

然後在 Dashboard 直接用 `job:http_requests:rate5m`，快很多。

## 實戰案例：找出資料庫慢查詢的根因

假設你的 API 變慢了，你懷疑是資料庫的問題。

### 第一步：確認是資料庫

```promql
rate(mysql_global_status_slow_queries[5m])
```

如果這個數字在增加，確實是資料庫慢查詢增多了。

### 第二步：找出是哪個表

```promql
topk(5,
  sum by (table) (rate(mysql_table_rows_read[5m]))
)
```

假設發現是 `orders` 表。

### 第三步：找出是哪個查詢類型

```promql
sum by (command) (rate(mysql_global_status_commands_total{command=~"select|update|insert"}[5m]))
```

假設發現是 `select` 激增。

### 第四步：關聯到 API

```promql
sum by (endpoint) (
  rate(http_requests_total{path=~".*order.*"}[5m])
)
```

找出哪個 API 在狂打 `orders` 表。

這就是「從現象到根因」的數據偵探過程。

## 容量規劃：預測何時需要擴容

這是老闆最關心的問題：「我們什麼時候需要加機器？」

### 方法 1：基於趨勢預測

```promql
predict_linear(
  sum(rate(http_requests_total[1h]))[7d:1h],
  30 * 24 * 3600  # 30 天後
) > 10000  # 假設單機極限是 10000 req/s
```

如果預測 30 天後流量會超過單機極限，該準備擴容了。

### 方法 2：基於使用率

```promql
(
  sum(rate(http_requests_total[5m]))
  /
  10000  # 單機極限
) > 0.7  # 70% 使用率
```

當使用率超過 70%，就該考慮加機器（不要等到 100% 才擴容）。

---

**PromQL 不是用來炫技的，而是用來快速定位問題的。**

當你能在 5 分鐘內從「API 變慢了」找到「是因為 orders 表的 select 查詢激增」，你就是團隊的救星。
