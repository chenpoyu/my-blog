---
layout: post
title: "監控方法論：RED vs USE，該選哪個？"
date: 2023-02-24 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Monitoring, RED Method, USE Method, SRE, Best Practices]
---

前面幾週我們學了很多 Prometheus 的技術細節，但技術只是工具。今天我們要談「哲學」：**到底該監控什麼？**

這不是小問題。我見過太多團隊收集了幾千個指標，結果出問題時還是不知道該看哪裡。

今天要介紹兩個經典方法論：**RED Method** 和 **USE Method**，以及如何在實戰中應用。

> **延伸閱讀**：我們在《Week 4: Grafana 視覺化》中已經使用過 RED Method 的指標，今天會深入探討其背後的哲學和實踐。

## RED Method：從使用者角度出發

RED 是 Google SRE 提出的方法，專注於「使用者體驗」。

**R**ate（速率）、**E**rrors（錯誤）、**D**uration（延遲）

### 1. Rate（速率）：流量健康嗎？

監控「每秒請求數（QPS）」。

```promql
sum(rate(http_requests_total[5m])) by (service)
```

**為什麼重要？**
- 流量突降 → 可能服務掛了，或上游服務掛了
- 流量突增 → 可能被攻擊，或行銷活動開始了

**告警範例**：

```yaml
- alert: TrafficDropped
  expr: |
    sum(rate(http_requests_total[5m])) by (service)
    <
    sum(rate(http_requests_total[5m] offset 1h)) by (service) * 0.5
  for: 10m
  annotations:
    summary: "{{ $labels.service }} 流量下降 50%"
```

### 2. Errors（錯誤）：有多少請求失敗？

監控「錯誤率」。

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)
* 100
```

**為什麼重要？**
- 使用者直接受影響
- 可能影響營收（例如付款失敗）

**告警範例**：

```yaml
- alert: HighErrorRate
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
      /
      sum(rate(http_requests_total[5m])) by (service)
    ) > 0.01
  for: 5m
  annotations:
    summary: "{{ $labels.service }} 錯誤率 > 1%"
```

### 3. Duration（延遲）：回應夠快嗎？

監控「P50、P95、P99 延遲」。

```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
```

**為什麼重要？**
- 延遲高 = 使用者體驗差
- 可能是資料庫慢、外部 API 慢、或程式碼效能問題

**告警範例**：

```yaml
- alert: HighLatency
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
    ) > 1
  for: 10m
  annotations:
    summary: "{{ $labels.service }} P95 延遲 > 1s"
