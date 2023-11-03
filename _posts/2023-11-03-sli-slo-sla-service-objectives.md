---
layout: post
title: "SLI、SLO、SLA：服務品質目標"
date: 2023-11-03 09:00:00 +0800
categories: [Observability, SRE]
tags: [SLI, SLO, SLA, SRE, Reliability]
---

「我們的系統很穩定」— 這是一個模糊的說法。

**SLI、SLO、SLA** 可以讓你用數字量化「穩定」。

今天我們來談談如何定義和追蹤服務品質目標。

## 三個概念

### SLI (Service Level Indicator) - 服務品質指標

**定義**：可測量的服務品質指標。

**範例**：
- 可用性：成功請求 / 總請求
- 延遲：P95 回應時間
- 錯誤率：5xx 錯誤 / 總請求

### SLO (Service Level Objective) - 服務品質目標

**定義**：SLI 的目標值。

**範例**：
- 可用性 ≥ 99.9%
- P95 延遲 ≤ 200ms
- 錯誤率 ≤ 0.1%

### SLA (Service Level Agreement) - 服務品質協議

**定義**：與客戶的合約，如果違反 SLO，會有賠償。

**範例**：
- 如果可用性 < 99.9%，退款 10%
- 如果可用性 < 99%，退款 25%

## 關係

```
SLI: 實際測量的值（例如：99.95%）
SLO: 內部目標（例如：≥ 99.9%）
SLA: 對外承諾（例如：≥ 99%）
```

**建議**：SLO 應該比 SLA 更嚴格，留一些 buffer。

## 如何選擇 SLI

### 從使用者角度思考

**錯誤**：監控 CPU、Memory、Disk

**正確**：監控使用者體驗（可用性、延遲、錯誤率）

### 常見的 SLI

#### 1. 可用性（Availability）

```promql
sum(rate(http_requests_total{status=~"2.."}[30d]))
  /
sum(rate(http_requests_total[30d]))
```

#### 2. 延遲（Latency）

```promql
histogram_quantile(0.95, 
  sum by (le) (
    rate(http_request_duration_seconds_bucket[30d])
  )
)
```

#### 3. 錯誤率（Error Rate）

```promql
sum(rate(http_requests_total{status=~"5.."}[30d]))
  /
sum(rate(http_requests_total[30d]))
```

#### 4. 吞吐量（Throughput）

```promql
sum(rate(http_requests_total[5m]))
```

## 如何設定 SLO

### 步驟 1：收集歷史資料

查看過去 3 個月的 SLI：

```promql
# 過去 90 天的可用性
sum(rate(http_requests_total{status=~"2.."}[90d]))
  /
sum(rate(http_requests_total[90d]))
```

**結果**：99.95%

### 步驟 2：考慮使用者期望

- B2B 企業客戶：需要 99.9% 以上
- B2C 一般使用者：99% 可能就夠了

### 步驟 3：考慮成本

| 可用性 | 每年停機時間 | 成本 |
|--------|--------------|------|
| 99%    | 3.65 天      | 低   |
| 99.9%  | 8.76 小時    | 中   |
| 99.99% | 52.56 分鐘   | 高   |
| 99.999%| 5.26 分鐘    | 非常高 |

### 步驟 4：設定 SLO

```yaml
slo:
  - name: "API Availability"
    target: 99.9%
    window: 30d
    
  - name: "API P95 Latency"
    target: 200ms
    window: 30d
    
  - name: "API Error Rate"
    target: 0.1%
    window: 30d
```

## Error Budget（錯誤預算）

### 定義

如果 SLO 是 99.9%，那你有 **0.1% 的錯誤預算**。

在 30 天內：
- 總請求：10,000,000
- 允許失敗：10,000 次

### 用途

#### 1. 平衡可靠性與速度

如果錯誤預算還有很多，可以**加快發布速度**。

如果錯誤預算耗盡，應該**暫停新功能，專注修 bug**。

#### 2. 量化風險

```
部署新功能的風險 = 預期失敗次數 / 剩餘錯誤預算
```

如果風險 > 50%，不應該部署。

### 計算錯誤預算

```promql
# 剩餘錯誤預算
1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
    /
  sum(rate(http_requests_total[30d]))
) / 0.001
```

**結果**：
- 如果是 0.95，表示還有 95% 的錯誤預算
- 如果是 -0.5，表示超支 50%

## 實作：SLO 追蹤

### Prometheus Recording Rules

