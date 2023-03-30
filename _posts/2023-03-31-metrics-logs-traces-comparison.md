---
layout: post
title: "Metrics vs Logs vs Traces：三大支柱的本質差異"
date: 2023-03-31 09:00:00 +0800
categories: [Observability, Logging]
tags: [Observability, Metrics, Logs, Traces, ELK Stack]
---

我們終於完成 Prometheus 學習！從今天開始要進入 Q2：**日誌管理（Logging）**。

但在開始之前，我要先回答一個根本問題：**為什麼有了 Metrics 還需要 Logs？它們到底有什麼不同？**

這不是學術問題，而是實戰關鍵。我見過太多團隊「只用 Logs」或「只用 Metrics」，結果出問題時兩眼一抹黑。

## 觀測性的三大支柱

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Metrics  │  │   Logs   │  │  Traces  │
└──────────┘  └──────────┘  └──────────┘
     ↓             ↓             ↓
  量化數據      詳細記錄      請求路徑
```

這不是「三選一」，而是「三者互補」。

## Metrics（指標）：告訴你「發生了什麼」

### 本質

**Metrics 是時間序列的數值。**

```
http_requests_total{method="GET", status="200"} 1234  @1678886400
http_requests_total{method="GET", status="200"} 1240  @1678886415
http_requests_total{method="GET", status="200"} 1245  @1678886430
```

### 特點

✅ **聚合性強**：可以輕鬆計算平均值、P95、總和  
✅ **儲存高效**：只存數字，不存文字  
✅ **查詢極快**：毫秒級回應  
✅ **趨勢分析**：適合看長期趨勢

❌ **缺乏細節**：只知道「有 5 個錯誤」，不知道「錯誤訊息是什麼」  
❌ **高基數問題**：不能用 user_id 當 label

### 適用場景

- 即時告警（錯誤率、延遲）
- Dashboard 視覺化
- 容量規劃
- SLO 追蹤

### 範例問題

❓ 「過去一小時 API 的錯誤率是多少？」  
✅ Metrics 可以回答

❓ 「哪個使用者遇到錯誤？」  
❌ Metrics 無法回答

## Logs（日誌）：告訴你「為什麼發生」

### 本質

**Logs 是事件的文字記錄。**

```
2023-03-31 10:15:23 ERROR [OrderService] Failed to create order
  - User ID: 12345
  - Order ID: 67890
  - Error: PaymentGatewayTimeoutException: Connection timeout after 30s
  - Stack trace: ...
```

### 特點

✅ **詳細完整**：包含所有上下文資訊  
✅ **可搜尋**：能用關鍵字搜尋  
✅ **高基數友善**：可以包含 user_id、order_id  
✅ **事後調查**：適合問題排查

❌ **儲存昂貴**：文字量大  
❌ **查詢較慢**：需要全文搜尋  
❌ **難以聚合**：不適合做統計

### 適用場景

- 錯誤調查（Exception stack trace）
- 審計日誌（誰在什麼時間做了什麼）
- 除錯（Debug logging）
- 合規要求

### 範例問題

❓ 「哪個使用者在 10:15 遇到付款錯誤？」  
✅ Logs 可以回答

❓ 「過去一週付款成功率的趨勢？」  
❌ Logs 不適合（要全掃一遍）

## Traces（追蹤）：告訴你「如何傳遞」

### 本質

**Traces 是一個請求在分散式系統中的完整路徑。**

```
[User Request]
    ↓ 100ms
[API Gateway]
    ↓ 50ms
[Order Service]
    ├─ 200ms → [Inventory Service]
    └─ 300ms → [Payment Service]
              ↓ 250ms
              [External Payment API]
```

### 特點

✅ **全局視角**：看到請求的完整生命週期  
✅ **跨服務追蹤**：知道時間花在哪個服務  
✅ **效能分析**：找出瓶頸  
✅ **依賴關係**：了解服務間的呼叫鏈

❌ **實作複雜**：需要改程式碼，傳遞 Trace ID  
❌ **儲存昂貴**：每個請求都要存  
❌ **採樣必要**：不可能追蹤所有請求

### 適用場景

- 微服務效能分析
- 找出慢查詢的根因
- 理解服務依賴關係
- 分散式系統除錯

### 範例問題

❓ 「這個請求為什麼花了 3 秒？」  
✅ Traces 可以回答（看到是 Payment Service 花了 2.5 秒）

❓ 「Payment Service 昨天一共被呼叫幾次？」  
❌ Traces 不適合（應該用 Metrics）

## 三者如何協同工作？

### 場景：半夜收到告警

**第一步：Metrics 告訴你「有問題」**

```promql
# 告警：API 錯誤率 > 5%
job:http_requests:error_rate5m > 0.05
```

你知道了：「過去 5 分鐘錯誤率是 8%」

**第二步：Metrics 縮小範圍**

```promql
# 哪個 endpoint 在出錯？
topk(5, sum(rate(http_requests_total{status=~"5.."}[5m])) by (endpoint))
```

你發現：「是 `/api/orders` 這個 endpoint」

**第三步：Logs 找出根因**

```
搜尋：endpoint="/api/orders" AND level=ERROR AND timestamp > 10 minutes ago
```

你看到日誌：

```
2023-03-31 03:15:42 ERROR [OrderService] 
PaymentGatewayTimeoutException: Connection timeout after 30s
  at PaymentService.charge(PaymentService.java:45)
  at OrderService.createOrder(OrderService.java:123)