```

## USE Method：從資源角度出發

USE 是 Brendan Gregg（效能調校大師）提出的方法，專注於「系統資源」。

**U**tilization（使用率）、**S**aturation（飽和度）、**E**rrors（錯誤）

### 1. Utilization（使用率）：資源用了多少？

監控「CPU、記憶體、磁碟、網路」的使用率。

```promql
# CPU 使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 記憶體使用率
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 磁碟使用率
(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100
```

**為什麼重要？**
- 使用率 > 80% → 該考慮擴容
- 使用率 < 20% → 可能資源浪費

### 2. Saturation（飽和度）：資源被塞爆了嗎？

監控「等待佇列、執行緒積壓、磁碟 I/O 等待」。

```promql
# Load Average（CPU 等待佇列）
node_load1 / count(node_cpu_seconds_total{mode="idle"}) by (instance)

# 記憶體 Swap 使用
node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes

# 磁碟 I/O 等待
rate(node_disk_io_time_seconds_total[5m])
```

**為什麼重要？**
- 飽和度高 = 資源不夠，排隊中
- 即使使用率只有 60%，如果飽和度高，系統還是會慢

### 3. Errors（錯誤）：硬體或系統層的錯誤

監控「磁碟錯誤、網路丟包、OOM Killer」。

```promql
# 磁碟讀寫錯誤
rate(node_disk_io_errors_total[5m])

# 網路丟包
rate(node_network_receive_drop_total[5m]) + rate(node_network_transmit_drop_total[5m])

# OOM Killer 次數
rate(node_vmstat_oom_kill[5m])
```

## RED vs USE：該選哪個？

這不是單選題，**兩個都要，但用途不同**。

### 用 RED 監控「服務」

適用於：
- Web API
- Microservices
- 任何對外提供服務的元件

**問題導向**：「使用者的體驗如何？」

### 用 USE 監控「資源」

適用於：
- 伺服器（VM、容器）
- 資料庫
- 訊息佇列

**問題導向**：「資源夠不夠用？」

### 實戰組合

我的監控策略是這樣的：

```
┌─────────────────────────────────────┐
│  使用者視角（RED）                    │
│  - API 錯誤率、延遲、流量             │
│  - 業務指標（訂單數、付款成功率）     │
└─────────────────────────────────────┘
              ↓ 出問題時往下挖
┌─────────────────────────────────────┐
│  服務內部（RED）                      │
│  - 資料庫查詢延遲                     │
│  - 外部 API 呼叫成功率                │
│  - 快取命中率                         │
└─────────────────────────────────────┘
              ↓ 還不夠時往下挖
┌─────────────────────────────────────┐
│  系統資源（USE）                      │
│  - CPU、記憶體、磁碟                  │
│  - 網路頻寬                           │
│  - 執行緒數、連線數                   │
└─────────────────────────────────────┘
```

## 實戰案例：從告警到根因

### 場景：半夜 3 點收到告警

**告警內容**：「API 錯誤率 > 5%」

### 第一步：確認影響範圍（RED）

```promql
# 哪些 endpoint 在出錯？
sum(rate(http_requests_total{status=~"5.."}[5m])) by (endpoint)

# 錯誤類型是什麼？
sum(rate(http_requests_total{status=~"5.."}[5m])) by (status)
```

假設發現是 `/api/orders` 的 500 錯誤。

### 第二步：檢查依賴服務（RED）

```promql
# 資料庫查詢有問題嗎？
rate(mysql_global_status_errors_total[5m])

# 外部 API 呼叫失敗嗎？
sum(rate(payment_api_requests_total{status=~"5.."}[5m]))
```

假設發現資料庫查詢錯誤激增。

### 第三步：檢查資源狀態（USE）

```promql
# 資料庫 CPU 滿了嗎？
rate(mysql_global_status_threads_running[5m])

# 連線數爆了嗎？
mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100

# 磁碟 I/O 卡住了嗎？
rate(node_disk_io_time_seconds_total{device="sda"}[5m])
```

假設發現連線數滿了（100%）。

### 結論

根因：**資料庫連線池耗盡**

解決方案：
1. 緊急：重啟連線池（釋放卡住的連線）
2. 短期：調高連線池上限
3. 長期：檢查程式碼是否有連線洩漏（connection leak）

**從告警到根因，總共 5 分鐘。**

這就是 RED + USE 的威力。

## 指標命名的最佳實踐

監控不只是收集數據，還要讓團隊「看得懂」。

### 命名規則

**格式**：`<namespace>_<subsystem>_<metric>_<unit>`

範例：
```
http_requests_total              # 總請求數（counter）
http_request_duration_seconds    # 請求延遲（histogram）
database_connections_active      # 活躍連線數（gauge）
payment_transactions_total       # 交易總數（counter）
```

### 不好的命名

❌ `request_count`（缺少 namespace）  
❌ `http_duration`（缺少 unit）  
❌ `api_latency_ms`（應該用秒，不是毫秒）  
❌ `total_errors`（太模糊）

### 標籤（Labels）的使用

✅ **好的標籤**：
```
http_requests_total{method="GET", endpoint="/api/orders", status="200"}
```

❌ **不好的標籤**：
```
http_requests_total{user_id="12345"}  # 會產生無限多個時間序列！
```

**黃金法則**：標籤的值的「基數（cardinality）」要有限。

- ✅ method: GET, POST, PUT, DELETE（4 種）
- ✅ status: 200, 404, 500, ...（約 50 種）
- ❌ user_id: 1, 2, 3, ...（無限多）

## 哪些指標是每個服務都該有的？

我的「指標標準包」：

### 1. HTTP 服務

```promql
# Rate
http_requests_total{method, endpoint, status}

# Errors
http_requests_total{status=~"5.."}

# Duration
http_request_duration_seconds{method, endpoint}
```

### 2. 資料庫

```promql
# 連線池
db_connections_active
db_connections_idle
db_connections_max

# 查詢
db_queries_total{operation}
db_query_duration_seconds{operation}
db_query_errors_total{operation}
```

### 3. 訊息佇列

```promql
# 生產者
queue_messages_sent_total{topic}

# 消費者
queue_messages_received_total{topic}
queue_messages_failed_total{topic}
queue_lag{topic, partition}  # 積壓量
```

### 4. 業務指標

```promql
# 電商範例
orders_created_total
orders_completed_total
orders_failed_total
revenue_total_dollars
```

---

**監控不是為了收集數據，而是為了回答問題。**

當系統出問題時，好的監控能讓你在 5 分鐘內找到根因。
