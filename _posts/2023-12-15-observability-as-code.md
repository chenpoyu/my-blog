---
layout: post
title: "Observability as Code：用程式碼管理你的監控"
date: 2023-12-15 09:00:00 +0800
categories: [Observability, Infrastructure as Code]
tags: [Observability as Code, IaC, Terraform, Automation, GitOps]
---

「為什麼 Production 的 Dashboard 跟 Staging 不一樣？」

「誰改了這個 Alert Rule？」

「我們可以還原到上週的監控設定嗎？」

**Observability as Code (OaC)** 的核心理念：

**把所有監控設定都用程式碼管理，就像管理應用程式一樣。**

## 為什麼需要 Observability as Code？

### 傳統方式的問題

**手動設定 Dashboard**
```
1. 登入 Grafana
2. 點選「Create Dashboard」
3. 手動加入 Panel
4. 調整查詢語法
5. 儲存

問題：
❌ 無法追蹤變更歷史
❌ 無法在環境間複製
❌ 難以協作
❌ 容易出錯
```

**手動設定 Alert Rule**
```
1. 登入 Prometheus
2. 編輯 prometheus.yml
3. 重新載入設定
4. 測試 Alert

問題：
❌ 設定散落各處
❌ 無法版本控制
❌ 難以 Code Review
```

### Observability as Code 的好處

✅ **版本控制**：所有變更都可追蹤
✅ **可重現**：可以在任何環境重建
✅ **Code Review**：透過 PR 審查變更
✅ **自動化**：CI/CD 自動部署
✅ **一致性**：所有環境使用相同設定

## Observability as Code 架構

```
┌─────────────────────────────────────────┐
│          Git Repository                  │
│  ┌────────────────────────────────────┐ │
│  │  observability/                    │ │
│  │    ├── dashboards/                 │ │
│  │    │   ├── app-dashboard.json      │ │
│  │    │   └── infra-dashboard.json    │ │
│  │    ├── alerts/                     │ │
│  │    │   ├── app-alerts.yml          │ │
│  │    │   └── infra-alerts.yml        │ │
│  │    ├── terraform/                  │ │
│  │    │   ├── main.tf                 │ │
│  │    │   └── variables.tf            │ │
│  │    └── scripts/                    │ │
│  │        └── deploy.sh               │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
              │
              │ git push
              ▼
┌─────────────────────────────────────────┐
│          CI/CD Pipeline                  │
│  1. Validate                            │
│  2. Test                                │
│  3. Deploy to Staging                   │
│  4. Deploy to Production                │
└─────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│      Observability Stack                │
│  ┌──────────┐  ┌──────────┐            │
│  │Prometheus│  │ Grafana  │            │
│  └──────────┘  └──────────┘            │
└─────────────────────────────────────────┘
```

## 實作 1：Grafana Dashboard as Code

### 使用 Terraform

```hcl
# terraform/grafana.tf
provider "grafana" {
  url  = var.grafana_url
  auth = var.grafana_api_key
}

# Dashboard
resource "grafana_dashboard" "app_dashboard" {
  config_json = file("${path.module}/../dashboards/app-dashboard.json")
  folder      = grafana_folder.observability.id
}

# Folder
resource "grafana_folder" "observability" {
  title = "Observability"
}

# Data Source
resource "grafana_data_source" "prometheus" {
  type = "prometheus"
  name = "Prometheus"
  url  = var.prometheus_url
  
  json_data {
    http_method = "POST"
    timeout     = 60
  }
}

# Alert Notification Channel
resource "grafana_notification_channel" "slack" {
  name = "Slack"
  type = "slack"
  
  settings = {
    url                 = var.slack_webhook_url
    recipient           = "#alerts"
    username            = "Grafana"
    mentionChannel      = "here"
    disableResolveMessage = false
  }
}
```

### Dashboard JSON

```json
{
  "dashboard": {
    "title": "Application Dashboard",
    "tags": ["application", "production"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "gridPos": {
          "x": 0,
          "y": 0,
          "w": 12,
          "h": 8
        }
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "gridPos": {
          "x": 12,
          "y": 0,
          "w": 12,
          "h": 8
        }
      }
    ]
  }
}
```