```

你知道了：「是第三方支付 API 超時」

**第四步：Traces 確認影響範圍**

查詢 Trace ID，看到：

```
[API Gateway] 50ms
  → [Order Service] 30100ms  ← 這裡很慢
      → [Payment Service] 30050ms  ← 瓶頸在這
```

你確認了：「所有經過 Payment Service 的請求都被拖慢」

**第五步：採取行動**

1. 切換到備用金流（降級）
2. 通知第三方金流商（修復）
3. 加入 Circuit Breaker（預防）

**從告警到根因到解決：10 分鐘。**

## 什麼時候用什麼？

| 問題類型 | 用 Metrics | 用 Logs | 用 Traces |
|---------|-----------|---------|-----------|
| 系統現在健康嗎？ | ✅ | ❌ | ❌ |
| 錯誤率趨勢如何？ | ✅ | ❌ | ❌ |
| 這個錯誤的詳細訊息是什麼？ | ❌ | ✅ | ❌ |
| 哪個使用者遇到問題？ | ❌ | ✅ | ✅ |
| 這個請求為什麼慢？ | ❌ | ❌ | ✅ |
| 服務 A 和 服務 B 的依賴關係？ | ❌ | ❌ | ✅ |
| 過去一週的 P95 延遲趨勢？ | ✅ | ❌ | ❌ |
| 誰在什麼時間修改了什麼資料？ | ❌ | ✅ | ❌ |

## 常見的錯誤觀念

### 錯誤 1：「有 Logs 就不需要 Metrics」

這是最大的誤區。

Logs 無法高效地回答「趨勢問題」。假設你想知道「過去 24 小時的錯誤率」，用 Logs 的話：

1. 掃描 24 小時的所有日誌（可能幾 GB）
2. 用正則表達式過濾 ERROR
3. 計算比例

**查詢時間：30 秒**

用 Metrics 的話：

```promql
job:http_requests:error_rate5m[24h]
```

**查詢時間：0.1 秒**

### 錯誤 2：「有 Metrics 就不需要 Logs」

Metrics 無法告訴你「細節」。

假設你看到「錯誤率突然上升」，Metrics 只能告訴你：

> 「過去 5 分鐘有 100 個錯誤」

但你不知道：
- 錯誤訊息是什麼？
- 哪些使用者受影響？
- 是程式碼 Bug 還是外部服務問題？

這時候必須看 Logs。

### 錯誤 3：「Traces 可以取代 Logs」

Traces 只追蹤「請求路徑」，不包含「業務邏輯細節」。

假設你想知道「為什麼這筆訂單被標記為詐騙」，Trace 只能告訴你：

> 「請求經過了 Fraud Detection Service，花了 50ms」

但不會告訴你「因為這個使用者在 1 小時內下了 100 筆訂單」。

這種資訊要從 Logs 找。

## 工具選擇

### Metrics
- **Prometheus**：開源、功能強大
- **VictoriaMetrics**：Prometheus 相容、效能更好
- **Datadog**、**New Relic**：商業方案、功能全面

### Logs
- **ELK Stack**（Elasticsearch + Logstash + Kibana）：開源、功能完整
- **Grafana Loki**：輕量、整合 Grafana
- **Splunk**：商業方案、企業級

### Traces
- **OpenTelemetry + Jaeger**：開源標準
- **Zipkin**：輕量、簡單
- **Datadog APM**、**New Relic**：商業方案

## 理想的觀測性架構

```
┌─────────────────────────────────────────┐
│           Grafana Dashboard             │  ← 統一介面
└────────┬───────────┬───────────┬────────┘
         │           │           │
    ┌────▼────┐ ┌────▼────┐ ┌───▼────┐
    │Prometheus│ │  Loki   │ │ Jaeger │
    │(Metrics) │ │ (Logs)  │ │(Traces)│
    └────┬────┘ └────┬────┘ └───┬────┘
         │           │           │
    ┌────▼───────────▼───────────▼────┐
    │         Your Application         │
    └──────────────────────────────────┘
```

---

**Metrics、Logs、Traces 不是競爭關係，而是互補關係。**

成熟的團隊會三者並用，讓它們各司其職。

當能在 Grafana 看到「錯誤率上升」，點一下就跳到 Kibana 看「錯誤日誌」，再點一下就跳到 Jaeger 看「完整 Trace」，你就達到了「全棧可觀測性」。
