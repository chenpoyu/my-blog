---
layout: post
title: "ELK Stack 故障排查實戰手冊"
date: 2023-06-16 09:00:00 +0800
categories: [Observability, Logging, Troubleshooting]
tags: [ELK Stack, Troubleshooting, Performance Issues, Common Problems]
---

凌晨 3 點,PagerDuty 響了：**「Elasticsearch 叢集 CPU 100%」**

你該怎麼辦？

今天我們來整理**ELK Stack 最常見的問題與解決方案**。

## 問題 1：Elasticsearch 查詢很慢

### 症狀

```
Kibana 查詢要等 30 秒以上
```

### 診斷

#### 1. 檢查 Slow Log

```bash
GET _nodes/stats/indices/search
```

或查看 Slow Log：

```bash
tail -f /var/log/elasticsearch/my-cluster_index_search_slowlog.log
```

#### 2. 確認查詢模式

```bash
GET _cat/thread_pool/search?v
```

如果 `queue` 很高,表示查詢請求堆積。

### 常見原因與解決方案

#### 原因 1：時間範圍太大

```
# 慢
過去 30 天的所有日誌

# 快
過去 1 小時的日誌
```

#### 原因 2：Shard 數量太多

```bash
GET _cat/shards?v | wc -l
# 如果超過 1000 個 Shard,考慮減少
```

**解決**：用 ILM Shrink 或減少新索引的 Shard 數量。

#### 原因 3：Heap 使用率過高

```bash
GET _cat/nodes?v&h=name,heap.percent
```

如果超過 85%,表示記憶體不足。

**解決**：增加 JVM Heap 或減少查詢併發。

#### 原因 4：磁碟 I/O 過高

```bash
iostat -x 1
```

如果 `%util` 接近 100%,表示磁碟瓶頸。

**解決**：升級到 SSD 或增加節點。

## 問題 2：Elasticsearch 寫入慢

### 症狀

```
Logstash 的 bulk queue 滿載
應用程式日誌延遲 10 分鐘才出現在 Kibana
```

### 診斷

```bash
GET _cat/thread_pool/write?v
```

如果 `queue` 很高,表示寫入請求堆積。

### 常見原因與解決方案

#### 原因 1：Refresh Interval 太短

預設 1 秒,可以調整到 30 秒：

```json
PUT logs-*/_settings
{
  "index.refresh_interval": "30s"
}
```

**效能提升**：50%

#### 原因 2：Replica 太多

如果你有 3 個節點,設定 2 個 Replica 會導致每筆資料寫 3 次。

```json
PUT logs-*/_settings
{
  "index.number_of_replicas": 1
}
```

#### 原因 3：Translog 同步模式

預設每次寫入都 fsync（很慢）。

```json
PUT logs-*/_settings
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "5s"
}
```

**效能提升**：3 倍（但可能丟失 5 秒內的資料）

#### 原因 4：Merge 太頻繁

```bash
GET _cat/nodes?v&h=name,merges.current
```

如果 `merges.current` 很高,表示 Merge 太頻繁。

**解決**：降低 Merge 優先級。

```json
PUT logs-*/_settings
{
  "index.merge.scheduler.max_thread_count": 1
}
```

## 問題 3：Elasticsearch 磁碟滿了

### 症狀

```
Elasticsearch 自動切換到唯讀模式
無法寫入新日誌
```

### 診斷

```bash
GET _cat/allocation?v
```

如果 `disk.percent` > 85%,Elasticsearch 會停止寫入。

### 解決方案

#### 方案 1：刪除舊日誌

```bash
DELETE logs-2023.01.*
```

#### 方案 2：實作 ILM

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "1d"
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### 方案 3：增加磁碟

在 AWS 直接擴展 EBS Volume：

```bash
aws ec2 modify-volume --volume-id vol-xxx --size 1000
```

#### 方案 4：移到 S3

```json
{
  "cold": {
    "min_age": "30d",
    "actions": {
      "searchable_snapshot": {
        "snapshot_repository": "my-s3-repo"
      }
    }
  }
}
```

## 問題 4：Logstash CPU 100%

### 症狀

```
Logstash 的 CPU 使用率持續 100%
日誌處理延遲
```

### 診斷

```bash
GET http://logstash:9600/_node/stats/pipelines
```

查看 `queue` 的使用率。

### 常見原因與解決方案

#### 原因 1：Grok Pattern 太複雜

