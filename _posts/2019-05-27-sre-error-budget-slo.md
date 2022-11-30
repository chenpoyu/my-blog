---
layout: post
title: "SRE 實踐：錯誤預算與 SLO"
date: 2019-05-27 10:00:00 +0800
categories: [DevOps, SRE]
tags: [SRE, SLO, SLI, Error Budget, Reliability]
---

上週學習了效能測試（參考 [效能測試策略](/posts/2019/05/20/performance-testing-strategy/)），這週來學習 Google 的 SRE（Site Reliability Engineering）實踐。

SRE 的核心概念：**用軟體工程方法解決運維問題**。

我們的問題：
- 開發團隊想快速迭代（每天上線）
- 運維團隊想保持穩定（不要亂改）
- 兩邊經常衝突

SRE 提供了一個解決方案：**錯誤預算 (Error Budget)**。

> 參考：《Site Reliability Engineering》by Google

## SRE 是什麼

傳統分工：
- 開發團隊：寫程式，追求新功能
- 運維團隊：維護系統，追求穩定

問題：
- 開發想上線 → 運維擔心出問題 → 衝突
- 出事時互相指責
- 缺乏量化標準

SRE 方法：
- 用工程方法處理運維
- 量化可靠性目標（SLO）
- 錯誤預算平衡速度與穩定

## SLI、SLO、SLA

### SLI (Service Level Indicator)

**服務水準指標**：衡量服務品質的具體指標。

常見 SLI：
- **可用性**：成功請求 / 總請求
- **延遲**：P95 回應時間
- **吞吐量**：每秒請求數
- **正確性**：正確回應 / 總回應

範例：
```
可用性 SLI = 成功請求數 / 總請求數
           = 999,000 / 1,000,000
           = 99.9%
```

### SLO (Service Level Objective)

**服務水準目標**：SLI 的目標值。

範例：
- 可用性 SLO：99.9%（每月允許 43 分鐘故障）
- 延遲 SLO：P95 < 200ms
- 錯誤率 SLO：< 0.1%

**不要追求 100%**！
- 100% 成本極高
- 使用者感知不到 99.9% vs 99.99%
- 阻礙創新（不敢上線新功能）

### SLA (Service Level Agreement)

**服務水準協議**：對外承諾，違反有懲罰。

範例：
```
SLA：99.95% 可用性
如果未達成：
- 99.0-99.95%：退款 10%
- 95.0-99.0%：退款 25%
- < 95.0%：退款 50%
```

關係：
```
SLA (對外承諾) = 99.9%
  ↓ (更嚴格)
SLO (內部目標) = 99.95%
  ↓ (測量)
SLI (實際表現) = 99.98%
```

## 錯誤預算

### 概念

如果 SLO 是 99.9%，表示允許 0.1% 的錯誤（**錯誤預算**）。

每月錯誤預算：
```
30 天 × 24 小時 × 60 分鐘 = 43,200 分鐘
錯誤預算 = 43,200 × 0.1% = 43.2 分鐘
```

### 使用錯誤預算

**場景一：預算充足**

目前可用性：99.95%
- 已使用：0.05% = 21.6 分鐘
- 剩餘預算：0.05% = 21.6 分鐘

決策：可以快速上線新功能（有容錯空間）

**場景二：預算耗盡**

目前可用性：99.85%
- 已使用：0.15% = 64.8 分鐘
- 超支：21.6 分鐘

決策：停止上線，專注穩定性（修 Bug、改善監控）

### 錯誤預算政策

制定明確的政策：

```markdown
## 錯誤預算政策

### 當錯誤預算充足時（> 50%）：
- 正常上線節奏（每天）
- 可以試驗性功能
- 允許適度風險

### 當錯誤預算告警時（10-50%）：
- 減緩上線（每 2-3 天）
- 增加測試範圍
- Code review 更嚴格

### 當錯誤預算耗盡時（< 10% 或負數）：
- 停止功能開發
- 只允許緊急修復和穩定性改善
- 進行事後分析 (Postmortem)
- 改善監控和測試

### 錯誤預算重置：
- 每月 1 號重置
- 或達成連續 7 天 SLO 後提前重置
```

## 設定 SLO

### 步驟一：選擇 SLI

針對 Order Service：

**可用性 SLI**：
```promql
sum(rate(http_requests_total{job="order-service", code=~"2.."}[5m])) /
sum(rate(http_requests_total{job="order-service"}[5m]))
```

**延遲 SLI**：
```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) by (le)
)
```

**錯誤率 SLI**：
```promql
sum(rate(http_requests_total{job="order-service", code=~"5.."}[5m])) /
sum(rate(http_requests_total{job="order-service"}[5m]))
```

### 步驟二：設定目標

根據業務需求和歷史資料：

