---
layout: post
title: "Prometheus 入門：從安裝到第一個指標"
date: 2023-01-13 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Prometheus, Metrics, 時序資料庫, 監控]
---

上週我們聊了為什麼要學觀測性技術。從今天開始，我們正式進入實戰：**Prometheus**。

這個名字來自希臘神話中的「普羅米修斯」，他為人類帶來了火種。而 Prometheus 監控系統，則為我們帶來了「系統健康狀態的可見性」。

> 本文使用版本: **Prometheus 2.40** (2023 年初的穩定版本)

## 為什麼是 Prometheus？

在雲原生（Cloud Native）時代，Prometheus 已經成為事實上的監控標準。原因很簡單：

1. **Pull 模式**：Prometheus 主動去抓取（scrape）目標的指標，而不是等目標推送（push）
2. **強大的查詢語法**：PromQL 可以做複雜的數據聚合和計算
3. **時序資料庫**：專門為時間序列數據優化，查詢速度快
4. **整合性強**：K8s、Docker、各種 Exporter 都有原生支持
5. **開源免費**：CNCF 畢業專案，社群活躍

但我要強調：**工具只是手段，重點是建立監控的思維。**

## Prometheus 的核心概念

在動手之前，先理解幾個關鍵概念：

### 1. Metrics（指標）

Prometheus 的指標有四種類型：

**Counter（計數器）**：只能增加的數值
```
http_requests_total{method="GET", endpoint="/api/users"} 1234
```

**Gauge（儀表盤）**：可增可減的數值
```
memory_usage_bytes 8589934592
```

**Histogram（直方圖）**：統計數據分佈
```
http_request_duration_seconds_bucket{le="0.1"} 100
http_request_duration_seconds_bucket{le="0.5"} 250
```

**Summary（摘要）**：類似 Histogram，但在客戶端計算分位數

### 2. Labels（標籤）

這是 Prometheus 的精髓所在：
```
http_requests_total{method="GET", endpoint="/api/users", status="200"}
http_requests_total{method="POST", endpoint="/api/orders", status="500"}
```

透過 Labels，你可以對同一個指標做多維度的分析。

**重要提醒**：不要濫用 Labels！如果某個 Label 有無限多種可能的值（例如使用者 ID），會導致時序資料庫爆炸。這是新手最常犯的錯誤。

### 3. Jobs & Instances

- **Job**：一組相同目的的 instances（例如：api-server）
- **Instance**：單一個監控目標（例如：api-server-1, api-server-2）

## 安裝 Prometheus

廢話不多說，直接動手。我們用 Docker 來快速體驗：

```bash
# 建立設定檔目錄
mkdir -p ~/prometheus-lab
cd ~/prometheus-lab

# 建立 prometheus.yml
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s      # 每 15 秒抓取一次指標
  evaluation_interval: 15s  # 每 15 秒評估一次告警規則

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF

# 啟動 Prometheus
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus:v2.40.0
```

開啟瀏覽器，前往 `http://localhost:9090`，你應該會看到 Prometheus 的 Web UI。

## 第一個查詢：監控 Prometheus 自己

Prometheus 很聰明，它會監控自己的運行狀態。在 Web UI 的 Graph 頁面，輸入：

```promql
up
```

你會看到：
```
up{instance="localhost:9090", job="prometheus"} 1
```

這表示 Prometheus 正常運行。`up` 是一個特殊指標，值為 1 表示目標健康，0 表示目標死掉。

再試試這個：
```promql
prometheus_http_requests_total
```

這會顯示 Prometheus 處理的 HTTP 請求總數。你可以看到很多 Labels，例如：
```
prometheus_http_requests_total{code="200", handler="/metrics"}
prometheus_http_requests_total{code="200", handler="/graph"}
```

## 來點實用的：監控 Node Exporter

光監控 Prometheus 自己沒什麼意思，我們來監控系統資源。

```bash
# 啟動 Node Exporter（用來收集系統指標）
docker run -d \
  --name node-exporter \
  -p 9100:9100 \
  prom/node-exporter:v1.5.0
```

修改 `prometheus.yml`，加入新的 scrape target：

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'node'
    static_configs:
      - targets: ['host.docker.internal:9100']  # Mac/Windows Docker
```

重啟 Prometheus：
```bash
docker restart prometheus
```

等待 15 秒（scrape_interval），然後查詢：

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

這會顯示你的系統可用記憶體（單位：GB）。

再試試 CPU 使用率：
```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

是不是開始有感覺了？

## 關鍵心法：從使用者角度思考

很多人學 Prometheus，就是照著文件把所有指標都收集起來，結果：

1. 儲存空間爆炸
2. 查詢速度變慢
3. 根本不知道要看什麼

我的建議是：**從使用者的痛點出發。**

假設你負責一個 API 服務，你應該監控：

- **可用性**：服務是否在線？（`up` 指標）
- **流量**：每秒請求數（QPS）
- **延遲**：P50、P95、P99 回應時間
- **錯誤率**：5xx 錯誤比例

這就是 Google SRE 提出的「四個黃金訊號」（Four Golden Signals）。

---

**監控不是收集所有數據，而是收集能幫助你做決策的數據。**

有了指標，你才能說：「我們的 API 回應時間比上週慢了 30%」，而不是憑感覺說「好像變慢了」。

這就是量化的力量。
