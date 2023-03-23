---
layout: post
title: "設計完整的監控體系：從零到成熟"
date: 2023-03-24 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Monitoring Strategy, SRE, Best Practices, Maturity Model]
---

過去 12 週我們學了很多 Prometheus 的技術細節。但今天我要問一個根本的問題：**你有「監控策略」嗎？**

很多團隊就是「看到什麼就監控什麼」，結果：
- 監控了 500 個指標，但不知道哪個重要
- 收到告警卻不知道該怎麼處理
- 出問題時還是靠猜

今天我們要談「監控體系設計」——如何從零開始建立一個完整、有效的監控策略。

## 監控的四個層次

Google SRE 提出的監控金字塔：

```
┌─────────────────────────┐
│  使用者體驗監控 (RUM)    │  ← 最重要
├─────────────────────────┤
│  業務指標監控            │
├─────────────────────────┤
│  應用程式監控 (APM)      │
├─────────────────────────┤
│  基礎設施監控            │  ← 基礎
└─────────────────────────┘
```

### 第一層：基礎設施監控

**監控對象**：CPU、記憶體、磁碟、網路

**工具**：Prometheus + Node Exporter

**關鍵指標**：
```promql
# CPU 使用率
instance:node_cpu:utilization

# 記憶體使用率
instance:node_memory:utilization

# 磁碟 I/O
rate(node_disk_io_time_seconds_total[5m])

# 網路頻寬
instance:node_network:receive_mbps
```

**告警閾值**：
- CPU > 80% 持續 10 分鐘 → Warning
- 記憶體 > 90% 持續 5 分鐘 → Critical
- 磁碟使用率 > 85% → Warning
- 磁碟使用率 > 95% → Critical

### 第二層：應用程式監控

**監控對象**：API、服務、資料庫、快取

**工具**：Prometheus + 各種 Exporter

**關鍵指標（RED Method）**：
```promql
# Rate（速率）
job:http_requests:rate5m

# Errors（錯誤）
job:http_requests:error_rate5m

# Duration（延遲）
job:http_request_duration:p95
```

**告警閾值**：
- 錯誤率 > 1% 持續 5 分鐘 → Warning
- 錯誤率 > 5% 持續 2 分鐘 → Critical
- P95 延遲 > 1s 持續 10 分鐘 → Warning
- P95 延遲 > 3s 持續 5 分鐘 → Critical

### 第三層：業務指標監控

**監控對象**：訂單、營收、使用者行為

**工具**：Prometheus + 自定義埋點

**關鍵指標**：
```promql
# 訂單轉換率
increase(orders_completed_total[1h]) / increase(orders_created_total[1h])

# 客單價
sum(order_amount_total) / count(orders_completed_total)

# 購物車放棄率
(increase(cart_created_total[1h]) - increase(orders_created_total[1h])) / increase(cart_created_total[1h])
```

**告警閾值**：
- 訂單轉換率下降 > 20% → Critical
- 營收比昨天同時段下降 > 30% → Critical

### 第四層：使用者體驗監控（RUM）

**監控對象**：前端效能、真實使用者體驗

**工具**：Google Analytics、New Relic Browser、Sentry

**關鍵指標**：
- 首次內容繪製（FCP）
- 最大內容繪製（LCP）
- 累積版面配置位移（CLS）
- 互動到首次繪製（FID）

這部分通常不用 Prometheus，但可以整合到同一個 Grafana Dashboard。

## 避免監控盲點

我見過很多「監控齊全」的系統，出問題時還是找不到根因。為什麼？**因為有盲點。**

### 盲點 1：只監控成功，不監控失敗

錯誤範例：

```java
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable Long id) {
    requestCounter.inc();  // ✅ 有記錄請求
    return userService.findById(id);
}
```

**問題**：如果 `findById()` 拋出異常，`requestCounter` 還是會增加，但你不知道失敗了。

正確範例：

```java
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable Long id) {
    try {
        User user = userService.findById(id);
        requestCounter.labels("success").inc();
        return user;
    } catch (UserNotFoundException e) {
        requestCounter.labels("not_found").inc();
        throw e;
    } catch (Exception e) {
        requestCounter.labels("error").inc();
        throw e;
    }
}
```

### 盲點 2：只監控平均值，不監控長尾

平均值會騙人。

假設你的 API 回應時間：
- 99% 的請求在 100ms 內
- 1% 的請求要 10 秒

**平均值**：199ms（看起來不錯）  
**P99**：10s（使用者會氣炸）

**解決方案**：永遠監控 P95、P99，不要只看平均值。

### 盲點 3：只監控服務，不監控依賴

你的 API 正常，但：
- 資料庫慢查詢增加了 → 沒監控
- Redis 記憶體快滿了 → 沒監控
- 第三方支付 API 超時率上升 → 沒監控

**解決方案**：用 Exporter 監控所有依賴。

### 盲點 4：只監控生產環境，不監控測試環境

很多問題在測試環境就能發現，但因為沒監控，等到上生產才爆炸。

**解決方案**：測試環境也要有完整監控（可以用較低的保留時間降低成本）。

## 監控成熟度模型

我設計了一個「五級成熟度模型」，看看你的團隊在哪一級？

### Level 0：無監控

**特徵**：
- 沒有任何監控系統
- 靠使用者回報問題
- 出問題時用 `top`、`ps` 手動查