| SLI | 目標 (SLO) | 說明 |
|-----|-----------|------|
| 可用性 | 99.9% | 每月 43 分鐘故障 |
| P95 延遲 | < 200ms | 95% 請求快速回應 |
| 錯誤率 | < 0.1% | 1000 個請求最多 1 個錯誤 |

### 步驟三：計算錯誤預算

```
時間窗口：30 天
總分鐘數：43,200
可用性 SLO：99.9%

錯誤預算 = 43,200 × (100% - 99.9%)
         = 43,200 × 0.1%
         = 43.2 分鐘/月
```

### 步驟四：監控

建立 Grafana Dashboard：

**可用性面板**：
```promql
# 當前可用性（7 天）
sum(rate(http_requests_total{job="order-service", code=~"2.."}[7d])) /
sum(rate(http_requests_total{job="order-service"}[7d])) * 100

# 顯示：99.95%（綠色）
```

**錯誤預算面板**：
```promql
# 剩餘錯誤預算（30 天）
(1 - (sum(rate(http_requests_total{job="order-service", code=~"2.."}[30d])) /
      sum(rate(http_requests_total{job="order-service"}[30d])))) /
(1 - 0.999) * 100

# 顯示：45%（黃色：還有 45% 預算）
```

**消耗速率面板**：
```promql
# 如果維持當前錯誤率，多久會耗盡預算
(1 - sum(rate(http_requests_total{job="order-service", code=~"2.."}[1h])) /
     sum(rate(http_requests_total{job="order-service"}[1h]))) /
(1 - 0.999) * 100

# 顯示：0.5%/小時（預計 90 小時耗盡）
```

### 步驟五：告警

**錯誤預算告警**：
```yaml
groups:
- name: slo
  rules:
  - alert: ErrorBudgetLow
    expr: |
      (1 - (sum(rate(http_requests_total{job="order-service", code=~"2.."}[30d])) /
            sum(rate(http_requests_total{job="order-service"}[30d])))) /
      (1 - 0.999) * 100 < 25
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Error budget low ({{ $value }}% remaining)"
      description: "Order service error budget is below 25%"

  - alert: ErrorBudgetExhausted
    expr: |
      (1 - (sum(rate(http_requests_total{job="order-service", code=~"2.."}[30d])) /
            sum(rate(http_requests_total{job="order-service"}[30d])))) /
      (1 - 0.999) * 100 < 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Error budget exhausted!"
      description: "Order service has exceeded its error budget. Stop feature releases!"
```

**快速消耗告警**：
```yaml
- alert: ErrorBudgetBurnRateTooHigh
  expr: |
    (1 - (sum(rate(http_requests_total{job="order-service", code=~"2.."}[1h])) /
          sum(rate(http_requests_total{job="order-service"}[1h])))) /
    (1 - 0.999) > 10  # 消耗速率 > 10x
  for: 15m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning too fast!"
    description: "At current rate, error budget will be exhausted in {{ $value }} hours"
```

## On-call 輪值

SRE 強調：開發團隊要參與 on-call（值班）。

### 輪值排程

使用 PagerDuty 或 OpsGenie：

```
週一-週二：Alice (主)、Bob (次)
週三-週四：Charlie (主)、Dave (次)
週五-週日：Eve (主)、Frank (次)
```

### On-call 手冊

為每個服務建立 Runbook：

```markdown
# Order Service Runbook

## 常見告警

### High Error Rate

**症狀**：錯誤率 > 1%

**可能原因**：
1. 資料庫連線失敗
2. 下游服務（Payment Service）故障
3. 無效請求（惡意流量）

**排查步驟**：
1. 檢查 Grafana Dashboard
2. 查看錯誤日誌：
   ```
   kubectl logs -l app=order-service --tail=100 | grep ERROR
   ```
3. 檢查下游服務狀態：
   ```
   curl http://payment-service/health
   ```

**緩解措施**：
- 如果是資料庫問題：切換到唯讀副本
- 如果是下游服務問題：啟用降級模式
- 如果是惡意流量：啟用 rate limiting

**聯絡人**：
- 團隊 Lead：Alice (alice@example.com, +886-912-345-678)
- DBA：Bob (bob@example.com, +886-912-345-679)

### High Latency

**症狀**：P95 > 500ms

...
```

### 值班報酬

值班應該有額外報酬：
- 平日值班：每天 $50
- 週末值班：每天 $100
- 處理事件：每次 $30

或是：值班時數可以換補休。

## Postmortem（事後分析）

事件發生後，進行無責備 (Blameless) 的分析。

### Postmortem 模板

```markdown
# Postmortem：Order Service 故障（2019-05-27）

## 摘要

**事件**：Order Service 完全無法使用
**影響**：所有使用者，持續 45 分鐘
**根本原因**：資料庫連線池耗盡

## 時間線

**14:00** - 部署新版本（v2.3.0）
**14:15** - 錯誤率開始上升（5%）
**14:20** - 錯誤率達 50%，觸發告警
**14:22** - On-call 工程師開始調查
**14:30** - 識別出資料庫連線問題
**14:35** - 嘗試增加連線池大小（無效）
**14:40** - 決定回滾到 v2.2.0
**14:45** - 回滾完成，服務恢復正常

## 根本原因

新版本的程式碼有 bug：

```java
// v2.3.0（錯誤）
public List<Order> getOrders(Long userId) {
    Connection conn = dataSource.getConnection();
    // 查詢...
    // 忘記關閉連線！
    return orders;
}

