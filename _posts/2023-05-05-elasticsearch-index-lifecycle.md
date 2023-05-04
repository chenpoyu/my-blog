---
layout: post
title: "Elasticsearch Index Lifecycle：高效管理海量日誌"
date: 2023-05-05 09:00:00 +0800
categories: [Observability, Logging]
tags: [Elasticsearch, ILM, Index Lifecycle, Data Management, Cost Optimization]
---

你的 Elasticsearch 叢集硬碟快滿了嗎？

這是每個使用 ELK Stack 的團隊都會遇到的問題：**日誌成長速度太快，儲存成本爆炸。**

今天我們來談談如何用 **Index Lifecycle Management (ILM)** 優雅地管理日誌的生命週期。

## 日誌的生命週期

日誌不是永遠都重要的。

**過去 24 小時的日誌**：
- 需要**快速查詢**（排查問題）
- 需要**高可用性**（隨時可能被調閱）
- 可以接受**高成本**（SSD、熱節點）

**過去 7-30 天的日誌**：
- 偶爾會查詢（分析趨勢）
- 可以接受**較慢的查詢**
- 應該**降低成本**（HDD、溫節點）

**超過 30 天的日誌**：
- 幾乎不會查詢（合規留存）
- 可以接受**非常慢的查詢**
- 必須**最小化成本**（冷節點、S3）

**超過 90 天的日誌**：
- 直接刪除（或封存到 S3）

這就是 **Hot-Warm-Cold 架構**。

## ILM 的四個階段

Elasticsearch 的 ILM 將日誌分為四個階段：

| 階段 | 說明 | 典型時間 | 儲存成本 |
|------|------|----------|----------|
| **Hot** | 正在寫入，頻繁查詢 | 0-1 天 | 高（SSD） |
| **Warm** | 唯讀，偶爾查詢 | 1-30 天 | 中（HDD） |
| **Cold** | 唯讀，很少查詢 | 30-90 天 | 低（冷儲存） |
| **Delete** | 刪除 | 90 天後 | 無 |

## 實作 ILM Policy

### 步驟 1：建立 ILM Policy

在 Kibana：**Stack Management → Index Lifecycle Policies → Create policy**

或用 API：

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "1d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "readonly": {}
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "my-s3-repo"
          }
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

讓我們逐一解釋每個階段。

### Hot Phase：資料寫入與頻繁查詢

```json
"hot": {
  "actions": {
    "rollover": {
      "max_primary_shard_size": "50GB",  // 單一 Shard 達到 50GB 時
      "max_age": "1d"                     // 或索引存在超過 1 天時
    }
  }
}
```

**Rollover** 的概念：

假設你的索引是 `logs-2023.05.05`，當它達到 50GB 或存在超過 1 天時，Elasticsearch 會：
1. 建立新索引 `logs-2023.05.06`
2. 把寫入導向新索引
3. 舊索引進入 Warm Phase

這樣每個索引的大小都可控，查詢效能更好。

### Warm Phase：唯讀與壓縮

```json
"warm": {
  "min_age": "1d",  // 進入 Hot Phase 後 1 天
  "actions": {
    "shrink": {
      "number_of_shards": 1  // 把多個 Shard 合併成 1 個
    },
    "forcemerge": {
      "max_num_segments": 1  // 合併 Lucene Segments（提升查詢效能）
    },
    "readonly": {}  // 設定為唯讀
  }
}
```

**為什麼要 Shrink？**

Hot Phase 時你可能用 5 個 Shard（分散寫入壓力），但 Warm Phase 不再寫入了，5 個 Shard 只會浪費資源。

**為什麼要 Forcemerge？**

Lucene 的 Segments 越多，查詢越慢。合併成 1 個 Segment 後，查詢效能提升 2-3 倍。

### Cold Phase：冷儲存

```json
"cold": {
  "min_age": "30d",
  "actions": {
    "searchable_snapshot": {
      "snapshot_repository": "my-s3-repo"  // 把資料移到 S3
    }
  }
}
```

**Searchable Snapshot** 的好處：
- 資料存在 S3（成本降低 90%）
- 仍然可以搜尋（但會慢很多）
- 本地只保留 metadata

### Delete Phase：刪除

```json
"delete": {
  "min_age": "90d",
  "actions": {
    "delete": {}  // 直接刪除
  }
}
```

## 建立 Index Template 套用 ILM

光有 ILM Policy 還不夠，你需要讓新建立的索引自動套用這個 Policy。

