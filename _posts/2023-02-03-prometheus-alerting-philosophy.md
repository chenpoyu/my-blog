---
layout: post
title: "告警哲學：如何避免「狼來了效應」"
date: 2023-02-03 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Prometheus, Alerting, AlertManager, SRE]
---

前三週我們學會了收集指標、視覺化圖表。但有個殘酷的現實：**你不可能 24 小時盯著 Dashboard。**

這時候就需要「告警（Alerting）」——讓系統在出問題時主動通知你。

但我要先潑你一盆冷水：**90% 的團隊做錯了告警，導致工程師疲於奔命，最後乾脆忽略所有告警。**

> 本文使用版本: **Prometheus 2.40 + AlertManager 0.25**

## 狼來了效應：告警疲勞的真相

我見過最慘的案例是這樣的：

某個團隊設定了 50 條告警規則，包括：
- CPU 使用率 > 70%
- 記憶體使用率 > 80%
- 磁碟使用率 > 85%
- API 回應時間 > 1 秒
- ...（還有 46 條）

結果呢？**每天收到 300 封告警郵件，工程師直接設定郵件規則自動刪除。**

當真正的災難發生時（資料庫當機），告警淹沒在一堆「CPU 又超過 70% 了」的噪音中，沒人注意到。

這就是「狼來了效應」。

## Google SRE 的告警黃金法則

Google SRE 團隊提出了一個簡單但強大的原則：

> **每一則告警，都應該是可執行的（actionable）。**

什麼叫「可執行的」？

✅ **這則告警代表使用者正在受影響**（例如：API 錯誤率 > 5%）  
✅ **收到告警的人有能力處理**（不是通知前端工程師後端資料庫掛了）  
✅ **需要立即處理**（不是「CPU 可能未來會滿」）

如果一則告警不符合以上三點，**那就不該是告警，而是應該放在 Dashboard 上定期檢視。**

## Prometheus 告警規則實戰

### 基本架構

Prometheus 的告警分兩層：

1. **Prometheus Server**：評估告警規則，決定是否觸發
2. **AlertManager**：接收告警，負責通知（郵件、Slack、PagerDuty 等）

### 安裝 AlertManager

```bash
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  prom/alertmanager:v0.25.0
```

修改 `prometheus.yml`，加入 AlertManager：

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['host.docker.internal:9093']

rule_files:
  - "alert_rules.yml"
```

### 第一條告警規則：API 高錯誤率

建立 `alert_rules.yml`：

```yaml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "API 錯誤率過高"
          description: "{{ $labels.job }} 的錯誤率為 {{ $value | humanizePercentage }}，已持續超過 2 分鐘"
```

重點解析：

- **`expr`**：PromQL 查詢，當錯誤率 > 5% 時觸發
- **`for: 2m`**：必須持續 2 分鐘才告警（避免短暫波動）
- **`severity: critical`**：嚴重程度標籤
- **`annotations`**：告警訊息內容

### 第二條規則：API 回應過慢

```yaml
- alert: HighLatency
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
    ) > 1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "API 回應時間過長"
    description: "{{ $labels.job }} 的 P95 延遲為 {{ $value }}s，已持續 5 分鐘"
```

注意 `severity: warning`——這不是立即災難，但需要關注。

### 第三條規則：服務掛掉

```yaml
- alert: ServiceDown
  expr: up == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "服務無法連線"
    description: "{{ $labels.job }} ({{ $labels.instance }}) 無法抓取指標"
```

這是最嚴重的告警，服務完全掛掉了。

## 告警嚴重程度分級

我的建議是分三級：

### Critical（緊急）
- **定義**：使用者正在受影響，需要立即處理
- **通知方式**：簡訊 + 電話 + Slack
- **範例**：
  - 服務完全掛掉
  - 資料庫連不上
  - API 錯誤率 > 10%

### Warning（警告）
- **定義**：可能影響使用者，需要在工作時間處理
- **通知方式**：Email + Slack
- **範例**：
  - P95 延遲 > 1 秒
  - 磁碟使用率 > 85%
  - 記憶體使用率 > 90%

### Info（資訊）
- **定義**：需要知道但不緊急
- **通知方式**：只在 Dashboard 顯示
- **範例**：
  - CPU 使用率 > 70%（不是立即問題）
  - 快取命中率下降

## AlertManager 設定

建立 `alertmanager.yml`：

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'job']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'team-email'
  
  routes:
    - match:
        severity: critical
      receiver: 'team-pager'
      continue: true
    
    - match:
        severity: warning
      receiver: 'team-slack'

receivers:
  - name: 'team-email'
    email_configs:
      - to: 'team@company.com'
        from: 'alertmanager@company.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alertmanager@company.com'
        auth_password: 'your-password'
  
  - name: 'team-pager'
    pagerduty_configs:
      - service_key: 'your-pagerduty-key'
  
  - name: 'team-slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
```

重點：
- **`group_by`**：相同類型的告警合併成一則通知
- **`repeat_interval`**：告警未解決前，每 12 小時再通知一次
- **路由規則**：critical 送 PagerDuty，warning 送 Slack

## 真實案例：電商大促期間的告警策略

我曾經參與過一次雙 11 活動，流量是平時的 50 倍。我們的告警策略是：

### 大促前（調整閾值）

```yaml
# 平時：錯誤率 > 1% 就告警
# 大促期間：調整為 > 5%（因為流量大，絕對錯誤數會增加）

- alert: HighErrorRate
  expr: |
    (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) > 
    {{ if eq $labels.env "promo" }}0.05{{ else }}0.01{{ end }}
```

### 大促期間（分級處理）

- **Critical**：付款流程失敗 → 立即處理
- **Warning**：商品頁面變慢 → 記錄但不中斷
- **Info**：搜尋功能偶爾超時 → 容忍

### 大促後（復盤）

檢視所有觸發的告警：
- 哪些是真正的問題？
- 哪些是誤報（false positive）？
- 閾值是否需要調整？

## 告警設計的反模式

我見過太多團隊犯這些錯誤：

### ❌ 反模式 1：監控系統資源而非使用者體驗

錯誤：
```yaml
- alert: HighCPU
  expr: node_cpu_usage > 80%
```

正確：
```yaml
- alert: SlowAPI
  expr: http_request_duration_p95 > 1
```

**為什麼？**CPU 高不一定影響使用者，但 API 慢一定影響。

### ❌ 反模式 2：設定過低的閾值

錯誤：
```yaml
- alert: AnyError
  expr: http_requests_total{status="500"} > 0
```

這會導致每次有一個 500 錯誤就告警。在大流量系統中，偶爾的錯誤是正常的。

正確：
```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status="500"}[5m]) > 10  # 每秒超過 10 個錯誤
  for: 2m
```

### ❌ 反模式 3：沒有 `for` 參數

錯誤：
```yaml
- alert: HighLatency
  expr: api_latency > 1
```

這會在短暫波動時就觸發。應該加上：
```yaml
for: 5m  # 持續 5 分鐘才告警
```

## 告警的終極目標

我常跟團隊說：

> **一個成熟的監控系統，應該讓工程師能安心睡覺。**

如果你每週都被半夜叫醒，問題不是系統不穩定（當然也可能是），而是**告警策略失敗了**。

理想狀態是：
- 收到告警 = 真的有問題
- 沒收到告警 = 系統真的健康
- 半夜被叫醒 = 真的是災難

---

**告警不是越多越好，而是越精準越好。**

寧可錯過一些小問題，也不要讓工程師在噪音中失去警覺。

當「狼真的來了」時，你要確保團隊會立刻反應，而不是習慣性忽略。