### 使用 Grafonnet (Jsonnet)

```jsonnet
// dashboards/app-dashboard.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local prometheus = grafana.prometheus;
local graphPanel = grafana.graphPanel;

dashboard.new(
  'Application Dashboard',
  tags=['application', 'production'],
  editable=true,
  time_from='now-6h',
)
.addRow(
  row.new(title='HTTP Metrics')
  .addPanel(
    graphPanel.new(
      'Request Rate',
      datasource='Prometheus',
      format='reqps',
    )
    .addTarget(
      prometheus.target(
        'sum(rate(http_requests_total[5m])) by (service)',
        legendFormat='{{service}}',
      )
    )
  )
  .addPanel(
    graphPanel.new(
      'Error Rate',
      datasource='Prometheus',
      format='percent',
    )
    .addTarget(
      prometheus.target(
        'sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)',
        legendFormat='{{service}}',
      )
    )
  )
)
```

## 實作 2：Prometheus Alert Rules as Code

### 使用 YAML

{% raw %}
```yaml
# alerts/app-alerts.yml
groups:
  - name: application
    interval: 30s
    rules:
      # High Error Rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "{{ $labels.service }} has error rate of {{ $value | humanizePercentage }}"
          runbook_url: "https://wiki.example.com/runbook/high-error-rate"
          dashboard_url: "https://grafana.example.com/d/app-dashboard"

      # High Latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          ) > 1
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High latency on {{ $labels.service }}"
          description: "{{ $labels.service }} P95 latency is {{ $value }}s"

      # Service Down
      - alert: ServiceDown
        expr: up{job="application"} == 0
        for: 1m
        labels:
          severity: critical
          team: sre
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been down for 1 minute"
```
{% endraw %}

### 使用 Terraform

{% raw %}
```hcl
# terraform/prometheus-alerts.tf
resource "prometheus_rule_group" "application" {
  name     = "application"
  interval = "30s"

  rule {
    alert = "HighErrorRate"
    expr  = <<-EOF
      sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
      / sum(rate(http_requests_total[5m])) by (service) > 0.05
    EOF
    for   = "5m"
    
    labels = {
      severity = "critical"
      team     = "backend"
    }
    
    annotations = {
      summary     = "High error rate on {{ $labels.service }}"
      description = "{{ $labels.service }} has error rate of {{ $value | humanizePercentage }}"
    }
  }
}
```
{% endraw %}

## 實作 3：Log Pipeline as Code

### Fluentd Configuration

```ruby
# fluentd/fluent.conf.erb
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<filter application.**>
  @type parser
  key_name log
  <parse>
    @type json
  </parse>
</filter>

<filter application.**>
  @type record_transformer
  <record>
    environment <%= ENV['ENVIRONMENT'] %>
    cluster <%= ENV['CLUSTER_NAME'] %>
  </record>
</filter>

<% if ENV['ENABLE_SAMPLING'] == 'true' %>
<filter application.**>
  @type sampling
  <rule>
    level INFO
    sample_rate 10
  </rule>
</filter>
<% end %>

<match application.**>
  @type elasticsearch
  host <%= ENV['ELASTICSEARCH_HOST'] %>
  port 9200
  logstash_format true
  logstash_prefix application
  
  <buffer>
    @type file
    path /var/log/fluentd-buffers/application
    flush_mode interval
    flush_interval 5s
  </buffer>
</match>
```

## 實作 4：完整 Terraform 範例