// v2.2.0（正確）
public List<Order> getOrders(Long userId) {
    try (Connection conn = dataSource.getConnection()) {
        // 查詢...
        return orders;
    } // 自動關閉連線 
}
```

連線沒有釋放，逐漸耗盡連線池（最大 50 個連線）。

## 影響

- 受影響使用者：~50,000
- 損失訂單：~200
- 錯誤預算消耗：45 分鐘 / 43.2 分鐘 = 104%（超支！）

## 緩解措施

- 14:40：回滾到舊版本

## 改進行動 (Action Items)

1. **[P0] 修復 bug**（負責人：Alice，截止：5/28）
   - 修正連線洩漏
   - 增加單元測試驗證連線釋放

2. **[P0] 改善監控**（負責人：Bob，截止：5/30）
   - 新增資料庫連線池監控
   - 告警：連線池使用率 > 80%

3. **[P1] 改善測試**（負責人：Charlie，截止：6/5）
   - 新增負載測試（模擬 1000 並發）
   - CI/CD 必須通過負載測試

4. **[P1] Code Review 流程**（負責人：Dave，截止：6/10）
   - 強制使用 try-with-resources
   - 設定 SonarQube 檢查資源洩漏

5. **[P2] 改善回滾流程**（負責人：Eve，截止：6/15）
   - 文件化快速回滾步驟
   - 練習一鍵回滾

## 經驗教訓

**做得好的**：
- 告警及時（5 分鐘內）
- 快速識別問題（20 分鐘）
- 決策果斷（回滾而不是繼續嘗試修復）

**需要改進的**：
- Code Review 沒有發現 bug
- 缺少負載測試（本地測試未發現問題）
- 監控不足（沒有連線池監控）

## 錯誤預算影響

這次事件消耗了本月 104% 的錯誤預算（超支 4%）。

**結果**：
- 停止所有功能開發（直到 6/1 預算重置）
- 只允許穩定性改善和 bug 修復
- 每日回顧 SLO 達成狀況
```

### 重點

1. **無責備**：不追究個人責任，focus 在系統改進
2. **時間線**：詳細記錄事件發生過程
3. **根本原因**：深入分析，不只是表面原因
4. **Action Items**：具體行動，指定負責人和截止日期
5. **追蹤**：定期檢查 Action Items 完成狀況

## Toil 管理

**Toil**：重複性、手動、無價值的運維工作。

範例：
- 手動重啟服務
- 手動擴展 Pod
- 手動清理日誌
- 手動部署

SRE 目標：**Toil < 50% 工作時間**

另外 50% 用在：
- 自動化
- 改善系統
- 工程專案

### 識別 Toil

記錄所有運維任務：

| 任務 | 頻率 | 時間 | Toil? | 自動化優先級 |
|-----|------|------|-------|-------------|
| 重啟服務 | 每週 3 次 | 5 分鐘 | ✅ | 高 |
| 擴展 Pod | 每天 2 次 | 2 分鐘 | ✅ | 高 |
| 清理日誌 | 每週 1 次 | 30 分鐘 | ✅ | 中 |
| Code Review | 每天 5 次 | 30 分鐘 | ❌ | - |
| On-call 值班 | 每月 4 天 | - | ❌ | - |

### 消除 Toil

**重啟服務**：
```yaml
# 使用 liveness probe，自動重啟
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

**擴展 Pod**：
```yaml
# 使用 HPA，自動擴展
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**清理日誌**：
```bash
# Cron job
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 2 * * *"  # 每天 2:00 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command:
            - /bin/sh
            - -c
            - find /var/log -name "*.log" -mtime +7 -delete
```

## 心得

SRE 的概念讓我們團隊更和諧了。以前開發和運維經常吵架（「你的 bug」vs「你亂改」），現在有了錯誤預算這個客觀標準。

錯誤預算很有用：
- 預算充足時：快速上線，嘗試新東西
- 預算不足時：放慢步調，專注穩定
- 沒有主觀爭論，只有客觀數據

Postmortem 也很重要。以前出事就找人罵，現在 focus 在改進系統。每次事件都讓系統更強壯。

不過要注意：
1. **SLO 不要太嚴格**：99.99% 很難達成，會限制創新
2. **Toil 要持續減少**：不然 SRE 就變成救火員
3. **On-call 要合理**：不能讓人過勞

SRE 不只是運維，是一種文化和方法論。

下週要研究雲端成本優化，學習如何降低 AWS 帳單。
