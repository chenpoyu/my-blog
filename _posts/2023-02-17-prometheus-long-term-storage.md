---
layout: post
title: "Prometheus 長期儲存：數據不該只活 15 天"
date: 2023-02-17 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Prometheus, Thanos, Storage, Time Series Database]
---

前面幾週我們學會了收集、查詢、告警。但有個殘酷的現實：**Prometheus 預設只保留 15 天的數據。**

當老闆問你：「去年雙 11 的流量高峰是多少？」你只能兩手一攤：「數據已經被刪了。」

今天我們要解決這個問題：**如何在不拖垮 Prometheus 的情況下，保留長期歷史數據？**

> 本文涵蓋: **Prometheus 儲存機制、Thanos、Remote Write**

## Prometheus 的儲存困境

Prometheus 使用自己的時序資料庫（TSDB），非常快，但有個問題：**所有數據都在本地磁碟。**

預設設定：
```yaml
storage:
  tsdb:
    retention.time: 15d  # 保留 15 天
    retention.size: 50GB # 或磁碟滿 50GB
```

超過限制，舊數據就被刪了。

### 為什麼不直接延長保留時間？

你可能會想：「那我設定 `retention.time: 365d` 不就好了？」

問題是：

1. **磁碟成本**：一年的數據可能要幾百 GB 甚至幾 TB
2. **查詢變慢**：查詢範圍越大，掃描的數據越多
3. **單點故障**：所有數據在一台機器上，掛了就全沒了

所以我們需要「分層儲存」：
- **熱數據**（近期）：Prometheus 本地，快速查詢
- **冷數據**（歷史）：物件儲存（S3、GCS），便宜但慢

## 解決方案 1：Thanos（推薦）

Thanos 是 CNCF 的開源專案，專門解決 Prometheus 的擴展性問題。

### Thanos 的架構

```
┌─────────────┐
│ Prometheus  │───┐
└─────────────┘   │
                  │ 上傳 blocks
┌─────────────┐   │   ┌──────────┐
│ Prometheus  │───┼──>│  Thanos  │──> S3/GCS/MinIO
└─────────────┘   │   │  Sidecar │
                  │   └──────────┘
┌─────────────┐   │
│ Prometheus  │───┘
└─────────────┘

                  ┌──────────┐
                  │  Thanos  │
使用者 ────────────>│  Query   │───> 統一查詢介面
                  └──────────┘
```

### 安裝 Thanos Sidecar

修改 Prometheus 啟動參數：

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v $(pwd)/data:/prometheus \
  prom/prometheus:v2.40.0 \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.retention.time=7d \
  --storage.tsdb.min-block-duration=2h \
  --storage.tsdb.max-block-duration=2h \
  --web.enable-lifecycle
```

重點：
- `retention.time=7d`：本地只保留 7 天（降低壓力）
- `min-block-duration=2h`：每 2 小時產生一個 block（方便上傳）

啟動 Thanos Sidecar：

```bash
docker run -d \
  --name thanos-sidecar \
  -p 10901:10901 \
  -v $(pwd)/data:/prometheus \
  -v $(pwd)/bucket.yml:/bucket.yml \
  thanosio/thanos:v0.30.0 \
  sidecar \
  --tsdb.path=/prometheus \
  --prometheus.url=http://prometheus:9090 \
  --objstore.config-file=/bucket.yml \
  --grpc-address=0.0.0.0:10901
```

### 設定物件儲存

建立 `bucket.yml`（以 S3 為例）：

```yaml
type: S3
config:
  bucket: "my-prometheus-data"
  endpoint: "s3.amazonaws.com"
  access_key: "YOUR_ACCESS_KEY"
  secret_key: "YOUR_SECRET_KEY"
```

如果預算有限，可以用 MinIO（自架 S3 相容儲存）：

```bash
docker run -d \
  --name minio \
  -p 9000:9000 \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=password" \
  -v $(pwd)/minio-data:/data \
  minio/minio server /data
```

### 啟動 Thanos Query

```bash
docker run -d \
  --name thanos-query \
  -p 10902:10902 \
  thanosio/thanos:v0.30.0 \
  query \
  --http-address=0.0.0.0:10902 \
  --store=thanos-sidecar:10901
```

現在訪問 `http://localhost:10902`，你會看到一個類似 Prometheus 的介面，但可以查詢長期歷史數據！

### 查詢示範

在 Thanos Query 中輸入：

```promql
rate(http_requests_total[5m])
```

選擇時間範圍：**過去 3 個月**

Thanos 會自動：
1. 近期數據從 Prometheus 查詢（快）
2. 歷史數據從 S3 查詢（慢但完整）
3. 合併結果返回

## 解決方案 2：Remote Write

如果 Thanos 對你來說太複雜，還有更簡單的方法：**Remote Write**。