{% raw %}
```hcl
# terraform/main.tf
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = "~> 1.40"
    }
    prometheus = {
      source = "rlex/prometheus"
      version = "~> 0.3"
    }
  }
}

# Variables
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "grafana_url" {
  description = "Grafana URL"
  type        = string
}

variable "grafana_api_key" {
  description = "Grafana API Key"
  type        = string
  sensitive   = true
}

# Grafana Provider
provider "grafana" {
  url  = var.grafana_url
  auth = var.grafana_api_key
}

# Data Sources
resource "grafana_data_source" "prometheus" {
  type = "prometheus"
  name = "Prometheus-${var.environment}"
  url  = "http://prometheus:9090"
}

resource "grafana_data_source" "loki" {
  type = "loki"
  name = "Loki-${var.environment}"
  url  = "http://loki:3100"
}

# Folders
resource "grafana_folder" "infrastructure" {
  title = "Infrastructure"
}

resource "grafana_folder" "application" {
  title = "Application"
}

# Dashboards
resource "grafana_dashboard" "infra_overview" {
  folder      = grafana_folder.infrastructure.id
  config_json = file("${path.module}/../dashboards/infra-overview.json")
}

resource "grafana_dashboard" "app_overview" {
  folder      = grafana_folder.application.id
  config_json = file("${path.module}/../dashboards/app-overview.json")
}

# Alert Channels
resource "grafana_notification_channel" "slack_critical" {
  name = "Slack-Critical"
  type = "slack"
  
  settings = {
    url       = var.slack_webhook_url
    recipient = "#alerts-critical"
    username  = "Grafana"
  }
}

resource "grafana_notification_channel" "pagerduty" {
  name = "PagerDuty"
  type = "pagerduty"
  
  settings = {
    integrationKey = var.pagerduty_integration_key
    severity       = "critical"
    autoResolve    = true
  }
}

# Alert Rules (使用 Grafana 8+ Unified Alerting)
resource "grafana_rule_group" "application_alerts" {
  name             = "application-alerts"
  folder_uid       = grafana_folder.application.uid
  interval_seconds = 60
  org_id           = 1

  rule {
    name      = "HighErrorRate"
    condition = "C"

    data {
      ref_id = "A"
      
      relative_time_range {
        from = 600
        to   = 0
      }

      datasource_uid = grafana_data_source.prometheus.uid
      model = jsonencode({
        expr         = "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)"
        refId        = "A"
        intervalMs   = 1000
        maxDataPoints = 43200
      })
    }

    data {
      ref_id = "C"
      
      relative_time_range {
        from = 600
        to   = 0
      }

      datasource_uid = "-100"
      model = jsonencode({
        conditions = [
          {
            evaluator = {
              params = [0.05]
              type   = "gt"
            }
            operator = {
              type = "and"
            }
            query = {
              params = ["A"]
            }
            reducer = {
              type = "last"
            }
            type = "query"
          }
        ]
        refId = "C"
        type  = "classic_conditions"
      })
    }

    no_data_state  = "NoData"
    exec_err_state = "Error"
    for            = "5m"

    annotations = {
      summary     = "High error rate on {{ $labels.service }}"
      description = "{{ $labels.service }} has error rate of {{ $value }}"
    }

    labels = {
      severity = "critical"
      team     = "backend"
    }
  }
}
```
{% endraw %}

## CI/CD Pipeline

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - test
  - deploy-staging
  - deploy-production

variables:
  TF_ROOT: terraform

# Validate
validate:
  stage: validate
  image: hashicorp/terraform:latest
  script:
    - cd $TF_ROOT
    - terraform init -backend=false
    - terraform validate
    - terraform fmt -check

# Test Dashboards
test-dashboards:
  stage: test
  image: node:16
  script:
    - npm install -g jsonlint
    - find dashboards -name '*.json' -exec jsonlint {} \;