Grok 是 Logstash 最慢的過濾器。

**解決**：
1. 簡化 Grok Pattern
2. 用條件判斷減少 Grok 嘗試次數
3. 改用結構化日誌（JSON）

#### 原因 2：Workers 不足

```yaml
# logstash.yml
pipeline.workers: 16  # 增加到 2 倍 CPU 核心數
```

#### 原因 3：Bulk Size 太小

```ruby
output {
  elasticsearch {
    bulk_size => 2000  # 增加到 2000-5000
  }
}
```

## 問題 5：Kafka Lag 很高

### 症狀

```
Kafka Consumer Lag > 100,000
日誌延遲 30 分鐘才出現在 Kibana
```

### 診斷

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group logstash-app
```

### 解決方案

#### 方案 1：增加 Logstash 實例

從 2 台增加到 4 台。

#### 方案 2：增加 Kafka Partitions

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --alter --topic app-logs --partitions 20
```

#### 方案 3：優化 Logstash Pipeline

移除不必要的過濾器。

## 問題 6：Elasticsearch 記憶體不足（OOM）

### 症狀

```
java.lang.OutOfMemoryError: Java heap space
Elasticsearch 節點不斷重啟
```

### 診斷

```bash
GET _nodes/stats/jvm
```

查看 `heap_used_percent`。

### 解決方案

#### 方案 1：增加 JVM Heap

```
# jvm.options
-Xms32g
-Xmx32g
```

（但不要超過 31GB）

#### 方案 2：減少 Shard 數量

```bash
GET _cat/shards?v | wc -l
```

如果超過 `20 * Heap GB`,就太多了。

**解決**：用 Shrink API 或減少新索引的 Shard 數量。

#### 方案 3：清除 Field Data Cache

```bash
POST logs-*/_cache/clear?fielddata=true
```

#### 方案 4：關閉不必要的功能

```json
PUT logs-*/_settings
{
  "index.queries.cache.enabled": false
}
```

## 問題 7：Kibana 登入很慢

### 症狀

```
Kibana 登入要等 30 秒
```

### 診斷

檢查 Elasticsearch 的健康狀態：

```bash
GET _cluster/health
```

如果是 `yellow` 或 `red`,表示叢集有問題。

### 解決方案

#### 方案 1：檢查 Unassigned Shards

```bash
GET _cat/shards?v | grep UNASSIGNED
```

手動分配：

```bash
POST _cluster/reroute
{
  "commands": [
    {
      "allocate_replica": {
        "index": "logs-2023.06.16",
        "shard": 0,
        "node": "es-node-2"
      }
    }
  ]
}
```

#### 方案 2：增加 Kibana 記憶體

```yaml
# kibana.yml
elasticsearch.requestTimeout: 90000
```

## 問題 8：資料丟失

### 症狀

```
某些日誌沒有出現在 Elasticsearch
```

### 診斷

#### 1. 檢查 Logstash 錯誤日誌

```bash
tail -f /var/log/logstash/logstash-plain.log
```

#### 2. 檢查 Elasticsearch Bulk Errors

```bash
GET logs-*/_search
{
  "query": {
    "match": {
      "error": "bulk"
    }
  }
}
```

### 常見原因與解決方案

#### 原因 1：Mapping 錯誤

```json
{
  "error": {
    "type": "mapper_parsing_exception",
    "reason": "failed to parse field [timestamp] of type [date]"
  }
}
```

**解決**：修正時間格式或更新 Mapping。

#### 原因 2：Kafka Retention 太短

Kafka 預設保留 7 天,如果 Logstash 消費不及時,資料會被刪除。

**解決**：增加 Retention。

```properties
log.retention.hours=168  # 7 天
```

## 快速檢查清單

遇到問題時,依序檢查：

### 1. 叢集健康狀態

```bash
GET _cluster/health
```

### 2. 節點狀態

```bash
GET _cat/nodes?v
```

### 3. Heap 使用率

```bash
GET _cat/nodes?v&h=name,heap.percent
```

### 4. 磁碟使用率

```bash
GET _cat/allocation?v
```

### 5. Pending Tasks

```bash
GET _cat/pending_tasks?v
```

### 6. Thread Pool

```bash
GET _cat/thread_pool?v
```

---

**大部分問題的根因都是：記憶體不足、磁碟滿了、Shard 太多。**

掌握這些基本診斷技巧,你就能快速定位問題。
