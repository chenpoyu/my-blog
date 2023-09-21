---
layout: post
title: "Q4 起點：可觀測性自動化的價值"
date: 2023-09-22 09:00:00 +0800
categories: [Observability, Automation]
tags: [Automation, SRE, Infrastructure as Code, GitOps]
---

前面三個季度，我們學習了 Metrics、Logs、Traces。但**手動配置和維護**這些系統，成本很高。

從今天開始，我們進入 Q4：**可觀測性自動化**。

## 為什麼需要自動化？

### 場景 1：新服務上線

傳統流程：

```
1. 在 Prometheus 中手動加入 Scrape Config
2. 在 Grafana 中手動建立 Dashboard
3. 在 AlertManager 中手動加入告警規則
4. 在 Jaeger 中手動加入 Service
5. 在 Kibana 中手動建立 Index Pattern
```

**問題**：
- 耗時（每個服務 30 分鐘）
- 容易出錯（忘記某個步驟）
- 不一致（每個服務的配置不同）

### 場景 2：修改告警規則

你想把所有服務的「CPU > 80%」告警改成「CPU > 90%」。

傳統方式：**手動修改 50 個告警規則**。

### 場景 3：擴展到新環境

你要在測試環境、預生產環境、生產環境都部署相同的可觀測性系統。

傳統方式：**手動部署 3 次**。

## 自動化的層次

### Level 1：腳本化（Scripting）

用 Shell 腳本自動化簡單的任務。

```bash
#!/bin/bash
# 為新服務建立 Grafana Dashboard

SERVICE_NAME=$1

curl -X POST http://grafana:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d '{
    "dashboard": {
      "title": "'"$SERVICE_NAME"' Dashboard",
      "panels": [
        {
          "title": "Request Rate",
          "targets": [
            {"expr": "rate(http_requests_total{service=\"'"$SERVICE_NAME"'\"}[5m])"}
          ]
        }
      ]
    }
  }'
```

**優點**：簡單、快速

**缺點**：
- 難以維護（Shell 腳本不好寫）
- 沒有版本控制
- 沒有回滾機制

### Level 2：Infrastructure as Code (IaC)

用宣告式的配置檔案定義基礎設施。

**Terraform 範例**：

```hcl
resource "grafana_dashboard" "service" {
  for_each = toset(var.services)
  
  config_json = templatefile("${path.module}/dashboard.json", {
    service_name = each.key
  })
}

variable "services" {
  default = ["order-service", "payment-service", "notification-service"]
}
```

**優點**：
- 宣告式（描述「是什麼」，而不是「怎麼做」）
- 版本控制（可以用 Git 管理）
- 可重複（可以在多個環境執行）

### Level 3：GitOps

把 IaC 配置檔案放在 Git 中，自動部署到環境。

```
Git Repo (配置檔案) → CI/CD Pipeline → Kubernetes (實際環境)
```

**ArgoCD 範例**：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: observability
spec:
  source:
    repoURL: https://github.com/your-org/observability-config
    path: production
  destination:
    server: https://kubernetes.default.svc
    namespace: observability
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

當你修改 Git Repo 中的配置檔案，ArgoCD 會自動部署到 Kubernetes。

### Level 4：自我修復（Self-Healing）

系統自動偵測問題並修復。

**範例**：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: order-service
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          failureThreshold: 3
```

如果 Pod 不健康，Kubernetes 會自動重啟。

## Q4 的學習路線

### Week 37-40：Service Discovery 與自動化

- **Consul**：服務註冊與發現
- **Prometheus Service Discovery**：自動發現服務
- **Grafana Provisioning**：自動建立 Dashboard

### Week 41-44：Synthetic Monitoring 與外部監控

- **Blackbox Exporter**：監控外部 API
- **Synthetic Monitoring**：模擬使用者行為
- **Uptime Monitoring**：監控服務可用性

### Week 45-48：Infrastructure as Code

- **Ansible**：自動化配置管理
- **Terraform**：自動化基礎設施部署
- **Helm**：自動化 Kubernetes 應用部署

### Week 49-52：SRE 實踐

- **SLI/SLO/SLA**：定義服務品質目標
- **Error Budget**：管理可靠性與速度的平衡
- **On-Call Runbook**：標準化處理流程
- **Postmortem**：事後分析與改進

## 實戰：自動化新服務的可觀測性

### 需求

當部署一個新服務時，自動完成：
1. Prometheus 自動發現服務
2. Grafana 自動建立 Dashboard
3. AlertManager 自動加入告警規則
4. Jaeger 自動開始追蹤

### 步驟 1：服務加上標籤

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
    observability: enabled  # 這個標籤會觸發自動化
spec:
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
```

### 步驟 2：Prometheus 自動發現

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # 只抓取有 prometheus.io/scrape=true 的 Pod
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # 使用 Pod 的 prometheus.io/port 註解作為 Port
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
```

### 步驟 3：Grafana 自動建立 Dashboard

**使用 Grafana Operator**：

```yaml
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDashboard
metadata:
  name: order-service-dashboard
spec:
  json: |
    {
      "title": "Order Service Dashboard",
      "panels": [
        {
          "title": "Request Rate",
          "targets": [
            {"expr": "rate(http_requests_total{service=\"order-service\"}[5m])"}
          ]
        },
        {
          "title": "Error Rate",
          "targets": [
            {"expr": "rate(http_requests_total{service=\"order-service\",status=\"500\"}[5m]) / rate(http_requests_total{service=\"order-service\"}[5m])"}
          ]
        },
        {
          "title": "P95 Latency",
          "targets": [
            {"expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service=\"order-service\"}[5m]))"}
          ]
        }
      ]
    }
```

### 步驟 4：AlertManager 自動加入告警規則

**使用 PrometheusRule CRD**：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
spec:
  groups:
    - name: order-service
      rules:
        - alert: HighErrorRate
          expr: rate(http_requests_total{service="order-service",status="500"}[5m]) / rate(http_requests_total{service="order-service"}[5m]) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate in order-service"
        
        - alert: HighLatency
          expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service="order-service"}[5m])) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High latency in order-service"
```

### 步驟 5：Jaeger 自動追蹤

不需要額外配置，只要應用程式使用 OpenTelemetry，Jaeger 會自動接收 Traces。

## 自動化的收益

### 時間節省

| 任務                      | 手動 | 自動化 | 節省 |
|---------------------------|------|--------|------|
| 部署新服務的可觀測性      | 30 分鐘 | 0 分鐘 | 30 分鐘 |
| 修改告警規則              | 1 小時 | 5 分鐘 | 55 分鐘 |
| 擴展到新環境              | 2 小時 | 10 分鐘 | 1.8 小時 |

如果你有 100 個服務，節省的時間 = **50 小時 × 100 = 5,000 小時**。

### 一致性

所有服務都使用相同的配置，減少人為錯誤。

### 可靠性

自動化可以確保每次部署都是正確的。

## 接下來的 12 週

接下來的 12 週，我們會深入學習：

1. **Service Discovery**：如何自動發現服務
2. **Synthetic Monitoring**：如何主動監控服務
3. **Infrastructure as Code**：如何用程式碼管理基礎設施
4. **SRE 實踐**：如何建立可靠的系統

---

**自動化不只是節省時間，更是提升可靠性和一致性的關鍵。**

當你能把可觀測性完全自動化，你就能專注在更重要的事情上：改進系統的可靠性和效能。
