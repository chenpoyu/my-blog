---
layout: post
title: "Grafana 視覺化：讓數據說人話"
date: 2023-01-27 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Grafana, Dashboard, Visualization, Prometheus]
---

前兩週我們學會了收集指標，但如果你一直盯著 Prometheus 的查詢介面看數字，不用三天就會瘋掉。

人類是視覺動物，我們需要**圖表、儀表板、顏色**來快速理解系統狀態。這就是 Grafana 存在的意義。

> 本文使用版本: **Grafana 9.3** (2023 年初穩定版)

## 為什麼需要 Grafana？

讓我先問你幾個問題：

1. 你能在 5 秒內判斷系統是否健康嗎？
2. 當 API 變慢時，你能立刻看出是哪個服務拖慢的嗎？
3. 凌晨三點被叫醒，你能快速找到根因嗎？

如果答案是「不能」，那你需要一個好的 Dashboard。

**好的視覺化設計，可以讓你在喝咖啡的 10 秒內,就發現系統異常。**

## 安裝 Grafana

用 Docker 快速啟動：

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana:9.3.0
```

開啟瀏覽器，前往 `http://localhost:3000`

- 預設帳號：`admin`
- 預設密碼：`admin`（第一次登入會要求改密碼）

## 連接 Prometheus 資料源

進入後，點選左側齒輪圖示 → **Data Sources** → **Add data source** → 選擇 **Prometheus**

設定：
- **URL**: `http://host.docker.internal:9090` (Mac/Windows Docker)
- 或如果在同一網路：`http://prometheus:9090`

點選 **Save & Test**，如果看到綠色勾勾，就成功了。

## 建立第一個 Dashboard

> **關於 RED Method**：這裡使用的指標基於 RED Method（Rate、Errors、Duration）。詳細說明請參考《Week 8: 監控方法論：RED vs USE》文章。

點選左側 **+** 號 → **Dashboard** → **Add a new panel**

### Panel 1: API 請求數（QPS）

在 **Query** 欄位輸入：

```promql
sum(rate(http_requests_total[5m])) by (endpoint)
```

這會計算過去 5 分鐘內，每個 endpoint 的每秒請求數。

**視覺化設定：**
- Panel type: **Time series**（折線圖）
- Title: `API Requests per Second (QPS)`
- Y-axis label: `req/s`

點選右上角 **Apply** 儲存。

### Panel 2: API 回應時間（P95）

新增另一個 Panel，輸入：

```promql
histogram_quantile(0.95, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)
```

**視覺化設定：**
- Panel type: **Time series**
- Title: `API Response Time (P95)`
- Y-axis label: `seconds`
- Unit: `seconds (s)`

### Panel 3: 錯誤率

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100
```

**視覺化設定：**
- Panel type: **Stat**（大數字顯示）
- Title: `Error Rate`
- Unit: `percent (0-100)`
- Thresholds:
  - Green: < 1%
  - Yellow: 1-5%
  - Red: > 5%

現在你有了一個基本的 Dashboard！點選右上角 **Save dashboard**，命名為 `API Monitoring`。

## 進階技巧：RED Method

Google SRE 提出的 **RED Method** 是監控最佳實踐：

- **Rate（速率）**：每秒請求數
- **Errors（錯誤）**：錯誤比例
- **Duration（延遲）**：回應時間

我們剛剛已經做了這三個，但還可以更進階。

### 顯示多個百分位數

```promql
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))  # P50
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))  # P95
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))  # P99
```

在同一個 Panel 中加入三條線，你可以看到「大部分使用者」(P50) 和「最倒楣的那 1%」(P99) 的體驗差異。

**這很重要：平均值會騙人，但百分位數不會。**

假設你有 1000 個請求：
- 999 個在 100ms 內完成
- 1 個卡了 10 秒

平均值是 109ms，看起來不錯。但 P99 是 10 秒，使用者會氣炸。

## 實戰案例：JVM 監控 Dashboard

如果你用 Java，記憶體洩漏是常見問題。讓我們做個 JVM Dashboard。

### Panel 1: Heap Memory 使用率

```promql
(jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"}) * 100
```

- Panel type: **Gauge**（儀表盤）
- Thresholds: Green < 70%, Yellow 70-85%, Red > 85%

### Panel 2: GC 暫停時間

```promql
rate(jvm_gc_pause_seconds_sum[5m])
```

如果這個數字持續上升，代表 GC 壓力大，可能記憶體不足。

### Panel 3: Thread Count

```promql
jvm_threads_current
```

如果執行緒數量暴增，可能有執行緒洩漏（thread leak）。

## Dashboard 設計哲學

我見過很多「看起來很炫」但「完全不實用」的 Dashboard。通常有這些問題：

❌ **資訊過載**：塞了 50 個 Panel，眼花撩亂
❌ **缺乏優先順序**：所有圖表一樣大
❌ **沒有上下文**：只有數字,不知道正常範圍
❌ **顏色濫用**：紅配綠,看了頭痛

我的建議：

### 1. 上方放最重要的指標（Above the fold）

使用者打開 Dashboard 的第一眼，應該看到：

> 「系統現在健康嗎？」

用大數字（Stat Panel）顯示：
- ✅ 服務可用性（99.9%）
- ⚠️ 錯誤率（0.5%）
- ✅ P95 延遲（150ms）

### 2. 用顏色傳達狀態

- **綠色**：一切正常
- **黃色**：需要注意
- **紅色**：馬上處理

不要用藍色、紫色、粉紅色搞花俏，這不是藝術展。

### 3. 設定合理的時間範圍

預設顯示「過去 1 小時」，並提供快速切換：
- Last 5 minutes（出問題時看細節）
- Last 1 hour（日常監控）
- Last 24 hours（趨勢分析）
- Last 7 days（容量規劃）

### 4. 加入 Annotations（註記）

當你部署新版本時，在 Dashboard 上標記一條線，這樣就能看出：

> 「咦，部署後 P95 延遲從 200ms 漲到 500ms，是不是這次改動有問題？」

Grafana 支援從 Git、CI/CD 自動加入 Annotations。

## 實用的 Panel 類型選擇

| 目的 | Panel Type | 範例 |
|------|-----------|------|
| 顯示趨勢 | Time series（折線圖） | CPU 使用率、請求數 |
| 顯示當下狀態 | Stat（大數字） | 錯誤率、可用性 |
| 顯示百分比 | Gauge（儀表盤） | 記憶體使用率 |
| 顯示分佈 | Bar chart（長條圖） | 各 endpoint 的流量佔比 |
| 顯示地理位置 | Geomap | 使用者來源國家 |

## 團隊協作：Dashboard as Code

當你的 Dashboard 設計得很好，其他團隊也想用，怎麼辦？

Grafana 支援 **JSON 匯出/匯入**。

點選 Dashboard 右上角齒輪圖示 → **JSON Model**，你會看到一整個 JSON 設定。

把它存到 Git，寫進 CI/CD，就能實現「基礎設施即代碼」。

更進階的做法是使用 **Grafana Provisioning**，在 `dashboards.yml` 中定義：

```yaml
apiVersion: 1

providers:
  - name: 'default'
    folder: 'Monitoring'
    type: file
    options:
      path: /etc/grafana/dashboards
```

這樣一來，Dashboard 就能版本控制、程式碼審查、自動部署。

---

**Dashboard 不是用來炫技的,而是用來快速發現問題的。**

一個好的 Dashboard，應該讓一個剛進公司的新人，在 30 秒內就能判斷系統是否健康。

如果做不到,那就是設計失敗。