### 步驟 2：建立 Index Template

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],  // 所有以 logs- 開頭的索引
  "template": {
    "settings": {
      "number_of_shards": 5,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",  // 套用 ILM Policy
      "index.lifecycle.rollover_alias": "logs"  // 使用 Alias
    }
  }
}
```

### 步驟 3：建立初始索引與 Alias

```json
PUT logs-000001
{
  "aliases": {
    "logs": {
      "is_write_index": true  // 標記為可寫入的索引
    }
  }
}
```

現在：
- 你的應用程式寫入 `logs`（Alias）
- 實際寫入 `logs-000001`
- 當 Rollover 時，自動建立 `logs-000002`，並把 `logs` Alias 指向它

## Logstash 設定

修改 Logstash 的 output：

```ruby
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "logs"  # 寫入 Alias，不是直接寫入索引
  }
}
```

## 查看 ILM 狀態

### API 查詢

```bash
# 查看索引的 ILM 狀態
GET logs-000001/_ilm/explain
```

回傳：

```json
{
  "indices": {
    "logs-000001": {
      "index": "logs-000001",
      "managed": true,
      "policy": "logs-policy",
      "phase": "hot",
      "age": "12h",
      "phase_time_millis": 1682990400000,
      "action": "rollover",
      "step": "check-rollover-ready"
    }
  }
}
```

### Kibana 介面

**Stack Management → Index Management**

你會看到每個索引的：
- **Phase**：Hot / Warm / Cold / Delete
- **Age**：存在多久
- **Size**：大小

## 實戰案例：從 5TB 降到 500GB

我們公司的 Elasticsearch 叢集原本有 5TB 的日誌，每個月的 AWS 帳單是 \$1500。

### 問題分析

```bash
GET _cat/indices?v&s=store.size:desc
```

發現：
- 80% 的儲存空間是 **30 天前的日誌**
- 90% 的查詢只查詢 **過去 7 天的日誌**

### 解決方案

1. **實作 ILM**：
   - Hot Phase：1 天
   - Warm Phase：7 天
   - Cold Phase：30 天
   - Delete Phase：90 天

2. **啟用 Searchable Snapshot**：
   - Cold Phase 的資料移到 S3
   - 本地只保留 Hot + Warm

3. **調整 Shard 數量**：
   - Hot Phase：5 個 Shard
   - Warm Phase：1 個 Shard

### 結果

- **儲存空間**：從 5TB 降到 500GB（降低 90%）
- **AWS 帳單**：從 \$1500/月 降到 \$300/月（降低 80%）
- **查詢效能**：Hot Phase 的查詢速度提升 50%（因為資料更集中）

## Hot-Warm-Cold 架構的硬體配置

### Hot Nodes（熱節點）

```yaml
node.roles: [ data_hot, ingest ]
```

- **硬體**：SSD、高 CPU、大記憶體
- **數量**：3-5 台（視流量而定）
- **用途**：接收寫入、頻繁查詢

### Warm Nodes（溫節點）

```yaml
node.roles: [ data_warm ]
```

- **硬體**：HDD、中等 CPU、中等記憶體
- **數量**：2-3 台
- **用途**：唯讀、偶爾查詢

### Cold Nodes（冷節點）

```yaml
node.roles: [ data_cold ]
```

- **硬體**：HDD、低 CPU、小記憶體
- **數量**：1-2 台（或直接用 S3）
- **用途**：很少查詢、合規留存

## 常見問題

### Q1：我的日誌還在 Hot Phase，為什麼沒有 Rollover？

檢查：
1. 索引是否有 `is_write_index: true` 的 Alias？
2. ILM Policy 是否正確套用？
3. `max_primary_shard_size` 或 `max_age` 是否已達到？

```bash
GET logs-000001/_ilm/explain
```

### Q2：Rollover 後，舊索引的 Alias 會消失嗎？

不會。Rollover 只會把 `is_write_index: true` 移到新索引，舊索引仍保留 Alias（但變成唯讀）。

### Q3：可以手動觸發 Rollover 嗎？

可以：

```bash
POST logs/_rollover
```

### Q4：Cold Phase 一定要用 Searchable Snapshot 嗎？

不一定。如果你沒有 S3，可以只用 `readonly` 和 `shrink`，資料還是留在 Elasticsearch。

### Q5：如何測試 ILM？

修改 `min_age` 為較短的時間（如 1m、5m），然後觀察索引的轉換。

```json
"warm": {
  "min_age": "1m"  // 測試用，生產環境改回 1d
}
```

## ILM 的最佳實踐

### 1. 不同的服務用不同的 Policy

```
logs-nginx-*    → nginx-logs-policy (保留 30 天)
logs-app-*      → app-logs-policy (保留 90 天)
logs-audit-*    → audit-logs-policy (保留 365 天)
```

### 2. 監控 ILM 的執行狀態

設定 Elasticsearch 的告警：

```
當索引卡在某個 Phase 超過 1 小時，發送告警
```

### 3. 定期檢查儲存空間

```bash
GET _cat/allocation?v
```

確保沒有任何節點的磁碟使用率超過 85%（Elasticsearch 會自動停止寫入）。

### 4. 使用 Data Tiers

Elasticsearch 7.10+ 支援 Data Tiers，可以自動把索引移到正確的節點。

```json
"hot": {
  "actions": {
    "set_priority": {
      "priority": 100  // Hot Phase 優先權最高
    }
  }
}
```

---

**ILM 不只是省錢，更是讓你的 Elasticsearch 長期穩定運行的關鍵。**

不要等到硬碟滿了才想起來，現在就設定 ILM！
