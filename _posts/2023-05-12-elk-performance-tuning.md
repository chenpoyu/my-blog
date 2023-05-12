---
layout: post
title: "ELK Stack 效能調校：從慢到快的優化之路"
date: 2023-05-12 09:00:00 +0800
categories: [Observability, Logging]
tags: [Elasticsearch, Performance Tuning, Optimization, JVM, Indexing]
---

你的 Elasticsearch 查詢越來越慢了嗎？

當日誌量從每天 10GB 成長到 100GB，甚至 1TB 時，效能問題就會浮現。

今天我們來談談如何優化 ELK Stack 的效能。

## 效能問題的常見症狀

### 1. 查詢慢

```
在 Kibana 查詢要等 10 秒以上
```

### 2. 寫入慢

```
Logstash 的 bulk queue 一直滿載
```

### 3. 磁碟滿

```
Elasticsearch 自動切換到唯讀模式
```

### 4. CPU 高

```
Elasticsearch 的 CPU 使用率常常 100%
```

### 5. 記憶體不足

```
Elasticsearch 頻繁 GC，甚至 OOM
```

讓我們逐一解決。

## Elasticsearch 硬體配置

### 1. 記憶體分配

Elasticsearch 的記憶體分為兩部分：
- **JVM Heap**：用於快取、聚合計算
- **OS Cache**：用於檔案系統快取（Lucene）

**黃金法則**：
```
JVM Heap = 50% 的實體記憶體（但不超過 31GB）
OS Cache = 剩下的 50%
```

**範例**：

| 實體記憶體 | JVM Heap | OS Cache |
|-----------|----------|----------|
| 8GB | 4GB | 4GB |
| 16GB | 8GB | 8GB |
| 32GB | 16GB | 16GB |
| 64GB | **31GB** | 33GB |
| 128GB | **31GB** | 97GB |

**為什麼不超過 31GB？**

因為 Java 的 Compressed OOPs（壓縮指標）只在 Heap < 32GB 時生效。超過 32GB 反而會浪費記憶體。

**設定方法**：

編輯 `jvm.options`：

```
-Xms16g
-Xmx16g
```

（兩個值要一樣，避免動態調整記憶體）

### 2. 磁碟

- **Hot Nodes**：SSD（NVMe 最好）
- **Warm Nodes**：SSD 或 HDD
- **Cold Nodes**：HDD 或 S3

**RAID 0**：提升 I/O 效能（但無備援，所以要靠 Replica 保證可用性）

### 3. CPU

- **Hot Nodes**：高頻率（如 3.5GHz+）
- **Warm/Cold Nodes**：核心數優先（如 16 cores）

### 4. 網路

- **內網頻寬**：至少 10 Gbps（節點間通訊）
- **外網頻寬**：視查詢流量而定

## JVM 調校

### 1. GC 演算法選擇

Elasticsearch 7.0+ 預設使用 **G1GC**，這已經很好了。

但如果你的 Heap > 16GB，可以試試 **ZGC**（低延遲 GC）：

```
-XX:+UseZGC
```

### 2. GC 日誌

啟用 GC 日誌來分析問題：

```
-Xlog:gc*,gc+age=trace,safepoint:file=/var/log/elasticsearch/gc.log:utctime,pid,tags:filecount=32,filesize=64m
```

### 3. 監控 GC

用以下 API 查看 GC 統計：

```bash
GET _nodes/stats/jvm
```

**警告信號**：
- **Young GC 頻率**：每秒超過 1 次（Heap 太小）
- **Old GC 頻率**：每小時超過 1 次（可能有記憶體洩漏）
- **GC 時間佔比**：超過 10%（Heap 太小或查詢太重）

## 索引效能優化

### 1. Shard 數量

**過多的 Shard 會拖慢查詢**。

**經驗法則**：
- 每個 Shard 大小：10-50GB
- 每個 Node 的 Shard 數量：不超過 20 個/GB Heap

**範例**：

假設你每天產生 100GB 的日誌，JVM Heap 是 16GB，有 3 個 Hot Nodes：

```
每個索引的 Shard 數量 = 100GB / 30GB = 3-4 個
每個 Node 的 Shard 數量 = 4 / 3 ≈ 1-2 個（遠低於 20 * 16 = 320 的上限）
```

所以 **3-4 個 Shard** 是合理的。

### 2. Replica 數量

- **生產環境**：至少 1 個 Replica（保證高可用性）
- **開發環境**：0 個 Replica（節省空間）

### 3. Refresh Interval

Elasticsearch 預設每 1 秒會把記憶體中的資料 flush 到磁碟（變成可搜尋）。

如果你可以接受 **30 秒的延遲**，可以調整：

```json
PUT logs-*/_settings
{
  "index.refresh_interval": "30s"
}
```

**效能提升**：寫入吞吐量提升 30-50%。

### 4. Translog Durability

Translog 是 Elasticsearch 的 WAL（Write-Ahead Log），用來保證資料不丟失。

預設每次寫入都會 fsync（確保資料寫入磁碟），但這很慢。

如果你可以接受 **5 秒內的資料丟失風險**，可以調整：

```json
PUT logs-*/_settings
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "5s"
}
```

**效能提升**：寫入吞吐量提升 2-3 倍。

### 5. Merge Policy

