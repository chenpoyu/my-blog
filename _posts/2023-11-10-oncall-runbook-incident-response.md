---
layout: post
title: "On-Call Runbook：標準化事件處理流程"
date: 2023-11-10 09:00:00 +0800
categories: [Observability, SRE]
tags: [On-Call, Runbook, Incident Response, SRE]
---

凌晨 3 點，你被告警吵醒。半夢半醒中，你需要快速定位並解決問題。

**Runbook** 可以讓你在壓力下，也能快速做出正確的決策。

今天我們來談談如何建立標準化的事件處理流程。

## Runbook 是什麼？

Runbook 是一份**操作手冊**，記錄了：
- 如何判斷問題的嚴重性
- 如何快速定位問題
- 如何解決問題
- 如何防止問題再次發生

## Runbook 的結構

### 基本結構

```markdown
# [服務名稱] - [問題描述]

## 1. 症狀（Symptoms）
- 告警名稱
- 可能的使用者影響

## 2. 嚴重性（Severity）
- P0 / P1 / P2 / P3

## 3. 快速診斷（Quick Diagnosis）
- 檢查哪些 Dashboard
- 檢查哪些 Logs
- 檢查哪些 Metrics

## 4. 解決步驟（Resolution Steps）
- 詳細的操作步驟

## 5. 預防措施（Prevention）
- 如何避免問題再次發生

## 6. 相關資源（Related Resources）
- Dashboard 連結
- Logs 查詢語句
- 相關文件
```

## 實戰：High Error Rate Runbook

```markdown
# Order Service - High Error Rate

## 1. 症狀

**告警**：`OrderServiceHighErrorRate`

**可能的使用者影響**：
- 使用者無法下單
- 下單後沒有收到確認郵件

## 2. 嚴重性

**P1**（主要功能無法使用，需要在 1 小時內解決）

## 3. 快速診斷

### Step 1: 檢查 Grafana Dashboard

開啟 Dashboard：http://grafana:3000/d/order-service

檢查：
- Error Rate Panel：錯誤率是否 > 5%？
- P95 Latency Panel：延遲是否異常？
- Request Volume Panel：流量是否異常？

### Step 2: 檢查 Jaeger

開啟 Jaeger：http://jaeger:16686

搜尋：
```
service=order-service
tags={"error": "true"}
minDuration=1s
```

找出最慢或失敗的 Trace。

### Step 3: 檢查 Kibana

開啟 Kibana：http://kibana:5601

查詢：
```
service:"order-service" AND level:ERROR AND @timestamp:[now-15m TO now]
```

找出最近 15 分鐘的錯誤日誌。

## 4. 解決步驟

### 情況 1：資料庫連線失敗

**症狀**：
- Logs 中有 `Connection timeout` 錯誤
- Dashboard 顯示 Database Connection Pool 用盡

**解決步驟**：

1. **檢查資料庫狀態**：
   ```bash
   kubectl get pods -n database
   ```
   
2. **檢查資料庫 CPU/Memory**：
   ```bash
   kubectl top pods -n database
   ```
   
3. **如果資料庫正常，重啟 Order Service**：
   ```bash
   kubectl rollout restart deployment/order-service -n production
   ```
   
4. **如果資料庫異常，聯絡 DBA**：
   - Slack: #database
   - On-Call: dba-oncall@example.com

### 情況 2：第三方 API (Stripe) 失敗

**症狀**：
- Logs 中有 `Stripe API returned 500` 錯誤
- Jaeger 顯示 `stripe-api` Span 失敗

**解決步驟**：

1. **檢查 Stripe 狀態頁**：https://status.stripe.com
   
2. **如果 Stripe 有問題**：
   - 啟用降級模式（使用備用支付方式）
   ```bash
   kubectl set env deployment/order-service -n production FALLBACK_PAYMENT=enabled
   ```
   
3. **通知相關團隊**：
   - Slack: #payment-team

### 情況 3：記憶體洩漏

**症狀**：
- Dashboard 顯示 Memory Usage 持續上升
- Logs 中有 `OutOfMemoryError`

**解決步驟**：

1. **立即重啟服務**（臨時解決）：
   ```bash
   kubectl rollout restart deployment/order-service -n production
   ```
   
2. **收集 Heap Dump**：
   ```bash
   kubectl exec -it order-service-xxx -n production -- jmap -dump:live,format=b,file=/tmp/heapdump.hprof 1
   kubectl cp production/order-service-xxx:/tmp/heapdump.hprof ./heapdump.hprof
   ```
   
3. **分析 Heap Dump**：
   使用 Eclipse MAT 或 VisualVM
   
4. **建立 Jira Ticket**：記錄問題，安排修復

### 情況 4：流量激增

**症狀**：
- Dashboard 顯示 Request Volume 突然增加 10 倍
- P95 Latency 上升

**解決步驟**：

1. **檢查流量來源**：
   ```bash
   kubectl logs deployment/order-service -n production | grep "User-Agent" | sort | uniq -c | sort -rn | head -20
   ```
   
2. **如果是攻擊，啟用 Rate Limiting**：
   ```bash
   kubectl apply -f rate-limit.yaml
   ```
   
3. **如果是正常流量，擴容**：
   ```bash
   kubectl scale deployment/order-service -n production --replicas=10
   ```

## 5. 預防措施

- [ ] 加入資料庫連線池監控
- [ ] 加入 Circuit Breaker 到第三方 API 呼叫
- [ ] 定期檢查記憶體使用情況
- [ ] 建立 Auto-scaling 規則

## 6. 相關資源

### Dashboards
- [Order Service Dashboard](http://grafana:3000/d/order-service)
- [Database Dashboard](http://grafana:3000/d/database)

### Logs
```
service:"order-service" AND level:ERROR
```

### Metrics
```promql
rate(http_requests_total{service="order-service",status="500"}[5m])
```

### Alerts
- [AlertManager](http://alertmanager:9093)

### Contacts
- **Team Lead**: @john (Slack)
- **DBA On-Call**: dba-oncall@example.com
- **Payment Team**: #payment-team (Slack)

## 7. 事後檢討（Postmortem）

完成事件處理後，建立 Postmortem 文件：
- 問題發生的時間
- 問題的根本原因
- 解決步驟
- 如何防止問題再次發生
```