Prometheus 可以把數據即時推送到遠端儲存。

### 設定 Remote Write

修改 `prometheus.yml`：

```yaml
remote_write:
  - url: "https://your-remote-storage.com/api/v1/write"
    basic_auth:
      username: "user"
      password: "password"
```

支援的遠端儲存：
- **VictoriaMetrics**：開源，性能優異
- **Cortex**：CNCF 專案，多租戶支援
- **Grafana Cloud**：託管服務，開箱即用
- **AWS Timestream**、**Google Cloud Monitoring**

### VictoriaMetrics 範例

啟動 VictoriaMetrics：

```bash
docker run -d \
  --name victoria-metrics \
  -p 8428:8428 \
  -v $(pwd)/victoria-data:/victoria-metrics-data \
  victoriametrics/victoria-metrics:latest
```

修改 `prometheus.yml`：

```yaml
remote_write:
  - url: "http://victoria-metrics:8428/api/v1/write"
```

重啟 Prometheus，數據就會自動寫入 VictoriaMetrics。

你可以在 Grafana 中加入 VictoriaMetrics 作為數據源，查詢長期數據。

## 成本優化：不是所有數據都值得保留

長期儲存很花錢，我們要聰明一點。

### 策略 1：降採樣（Downsampling）

保留原始數據：
- **近 7 天**：完整數據（15 秒一個點）
- **7-30 天**：1 分鐘聚合
- **30-90 天**：5 分鐘聚合
- **90 天以上**：1 小時聚合

Thanos Compactor 會自動做這件事：

```bash
docker run -d \
  --name thanos-compactor \
  -v $(pwd)/bucket.yml:/bucket.yml \
  thanosio/thanos:v0.30.0 \
  compact \
  --objstore.config-file=/bucket.yml \
  --data-dir=/tmp/thanos-compact \
  --retention.resolution-raw=7d \
  --retention.resolution-5m=30d \
  --retention.resolution-1h=365d
```

### 策略 2：選擇性保留

不是所有指標都要保留一年。

用 `write_relabel_configs` 過濾：

```yaml
remote_write:
  - url: "https://long-term-storage.com/write"
    write_relabel_configs:
      # 只保留業務關鍵指標
      - source_labels: [__name__]
        regex: 'http_requests_total|http_request_duration_seconds.*|order_.*|payment_.*'
        action: keep
      
      # 不保留 debug 指標
      - source_labels: [__name__]
        regex: '.*_debug_.*'
        action: drop
```

這樣可以大幅降低儲存成本。

### 策略 3：分層查詢

在 Grafana Dashboard 中：

- **預設時間範圍**：過去 24 小時（查 Prometheus，快）
- **歷史分析**：過去 30 天（查 Thanos，慢但可接受）
- **年度報告**：過去 12 個月（提前準備，不要臨時查）

## 實戰案例：合規要求的長期儲存

我曾經遇到一個金融客戶，監管單位要求保留 7 年的交易監控數據。

我們的方案：

### 第一層：Prometheus（7 天）
- 所有指標
- 15 秒精度
- 本地 SSD，極快

### 第二層：Thanos + S3（30 天）
- 所有指標
- 15 秒精度
- 用於事故調查

### 第三層：Thanos + S3 Glacier（1 年）
- 關鍵業務指標
- 5 分鐘精度
- 用於季度審查

### 第四層：S3 Glacier Deep Archive（7 年）
- 僅交易相關指標
- 1 小時精度
- 用於合規稽核（查詢很慢，但便宜）

**成本**：
- Prometheus 本地：$500/月（SSD）
- S3 Standard：$1000/月
- S3 Glacier：$200/月
- S3 Glacier Deep Archive：$50/月

比起全部用 Prometheus 本地儲存（估計要 $20000/月），省了 90% 以上。

## 高可用性：Prometheus 的備援

單機 Prometheus 掛了怎麼辦？

### 方案 1：多個 Prometheus + Thanos Query

```
┌─────────────┐
│ Prometheus 1│───┐
└─────────────┘   │
                  ├──> Thanos Query ───> 使用者
┌─────────────┐   │
│ Prometheus 2│───┘
└─────────────┘

兩個 Prometheus 抓取相同目標（deduplication）
```

Thanos Query 會自動去重（deduplication），使用者看到的是統一結果。

### 方案 2：Prometheus Operator (K8s)

如果你用 Kubernetes：

```bash
helm install prometheus-operator prometheus-community/kube-prometheus-stack
```

這會自動設定：
- Prometheus 的 StatefulSet（持久化儲存）
- Alertmanager 的高可用（3 個副本）
- Grafana 的持久化設定

---

**數據是資產，但無限制地保留數據是浪費。**

聰明的工程師會根據數據的價值，選擇合適的保留策略。

當老闆問「去年這時候的流量是多少」時，你能立刻拿出數據，這就是專業。