Lucene 的 Segments 會定期合併（Merge），這會消耗大量 I/O。

調整 Merge 速度：

```json
PUT logs-*/_settings
{
  "index.merge.scheduler.max_thread_count": 1  # 預設是 Math.max(1, CPU / 2)
}
```

降低 Merge 的優先級，讓查詢和寫入更順暢。

## Bulk API 優化

### 1. Bulk Size

Logstash 的 Bulk Size 預設是 **125**（125 筆資料一次寫入）。

**最佳實踐**：
- Bulk Size：1000-5000 筆
- Bulk Payload：5-15MB

**Logstash 設定**：

```ruby
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "logs"
    bulk_size => 2000  # 每次寫入 2000 筆
  }
}
```

### 2. Pipeline Workers

Logstash 的 Workers 數量預設是 **CPU 核心數**。

如果你的 Logstash 有 8 個核心，可以增加到 16：

```yaml
# logstash.yml
pipeline.workers: 16
```

### 3. Queue Size

Logstash 的 Queue 預設是 **1000**。

如果你的寫入流量不穩定（如突發流量），可以增加：

```yaml
# logstash.yml
queue.type: persisted
queue.max_bytes: 10gb
```

這樣 Logstash 會把資料先寫入磁碟，避免丟失。

## 查詢效能優化

### 1. 縮小時間範圍

```
# 慢（查詢 30 天）
@timestamp:[now-30d TO now]

# 快（查詢 1 天）
@timestamp:[now-1d TO now]
```

**效能提升**：10-100 倍。

### 2. 使用 Filter Context 而非 Query Context

**Query Context**（計算相關性分數）：
```json
{
  "query": {
    "match": {
      "message": "error"
    }
  }
}
```

**Filter Context**（不計算分數，可快取）：
```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } }
      ]
    }
  }
}
```

**效能提升**：2-5 倍。

### 3. 避免 Wildcard 和 Regex

```
# 慢
message:*timeout*

# 快
message:timeout
```

### 4. 使用 Keyword 而非 Text

```
# 慢（Text 欄位，需要分詞）
message:"order processing failed"

# 快（Keyword 欄位，精確匹配）
errorType:"OrderProcessingException"
```

### 5. Scroll API vs Search After

如果你要匯出大量資料（如超過 10,000 筆），不要用普通的 Search API。

**Scroll API**（適合一次性匯出）：

```bash
POST logs/_search?scroll=1m
{
  "size": 1000
}

# 然後持續呼叫
POST _search/scroll
{
  "scroll": "1m",
  "scroll_id": "..."
}
```

**Search After**（適合分頁）：

```json
{
  "size": 1000,
  "sort": [
    { "@timestamp": "asc" }
  ],
  "search_after": [1682990400000]
}
```

## 監控與告警

### 1. 監控關鍵指標

用 Elasticsearch 的 Monitoring API 或 Grafana + Prometheus。

**關鍵指標**：

| 指標 | 說明 | 告警閾值 |
|------|------|----------|
| **JVM Heap Used %** | JVM 記憶體使用率 | > 85% |
| **Disk Used %** | 磁碟使用率 | > 85% |
| **CPU Usage %** | CPU 使用率 | > 80% |
| **Indexing Rate** | 寫入速度 | 突然下降 50% |
| **Search Latency** | 查詢延遲 | P95 > 5s |
| **GC Time %** | GC 時間佔比 | > 10% |

### 2. 告警設定

在 Kibana：**Stack Management → Watcher**

**範例：磁碟使用率超過 85% 時發送 Slack 通知**

```json
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "http": {
      "request": {
        "url": "http://localhost:9200/_cat/allocation?format=json"
      }
    }
  },
  "condition": {
    "script": {
      "source": "ctx.payload.hits.hits.any(hit -> hit._source.disk.used_percent > 85)"
    }
  },
  "actions": {
    "notify_slack": {
      "webhook": {
        "url": "https://hooks.slack.com/services/..."
      }
    }
  }
}
```

## 實戰案例：從 10 秒降到 1 秒

### 問題

我們的 Kibana 查詢要等 10 秒，完全無法用來即時排查問題。

### 分析

```bash
GET _cat/nodes?v
```

發現：
- **JVM Heap 使用率**：95%（頻繁 GC）
- **Disk Used %**：92%（快滿了）
- **Shard 數量**：每個 Node 有 500+ 個 Shard

### 解決方案

1. **增加記憶體**：
   - 從 16GB 增加到 64GB
   - JVM Heap 設定為 31GB

2. **實作 ILM**：
   - 刪除 30 天前的日誌
   - Disk Used % 降到 60%

3. **減少 Shard 數量**：
   - 從每個索引 10 個 Shard 降到 3 個
   - Shard 總數從 500 降到 150

4. **調整 Refresh Interval**：
   - 從 1 秒改為 30 秒
   - 寫入吞吐量提升 40%

### 結果

- **查詢時間**：從 10 秒降到 1 秒（提升 10 倍）
- **寫入吞吐量**：從 10,000 docs/s 提升到 14,000 docs/s
- **JVM Heap 使用率**：從 95% 降到 60%
- **GC 時間**：從 15% 降到 3%

---

**效能調校不是一次性的工作，而是持續監控與優化的過程。**

每當你的日誌量成長 10 倍，就該重新檢視你的配置。
