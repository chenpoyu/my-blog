---
layout: post
title: "集中式日誌架構設計：支撐百萬級 QPS"
date: 2023-06-09 09:00:00 +0800
categories: [Observability, Logging, Architecture]
tags: [ELK Stack, Architecture Design, High Availability, Scalability]
---

當你的系統從 10 個服務成長到 100 個，日誌量從每天 10GB 成長到 1TB，ELK Stack 的架構也需要進化。

今天我們來談談**生產級的集中式日誌架構**。

> **前置知識**：我們在《Week 14: ELK Stack 入門》介紹了基礎的 Docker Compose 配置。這裏展示的是**生產級的進階架構**，包含高可用性和水平擴展。

## 小型架構（< 10 個服務）

### 架構圖

```
應用程式 → Logstash → Elasticsearch(單節點) → Kibana
```

### 特性

- **優點**：簡單、快速部署
- **缺點**：無高可用性、效能有限
- **適用場景**：開發環境、小團隊

### 配置建議

| 元件 | 配置 |
|------|------|
| Elasticsearch | 1 台 (16GB RAM, 4 CPU, 500GB SSD) |
| Logstash | 1 台 (8GB RAM, 2 CPU) |
| Kibana | 與 Elasticsearch 共用 |

## 中型架構（10-50 個服務）

### 架構圖

```
應用程式 → Filebeat → Kafka → Logstash → Elasticsearch(3 節點) → Kibana
```

### 為什麼要加入 Kafka？

**問題**：Logstash 直接連 Elasticsearch，如果 Elasticsearch 慢或掛掉，日誌會丟失。

**解決**：Kafka 作為緩衝層。

- **解耦**：應用程式不直接依賴 Elasticsearch
- **峰值平滑**：Kafka 可以暫存日誌，避免 Elasticsearch 過載
- **重放能力**：可以重新處理歷史日誌

### 為什麼用 Filebeat 而非 Logstash？

| 特性 | Filebeat | Logstash |
|------|----------|----------|
| 記憶體佔用 | 10-50MB | 500MB-1GB |
| CPU 使用 | 低 | 高 |
| 功能 | 簡單（只負責收集） | 複雜（解析、轉換） |

**結論**：在應用程式端用 Filebeat（輕量），在 Kafka 後用 Logstash（強大）。

### Elasticsearch 叢集配置

```yaml
# elasticsearch.yml

# Node 1 (Master + Data Hot)
node.name: es-node-1
node.roles: [ master, data_hot, ingest ]

# Node 2 (Master + Data Hot)
node.name: es-node-2
node.roles: [ master, data_hot, ingest ]

# Node 3 (Master + Data Hot)
node.name: es-node-3
node.roles: [ master, data_hot, ingest ]

# 叢集設定
cluster.name: logs-cluster
discovery.seed_hosts: ["es-node-1", "es-node-2", "es-node-3"]
cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]

# 記憶體鎖定
bootstrap.memory_lock: true

# 網路設定
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
```

### 配置建議

| 元件 | 數量 | 配置 |
|------|------|------|
| Elasticsearch | 3 台 | 32GB RAM, 8 CPU, 1TB SSD (Hot Nodes) |
| Kafka | 3 台 | 16GB RAM, 4 CPU, 500GB HDD |
| Logstash | 2 台 | 16GB RAM, 4 CPU |
| Kibana | 1 台 | 8GB RAM, 2 CPU |

## 大型架構（50+ 個服務，億級日誌）

### 架構圖

```
應用程式 → Filebeat → Kafka → Logstash Pipeline → Elasticsearch (Hot-Warm-Cold) → Kibana
                                                  ↓
                                            S3 (Cold Storage)
```

### 關鍵設計

#### 1. Hot-Warm-Cold 架構

```
Hot Nodes (0-1天):   SSD, 高效能, 寫入 + 查詢
Warm Nodes (1-30天): HDD, 中效能, 唯讀查詢
Cold Nodes (30-90天): S3, 低成本, 很少查詢
Delete (90天後):      刪除
```

#### 2. Logstash Pipeline

用多個 Logstash 實例分擔負載。

```ruby
# logstash-01.yml (處理應用程式日誌)
input {
  kafka {
    bootstrap_servers => "kafka-1:9092,kafka-2:9092,kafka-3:9092"
    topics => ["app-logs"]
    group_id => "logstash-app"
  }
}

filter {
  # 應用程式日誌的解析邏輯
}

output {
  elasticsearch {
    hosts => ["es-1:9200", "es-2:9200", "es-3:9200"]
    index => "logs-app-%{+YYYY.MM.dd}"
  }
}
```

```ruby
# logstash-02.yml (處理 Nginx 日誌)
input {
  kafka {
    bootstrap_servers => "kafka-1:9092,kafka-2:9092,kafka-3:9092"
    topics => ["nginx-logs"]
    group_id => "logstash-nginx"
  }
}

filter {
  # Nginx 日誌的解析邏輯
}

output {
  elasticsearch {
    hosts => ["es-1:9200", "es-2:9200", "es-3:9200"]
    index => "logs-nginx-%{+YYYY.MM.dd}"
  }
}
```