**問題**：
- 客戶比你更早發現問題
- 無法量化系統健康度
- 無法做容量規劃

### Level 1：基礎監控

**特徵**：
- 有 CPU、記憶體、磁碟監控
- 有基本的 Dashboard
- 沒有告警（或告警很少）

**問題**：
- 只知道「機器的狀態」，不知道「服務的狀態」
- 必須主動去看 Dashboard

### Level 2：服務監控 + 告警

**特徵**：
- 監控 API 的 QPS、延遲、錯誤率
- 設定了告警規則
- 有 on-call 輪值制度

**問題**：
- 告警太多（狼來了效應）
- 不知道告警的優先順序
- 沒有 Runbook（收到告警不知道該怎麼處理）

### Level 3：監控 + SLO

**特徵**：
- 定義了 SLO（Service Level Objective）
- 告警基於 SLO（例如：錯誤預算快用完時才告警）
- 有明確的 Runbook
- 定期檢討告警品質

**這是大部分成熟團隊的目標。**

### Level 4：全棧可觀測性

**特徵**：
- Metrics + Logs + Traces 完整整合
- 從告警到根因分析完全自動化
- 用 AIOps 預測問題
- 監控驅動架構決策

**這是 Google、Netflix 等公司的水準。**

## 如何設計 SLO（服務水準目標）

SLO 是「成熟監控」的關鍵。

### SLO 三要素

1. **SLI（Service Level Indicator）**：可量化的指標
2. **目標值**：例如「99.9%」
3. **時間窗口**：例如「過去 30 天」

### 範例：API 服務的 SLO

**SLI 1：可用性**
```
SLO: 99.9% 的請求在過去 30 天內成功（status < 500）
```

```promql
sum(rate(http_requests_total{status!~"5.."}[30d]))
/
sum(rate(http_requests_total[30d]))
* 100 >= 99.9
```

**SLI 2：延遲**
```
SLO: 95% 的請求在過去 30 天內回應時間 < 500ms
```

```promql
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[30d])) by (le)
) < 0.5
```

### 錯誤預算（Error Budget）

如果 SLO 是 99.9%，那「錯誤預算」就是 0.1%。

```
每月錯誤預算 = 30 天 * 24 小時 * 60 分鐘 * 0.1% = 43.2 分鐘
```

意思是：**每個月可以掛 43 分鐘。**

### 基於 SLO 的告警

不要在「錯誤率 > 1%」時告警，而是在「錯誤預算快用完」時告警。

```yaml
- alert: ErrorBudgetBurning
  expr: |
    (
      1 - (
        sum(rate(http_requests_total{status!~"5.."}[1h]))
        /
        sum(rate(http_requests_total[1h]))
      )
    ) > 0.001  # 錯誤率 > 0.1%
  for: 5m
  annotations:
    summary: "錯誤預算正在快速消耗"
    description: "如果持續這個速度，錯誤預算將在 {{ $value }} 小時內用完"
```

## 實戰：監控 Runbook

當告警觸發時，工程師應該有明確的處理步驟。

### Runbook 範例：HighErrorRate

```markdown
# 告警：HighErrorRate

## 嚴重程度
Critical

## 影響範圍
使用者無法正常使用服務

## 立即行動（5 分鐘內）
1. 檢查 Grafana Dashboard：http://grafana/d/api-overview
2. 確認哪個 endpoint 在出錯：
   ```
   topk(5, sum(rate(http_requests_total{status=~"5.."}[5m])) by (endpoint))
   ```
3. 檢查日誌是否有異常：
   ```
   kubectl logs -l app=api-server --tail=100 | grep ERROR
   ```

## 常見根因
1. **資料庫連線池耗盡**
   - 檢查：`db_connections_active / db_connections_max`
   - 解決：重啟服務或調高連線池上限

2. **外部 API 超時**
   - 檢查：`rate(external_api_timeout_total[5m])`
   - 解決：啟用降級模式（返回快取數據）

3. **記憶體不足導致 OOM**
   - 檢查：`container_memory_usage_bytes / container_spec_memory_limit_bytes`
   - 解決：重啟 Pod 或水平擴展

## 升級路徑
如果 30 分鐘內無法解決，升級給 Tech Lead
```

## 監控的投資回報率（ROI）

老闆可能會問：「監控要花多少錢？值得嗎？」

### 成本估算

假設一個中型團隊（10 個服務、50 台機器）：

- **Prometheus Server**：$200/月（2 vCPU、8GB RAM）
- **Thanos（長期儲存）**：$500/月（S3 儲存）
- **Grafana**：$0（開源）或 $100/月（Grafana Cloud）
- **AlertManager**：$50/月（1 vCPU、2GB RAM）
- **工程時間**：每週 4 小時維護 × $100/hr = $1600/月

**總成本：約 $2500/月**

### 收益估算

假設沒有監控：

- **平均每月 1 次嚴重事故**
- **每次事故影響 2 小時**
- **影響 1000 個使用者**
- **每個使用者價值 $100/月**

**損失 = 1000 使用者 × $100 × (2hr / 720hr) = $278/次**

加上：
- 工程師加班處理事故：4 人 × 4 小時 × $100/hr = $1600
- 商譽損失：無法量化，但很重要

**每次事故至少損失 $2000**

**ROI = $2000 / $2500 = 80%**（不含商譽）

---

**監控不是成本，是投資。**

當你能在問題影響使用者前就發現並修復，你就證明了監控的價值。