```yaml
groups:
  - name: slo
    interval: 1m
    rules:
      # 可用性 SLI
      - record: sli:availability:ratio_rate30d
        expr: |
          sum(rate(http_requests_total{status=~"2.."}[30d]))
            /
          sum(rate(http_requests_total[30d]))
      
      # P95 延遲 SLI
      - record: sli:latency:p95_30d
        expr: |
          histogram_quantile(0.95,
            sum by (le) (
              rate(http_request_duration_seconds_bucket[30d])
            )
          )
      
      # 錯誤率 SLI
      - record: sli:error_rate:ratio_rate30d
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[30d]))
            /
          sum(rate(http_requests_total[30d]))
      
      # 錯誤預算
      - record: slo:error_budget:remaining
        expr: |
          1 - (sli:error_rate:ratio_rate30d / 0.001)
```

### Grafana Dashboard

```json
{
  "panels": [
    {
      "title": "Availability SLO",
      "targets": [{
        "expr": "sli:availability:ratio_rate30d * 100"
      }],
      "thresholds": [
        { "value": 99.9, "color": "green" },
        { "value": 99, "color": "yellow" },
        { "value": 0, "color": "red" }
      ]
    },
    {
      "title": "Error Budget Remaining",
      "targets": [{
        "expr": "slo:error_budget:remaining * 100"
      }],
      "thresholds": [
        { "value": 50, "color": "green" },
        { "value": 20, "color": "yellow" },
        { "value": 0, "color": "red" }
      ]
    }
  ]
}
```

### 告警規則

```yaml
groups:
  - name: slo_alerts
    rules:
      # SLO 違反
      - alert: SLOViolation
        expr: sli:availability:ratio_rate30d < 0.999
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "Availability SLO violated"
      
      # 錯誤預算快耗盡
      - alert: ErrorBudgetLow
        expr: slo:error_budget:remaining < 0.2
        labels:
          severity: warning
        annotations:
          summary: "Error budget is running low ({{ $value | humanizePercentage }})"
      
      # 錯誤預算耗盡
      - alert: ErrorBudgetExhausted
        expr: slo:error_budget:remaining < 0
        labels:
          severity: critical
        annotations:
          summary: "Error budget exhausted, stop deploying!"
```

## 多窗口 SLO

不要只看 30 天，也要看短期。

### 1-hour Burn Rate

```promql
# 1 小時的錯誤率
sum(rate(http_requests_total{status=~"5.."}[1h]))
  /
sum(rate(http_requests_total[1h]))

# 與 SLO 的比較
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
    /
  sum(rate(http_requests_total[1h]))
) / 0.001
```

如果 Burn Rate > 10，表示正在以 **10 倍速度** 消耗錯誤預算。

### 多窗口告警

```yaml
- alert: HighBurnRate
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h]))
        /
      sum(rate(http_requests_total[1h]))
    ) / 0.001 > 10
  for: 5m
  annotations:
    summary: "Burning error budget at 10x rate"
```

## 實戰案例

### 案例：電商網站

**業務需求**：
- 結帳流程必須非常可靠
- 瀏覽商品可以容忍一些錯誤

**SLO 設定**：

| 服務          | SLI        | SLO     |
|---------------|------------|---------|
| 結帳 API      | 可用性     | 99.95%  |
| 結帳 API      | P95 延遲   | 500ms   |
| 商品瀏覽 API  | 可用性     | 99.5%   |
| 商品瀏覽 API  | P95 延遲   | 1s      |

### 案例：金融系統

**業務需求**：
- 交易必須 100% 可靠
- 查詢可以容忍一些延遲

**SLO 設定**：

| 服務          | SLI        | SLO     |
|---------------|------------|---------|
| 交易 API      | 可用性     | 99.99%  |
| 交易 API      | P95 延遲   | 200ms   |
| 查詢 API      | 可用性     | 99.9%   |
| 查詢 API      | P95 延遲   | 2s      |

## 實戰建議

### 1. 從簡單開始

不要一開始就設定 10 個 SLI，先從 3 個開始：
- 可用性
- P95 延遲
- 錯誤率

### 2. 定期回顧

每季回顧 SLO，根據實際情況調整。

### 3. 與團隊分享

在每週會議中，展示 SLO 和錯誤預算。

### 4. 用錯誤預算做決策

```
if error_budget > 50%:
    加快發布
elif error_budget > 20%:
    正常發布
else:
    停止發布，專注修 bug
```

---

**SLI、SLO、SLA 可以讓你用數字量化服務品質。**

當你有明確的目標，你就能平衡可靠性與速度，做出更好的決策。