#### 3. Elasticsearch 叢集配置

```yaml
# Master Nodes (3 台)
node.roles: [ master ]
# 專門負責叢集管理，不儲存資料

# Hot Nodes (5 台)
node.roles: [ data_hot, ingest ]
# SSD, 接收寫入 + 頻繁查詢

# Warm Nodes (3 台)
node.roles: [ data_warm ]
# HDD, 唯讀查詢

# Cold Nodes (2 台)
node.roles: [ data_cold ]
# HDD, 很少查詢（或直接用 S3）

# Coordinating Nodes (2 台)
node.roles: []
# 專門負責路由查詢請求，不儲存資料
```

### 配置建議

| 元件 | 數量 | 配置 |
|------|------|------|
| Elasticsearch Master | 3 台 | 8GB RAM, 2 CPU, 100GB SSD |
| Elasticsearch Hot | 5 台 | 64GB RAM, 16 CPU, 2TB NVMe SSD |
| Elasticsearch Warm | 3 台 | 32GB RAM, 8 CPU, 4TB HDD |
| Elasticsearch Cold | 2 台 | 16GB RAM, 4 CPU, 8TB HDD |
| Elasticsearch Coordinating | 2 台 | 16GB RAM, 4 CPU |
| Kafka | 5 台 | 32GB RAM, 8 CPU, 1TB HDD |
| Logstash | 4-6 台 | 32GB RAM, 8 CPU |
| Kibana | 2 台 (Load Balanced) | 16GB RAM, 4 CPU |

## 高可用性設計

### 1. Elasticsearch

```yaml
# 每個索引至少 1 個 Replica
index.number_of_replicas: 1

# 最少 2 個 Master 節點存活才能選舉
discovery.zen.minimum_master_nodes: 2
```

### 2. Kafka

```properties
# Kafka 的 Replication Factor
default.replication.factor=3
min.insync.replicas=2
```

### 3. Logstash

部署多個 Logstash 實例，任一台掛了不影響整體。

### 4. Kibana

用 Nginx 或 HAProxy 做 Load Balancing：

```nginx
upstream kibana {
    server kibana-1:5601;
    server kibana-2:5601;
}

server {
    listen 80;
    location / {
        proxy_pass http://kibana;
    }
}
```

## 效能優化

### 1. Kafka Partitions

```
Topic: app-logs
Partitions: 10 (可以同時有 10 個 Logstash Consumer)
```

### 2. Logstash Workers

```yaml
# logstash.yml
pipeline.workers: 16  # 增加並行處理
pipeline.batch.size: 2000  # 增加 Batch Size
```

### 3. Elasticsearch Index Settings

```json
{
  "index": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "refresh_interval": "30s",
    "translog.durability": "async"
  }
}
```

## 監控與告警

### 關鍵指標

| 指標 | 說明 | 告警閾值 |
|------|------|----------|
| **Kafka Lag** | Logstash 消費延遲 | > 100,000 |
| **Logstash CPU** | CPU 使用率 | > 80% |
| **ES Heap Used** | JVM 記憶體使用率 | > 85% |
| **ES Disk Used** | 磁碟使用率 | > 85% |
| **ES Indexing Rate** | 寫入速度 | 突然下降 50% |
| **ES Search Latency** | 查詢延遲 | P95 > 5s |

### Prometheus + Grafana

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'elasticsearch'
    static_configs:
      - targets: ['es-1:9200', 'es-2:9200', 'es-3:9200']
  
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092']
  
  - job_name: 'logstash'
    static_configs:
      - targets: ['logstash-1:9600', 'logstash-2:9600']
```

## 災難復原

### 1. Elasticsearch Snapshot

每天自動備份到 S3：

```json
PUT _snapshot/my-backup
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "region": "us-west-2"
  }
}

PUT _snapshot/my-backup/snapshot-2023-06-09?wait_for_completion=true
{
  "indices": "logs-*",
  "ignore_unavailable": true
}
```

### 2. Kafka Backup

Kafka 的資料會自動複製到多個 Broker，通常不需要額外備份。

### 3. 跨區域部署

在兩個不同的 AWS Region 部署 ELK Stack，用 Kafka MirrorMaker 同步資料。

## 成本優化

### 1. 使用 S3 作為 Cold Storage

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

**成本降低**：90%（從 SSD 到 S3）

### 2. 調整 Replica 數量

```
Hot Nodes: 1 Replica (高可用性)
Warm Nodes: 0 Replica (節省空間)
```

### 3. 使用 Spot Instances

Warm/Cold Nodes 可以用 AWS Spot Instances（成本降低 70%）。

---

**好的架構不是一次到位，而是隨著業務成長逐步演進。**

從單節點開始，當遇到瓶頸時再擴展，不要過度設計。