# Test Alert Rules
test-alerts:
  stage: test
  image: prom/prometheus:latest
  script:
    - promtool check rules alerts/*.yml

# Deploy to Staging
deploy-staging:
  stage: deploy-staging
  image: hashicorp/terraform:latest
  only:
    - develop
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform workspace select staging || terraform workspace new staging
    - terraform plan -var="environment=staging"
    - terraform apply -auto-approve -var="environment=staging"

# Deploy to Production
deploy-production:
  stage: deploy-production
  image: hashicorp/terraform:latest
  only:
    - main
  when: manual
  script:
    - cd $TF_ROOT
    - terraform init
    - terraform workspace select production || terraform workspace new production
    - terraform plan -var="environment=production"
    - terraform apply -auto-approve -var="environment=production"
```

### GitHub Actions

```yaml
# .github/workflows/observability.yml
name: Deploy Observability

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        
      - name: Terraform Validate
        run: |
          cd terraform
          terraform init -backend=false
          terraform validate

      - name: Test Alert Rules
        run: |
          docker run --rm -v $(pwd)/alerts:/alerts prom/prometheus:latest \
            promtool check rules /alerts/*.yml

  deploy-staging:
    needs: validate
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Staging
        env:
          GRAFANA_URL: ${{ secrets.STAGING_GRAFANA_URL }}
          GRAFANA_API_KEY: ${{ secrets.STAGING_GRAFANA_API_KEY }}
        run: |
          cd terraform
          terraform init
          terraform workspace select staging || terraform workspace new staging
          terraform apply -auto-approve \
            -var="environment=staging" \
            -var="grafana_url=$GRAFANA_URL" \
            -var="grafana_api_key=$GRAFANA_API_KEY"

  deploy-production:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Production
        env:
          GRAFANA_URL: ${{ secrets.PROD_GRAFANA_URL }}
          GRAFANA_API_KEY: ${{ secrets.PROD_GRAFANA_API_KEY }}
        run: |
          cd terraform
          terraform init
          terraform workspace select production || terraform workspace new production
          terraform apply -auto-approve \
            -var="environment=production" \
            -var="grafana_url=$GRAFANA_URL" \
            -var="grafana_api_key=$GRAFANA_API_KEY"
```

## 版本控制策略

### Git Workflow

```bash
# 1. 建立新 Branch
git checkout -b feature/add-latency-alert

# 2. 修改 Alert Rule
vim alerts/app-alerts.yml

# 3. 測試
promtool check rules alerts/app-alerts.yml

# 4. Commit
git add alerts/app-alerts.yml
git commit -m "Add latency alert for order service"

# 5. 建立 Pull Request
git push origin feature/add-latency-alert

# 6. Code Review
# Team members review the changes

# 7. Merge
git checkout main
git merge feature/add-latency-alert

# 8. 自動部署
# CI/CD pipeline automatically deploys to production
```

## 最佳實踐

### 1. 模組化

```
observability/
├── modules/
│   ├── dashboards/
│   │   ├── red-metrics/
│   │   │   └── dashboard.json
│   │   └── use-metrics/
│   │       └── dashboard.json
│   ├── alerts/
│   │   ├── availability/
│   │   │   └── rules.yml
│   │   └── performance/
│   │       └── rules.yml
│   └── notification-channels/
│       └── main.tf
```

### 2. 環境變數

```hcl
# terraform/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "alert_thresholds" {
  description = "Alert thresholds per environment"
  type = map(object({
    error_rate    = number
    latency_p95   = number
    cpu_threshold = number
  }))
  default = {
    dev = {
      error_rate    = 0.10
      latency_p95   = 2.0
      cpu_threshold = 90
    }
    staging = {
      error_rate    = 0.05
      latency_p95   = 1.5
      cpu_threshold = 85
    }
    production = {
      error_rate    = 0.01
      latency_p95   = 1.0
      cpu_threshold = 80
    }
  }
}
```

### 3. 測試

```bash
# 測試 Alert Rules
promtool check rules alerts/*.yml

# 測試 Dashboard JSON
jsonlint dashboards/*.json

# 測試 Terraform
terraform validate
terraform plan
```

### 4. 文件化

```markdown
# README.md

## Observability as Code

### 目錄結構
- `dashboards/`: Grafana Dashboards (JSON)
- `alerts/`: Prometheus Alert Rules (YAML)
- `terraform/`: Terraform 設定
- `scripts/`: 部署腳本

### 部署流程
1. 修改設定檔
2. 執行測試：`make test`
3. 建立 PR
4. Code Review
5. Merge 後自動部署

### 環境
- **Staging**: 自動部署 (develop branch)
- **Production**: 手動批准 (main branch)
```

## 總結

Observability as Code 的核心原則：

1. **一切皆程式碼**：Dashboard、Alert、設定都用程式碼管理
2. **版本控制**：所有變更都可追蹤和還原
3. **自動化**：透過 CI/CD 自動部署
4. **測試**：變更前先測試
5. **Code Review**：團隊協作審查

**Observability as Code = 更可靠的監控 + 更快的疊代速度**

---

**相關文章：**
- 《Week 39: Observability Automation 簡介》
- 《Week 43: Terraform 基礎設施自動化》
- 《Week 49: Observability Maturity Model》