## Runbook 模板

### 模板檔案

```markdown
# [服務名稱] - [問題描述]

## 1. 症狀
- [ ] 告警名稱：
- [ ] 使用者影響：

## 2. 嚴重性
- [ ] P0 / P1 / P2 / P3

## 3. 快速診斷

### Grafana Dashboard
- URL: 
- 檢查項目：

### Jaeger
- URL: 
- 搜尋條件：

### Kibana
- URL: 
- 查詢語句：

## 4. 解決步驟

### 情況 1: [情況描述]
症狀：

解決步驟：
1. 
2. 
3. 

### 情況 2: [情況描述]
症狀：

解決步驟：
1. 
2. 
3. 

## 5. 預防措施
- [ ] 
- [ ] 

## 6. 相關資源
- Dashboards: 
- Logs: 
- Metrics: 
- Contacts: 

## 7. 事後檢討
- [ ] 建立 Postmortem 文件
```

## 自動化 Runbook

### 用 Ansible 自動化常見操作

```yaml
# runbooks/restart-service.yml
---
- name: Restart Service
  hosts: "{{ service }}"
  tasks:
    - name: Restart service
      systemd:
        name: "{{ service }}"
        state: restarted
```

執行：

```bash
ansible-playbook runbooks/restart-service.yml -e "service=order-service"
```

### 用 Kubernetes Job 自動化

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: clear-cache
spec:
  template:
    spec:
      containers:
      - name: clear-cache
        image: redis:7
        command: ["redis-cli", "-h", "redis", "FLUSHALL"]
      restartPolicy: Never
```

執行：

```bash
kubectl create -f clear-cache-job.yaml
```

## On-Call 輪值

### 輪值表

| 週次 | Primary On-Call | Secondary On-Call |
|------|-----------------|-------------------|
| Week 1 | Alice | Bob |
| Week 2 | Bob | Charlie |
| Week 3 | Charlie | Alice |

### PagerDuty 整合

```yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_SERVICE_KEY'
        description: '{{ .CommonAnnotations.summary }}'
        details:
          runbook: '{{ .CommonAnnotations.runbook }}'
          dashboard: '{{ .CommonAnnotations.dashboard }}'

route:
  receiver: 'pagerduty'
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
```

## 實戰建議

### 1. 每個告警都要有 Runbook

在 Prometheus 告警規則中，加入 Runbook 連結：

```yaml
annotations:
  runbook: "https://wiki.example.com/runbooks/order-service-high-error-rate"
```

### 2. 定期演練

每個月進行一次 Chaos Engineering 演練，測試 Runbook 是否有效。

### 3. 持續改進

每次事件後，更新 Runbook，加入新的情況和解決步驟。

### 4. 簡化步驟

盡量自動化，減少手動操作。

---

**Runbook 可以讓你在壓力下，也能快速做出正確的決策。**

當你有清晰的操作手冊，MTTR（平均修復時間）可以降低 50% 以上。
