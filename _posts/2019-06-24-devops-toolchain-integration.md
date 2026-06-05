---
layout: post
title: "DevOps 工具鏈整合"
date: 2019-06-24 10:00:00 +0800
categories: [DevOps, Integration]
tags: [Jenkins, CI/CD, DevOps, Automation]
---

上週研究了多雲架構（參考 [多雲架構設計](/posts/2019/06/17/multi-cloud-architecture/)），這週來談 DevOps 工具鏈整合。

我們的問題：工具太多，不連貫。

一個功能從需求到上線的流程：
1. PM 在 Jira 建立 ticket
2. 開發在 GitLab 寫程式碼
3. Jenkins 建置
4. SonarQube 掃描
5. Docker image 推到 Harbor
6. ArgoCD 部署到 Kubernetes
7. Prometheus 監控
8. Grafana 可視化
9. Slack 通知

**問題**：這些工具彼此獨立，需要手動在多個系統切換，效率低。

這週目標：**串起所有工具，實現自動化工作流**。

> 工具：GitLab、Jenkins、Jira、Slack、PagerDuty、Grafana

## 工具鏈架構

```
Jira (需求管理)
  ↓
GitLab (程式碼管理)
  ↓ webhook
Jenkins (CI/CD)
  ├→ SonarQube (程式碼品質)
  ├→ Dependency-Check (安全掃描)
  ├→ Harbor (容器 registry)
  └→ ArgoCD (部署)
      ↓
      Kubernetes (運行環境)
      ↓
      Prometheus (監控)
      ↓
      Grafana (可視化)
      ↓
      Alertmanager (告警)
      ↓
      PagerDuty / Slack (通知)
      ↓
      Jira (自動建立 incident ticket)
```

目標：全流程自動化，開發只需專注寫程式碼。

## Jira 整合

### GitLab ↔ Jira

**需求**：Git commit 自動關聯 Jira ticket。

GitLab 配置：
```bash
# Settings → Integrations → Jira
Jira URL: https://mycompany.atlassian.net
Username: gitlab-bot@example.com
Password: <API_TOKEN>
```

Commit message 格式：
```bash
git commit -m "PROJ-123: Add order validation logic"
```

`PROJ-123` 是 Jira ticket ID。

效果：
- Jira ticket 自動顯示相關 commits
- Commit 點擊可連到 GitLab
- Ticket 狀態自動更新

Commit message 關鍵字：
```bash
# 開始工作
git commit -m "PROJ-123: Start implementing feature"
→ Jira ticket 狀態：TODO → In Progress

# 完成工作
git commit -m "PROJ-123: Fix bug #done"
→ Jira ticket 狀態：In Progress → Done

# 關聯多個 tickets
git commit -m "PROJ-123 PROJ-124: Refactor payment logic"
```

### Jenkins ↔ Jira

**需求**：部署成功後，自動更新 Jira。

Jenkins Pipeline：
```groovy
pipeline {
    agent any
    
    environment {
        JIRA_SITE = 'mycompany.atlassian.net'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/'
                
                // 更新 Jira
                script {
                    def issueKey = env.GIT_COMMIT_MESSAGE.find(/PROJ-\d+/)
                    if (issueKey) {
                        jiraAddComment(
                            site: JIRA_SITE,
                            idOrKey: issueKey,
                            comment: "部署成功！\nBuild: ${BUILD_URL}\nCommit: ${GIT_COMMIT}"
                        )
                        
                        jiraTransitionIssue(
                            site: JIRA_SITE,
                            idOrKey: issueKey,
                            input: [
                                transition: [name: 'Deploy to Production']
                            ]
                        )
                    }
                }
            }
        }
    }
}
```

效果：
- 部署成功 → Jira 自動新增評論
- Ticket 狀態：Testing → Deployed

## GitLab CI/CD 進階整合

### 完整 Pipeline

`.gitlab-ci.yml`：
```yaml
variables:
  DOCKER_IMAGE: registry.example.com/order-service
  SONAR_HOST: https://sonar.example.com
  
stages:
  - build
  - test
  - security-scan
  - package
  - deploy
  - notify

build:
  stage: build
  image: maven:3.6-jdk-11
  script:
    - mvn clean compile
  artifacts:
    paths:
      - target/
    expire_in: 1 hour

unit-test:
  stage: test
  image: maven:3.6-jdk-11
  script:
    - mvn test
    - mvn jacoco:report
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
      cobertura: target/site/jacoco/jacoco.xml
  coverage: '/Total.*?([0-9]{1,3})%/'

integration-test:
  stage: test
  image: maven:3.6-jdk-11
  services:
    - postgres:11
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
  script:
    - mvn verify -P integration-test

sonarqube-scan:
  stage: security-scan
  image: maven:3.6-jdk-11
  script:
    - mvn sonar:sonar
      -Dsonar.host.url=$SONAR_HOST
      -Dsonar.login=$SONAR_TOKEN
      -Dsonar.projectKey=$CI_PROJECT_NAME
      -Dsonar.projectName=$CI_PROJECT_NAME
  allow_failure: false  # Quality gate 失敗就停止

dependency-check:
  stage: security-scan
  image: owasp/dependency-check:latest
  script:
    - /usr/share/dependency-check/bin/dependency-check.sh
      --project $CI_PROJECT_NAME
      --scan .
      --format HTML
      --out ./reports
      --failOnCVSS 7
  artifacts:
    paths:
      - reports/
    when: always

docker-build:
  stage: package
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker tag $DOCKER_IMAGE:$CI_COMMIT_SHA $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
    - docker push $DOCKER_IMAGE:latest
    
trivy-scan:
  stage: security-scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $DOCKER_IMAGE:$CI_COMMIT_SHA
  dependencies:
    - docker-build

deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server=$K8S_SERVER
    - kubectl config set-credentials gitlab --token=$K8S_TOKEN
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default
    - kubectl set image deployment/order-service order-service=$DOCKER_IMAGE:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/order-service -n staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/order-service order-service=$DOCKER_IMAGE:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/order-service -n production
  environment:
    name: production
    url: https://api.example.com
  when: manual  # 手動觸發
  only:
    - master

notify-slack:
  stage: notify
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST $SLACK_WEBHOOK_URL \
        -H 'Content-Type: application/json' \
        -d '{
          "text": "部署成功！",
          "attachments": [{
            "color": "good",
            "fields": [
              {"title": "專案", "value": "'"$CI_PROJECT_NAME"'", "short": true},
              {"title": "環境", "value": "Production", "short": true},
              {"title": "Commit", "value": "'"$CI_COMMIT_SHORT_SHA"'", "short": true},
              {"title": "作者", "value": "'"$GITLAB_USER_NAME"'", "short": true}
            ]
          }]
        }'
  only:
    - master
  when: on_success

notify-failure:
  stage: notify
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST $SLACK_WEBHOOK_URL \
        -H 'Content-Type: application/json' \
        -d '{
          "text": "🚨 部署失敗！",
          "attachments": [{
            "color": "danger",
            "fields": [
              {"title": "專案", "value": "'"$CI_PROJECT_NAME"'"},
              {"title": "Pipeline", "value": "'"$CI_PIPELINE_URL"'"},
              {"title": "失敗階段", "value": "'"$CI_JOB_STAGE"'"}
            ]
          }]
        }'
  when: on_failure
```

效果：
1. Commit → 自動建置
2. 測試失敗 → 停止並通知
3. 程式碼品質不達標 → 停止
4. 安全掃描發現漏洞 → 停止
5. 全部通過 → 部署到 Staging
6. 手動確認 → 部署到 Production
7. Slack 通知結果

## 監控告警整合

### Prometheus → Alertmanager → PagerDuty/Slack

**Prometheus 告警規則**：

`alerts.yml`：
```yaml
groups:
- name: order-service
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{job="order-service",status=~"5.."}[5m])) /
      sum(rate(http_requests_total{job="order-service"}[5m])) > 0.05
    for: 5m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "Order Service error rate too high"
      description: "Error rate is {{ $value | humanizePercentage }}"
      dashboard: "https://grafana.example.com/d/order-service"
      
  - alert: HighLatency
    expr: |
      histogram_quantile(0.95,
        rate(http_request_duration_seconds_bucket{job="order-service"}[5m])
      ) > 1
    for: 10m
    labels:
      severity: warning
      team: backend
    annotations:
      summary: "Order Service latency high"
      description: "P95 latency is {{ $value }}s"
      
  - alert: PodCrashLooping
    expr: |
      rate(kube_pod_container_status_restarts_total{namespace="production",pod=~"order-service.*"}[15m]) > 0
    for: 5m
    labels:
      severity: critical
      team: sre
    annotations:
      summary: "Pod {{ $labels.pod }} is crash looping"
      description: "Pod has restarted {{ $value }} times in the last 15 minutes"
```

**Alertmanager 配置**：

`alertmanager.yml`：
```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  
  routes:
  # Critical 告警 → PagerDuty（叫人起來處理）
  - match:
      severity: critical
    receiver: 'pagerduty-critical'
    continue: true
  
  # Critical 告警也發到 Slack
  - match:
      severity: critical
    receiver: 'slack-critical'
  
  # Warning 告警 → Slack
  - match:
      severity: warning
    receiver: 'slack-warning'

receivers:
- name: 'default'
  slack_configs:
  - channel: '#alerts'
    text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'

- name: 'pagerduty-critical'
  pagerduty_configs:
  - service_key: '<PAGERDUTY_SERVICE_KEY>'
    description: '{{ .GroupLabels.alertname }}'
    details:
      firing: '{{ .Alerts.Firing | len }}'
      resolved: '{{ .Alerts.Resolved | len }}'
      dashboard: '{{ range .Alerts }}{{ .Annotations.dashboard }}{{ end }}'

- name: 'slack-critical'
  slack_configs:
  - channel: '#incidents'
    title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
    text: |
      {{ range .Alerts }}
      *Summary*: {{ .Annotations.summary }}
      *Description*: {{ .Annotations.description }}
      *Dashboard*: {{ .Annotations.dashboard }}
      {{ end }}
    color: danger

- name: 'slack-warning'
  slack_configs:
  - channel: '#monitoring'
    title: '⚠️ WARNING: {{ .GroupLabels.alertname }}'
    text: |
      {{ range .Alerts }}
      {{ .Annotations.summary }}
      {{ end }}
    color: warning
```

### PagerDuty → Jira

**需求**：Critical 事件自動建立 Jira incident ticket。

PagerDuty Webhook：
```json
{
  "webhook": {
    "endpoint_url": "https://api.example.com/pagerduty-webhook",
    "type": "incident.triggered",
    "filter": {
      "type": "incident_triggered"
    }
  }
}
```

Webhook Handler：
```python
from flask import Flask, request
from jira import JIRA

app = Flask(__name__)
jira = JIRA('https://mycompany.atlassian.net', basic_auth=('bot@example.com', 'API_TOKEN'))

@app.route('/pagerduty-webhook', methods=['POST'])
def pagerduty_webhook():
    data = request.json
    
    if data['event'] == 'incident.triggered':
        incident = data['incident']
        
        # 建立 Jira incident ticket
        issue = jira.create_issue(
            project='OPS',
            summary=f"[P1] {incident['title']}",
            description=f"""
PagerDuty Incident: {incident['html_url']}
Service: {incident['service']['name']}
Urgency: {incident['urgency']}
Status: {incident['status']}

Details:
{incident['body']['details']}
            """,
            issuetype={'name': 'Incident'},
            priority={'name': 'Highest'},
            labels=['incident', 'pagerduty']
        )
        
        # 回傳 Jira ticket 到 PagerDuty
        # (在 PagerDuty incident 顯示 Jira 連結)
        return {'jira_issue': issue.key}, 200
    
    return {'status': 'ok'}, 200
```

效果：
1. Prometheus 偵測到高錯誤率
2. Alertmanager 觸發 PagerDuty
3. PagerDuty 叫醒 on-call 工程師
4. 同時自動建立 Jira ticket
5. 工程師處理完畢後關閉 ticket

## Slack 中央控制

### Slack Bot

建立 Slack Bot，在 Slack 執行 DevOps 操作。

**功能**：
- 查詢服務狀態
- 觸發部署
- 查看日誌
- 回滾版本

安裝 Slack App：
```python
from slack_bolt import App
from slack_bolt.adapter.flask import SlackRequestHandler
import subprocess

app = App(token="xoxb-xxx", signing_secret="xxx")

# 查詢狀態
@app.command("/status")
def status_command(ack, command, say):
    ack()
    
    service = command['text']
    result = subprocess.run(
        ['kubectl', 'get', 'pods', '-l', f'app={service}', '-o', 'json'],
        capture_output=True, text=True
    )
    
    # 解析並回傳
    say(f"```{result.stdout}```")

# 部署
@app.command("/deploy")
def deploy_command(ack, command, say):
    ack()
    
    # 解析參數: /deploy order-service v1.2.3 production
    parts = command['text'].split()
    service, version, env = parts[0], parts[1], parts[2]
    
    # 觸發 GitLab Pipeline
    response = requests.post(
        f'https://gitlab.example.com/api/v4/projects/123/trigger/pipeline',
        data={
            'token': 'TRIGGER_TOKEN',
            'ref': 'master',
            'variables[SERVICE]': service,
            'variables[VERSION]': version,
            'variables[ENV]': env
        }
    )
    
    pipeline_url = response.json()['web_url']
    say(f"部署已觸發：{pipeline_url}")

# 查看日誌
@app.command("/logs")
def logs_command(ack, command, say):
    ack()
    
    service = command['text']
    result = subprocess.run(
        ['kubectl', 'logs', '-l', f'app={service}', '--tail=50'],
        capture_output=True, text=True
    )
    
    say(f"```{result.stdout}```")

# 回滾
@app.command("/rollback")
def rollback_command(ack, command, say):
    ack()
    
    service = command['text']
    result = subprocess.run(
        ['kubectl', 'rollout', 'undo', f'deployment/{service}'],
        capture_output=True, text=True
    )
    
    say(f"Rollback completed: {result.stdout}")
```

使用：
```
/status order-service
→ Bot 回傳 Pod 狀態

/deploy order-service v1.2.3 production
→ 觸發部署，回傳 Pipeline URL

/logs order-service
→ 顯示最新 50 行日誌

/rollback order-service
→ 回滾到前一個版本
```

### ChatOps

在 Slack 完成所有操作，不需切換工具。

**Pull Request 通知**：
```yaml
# GitLab webhook → Slack
POST https://hooks.slack.com/services/XXX/YYY/ZZZ
{
  "text": "New MR: <https://gitlab.example.com/group/project/merge_requests/42|!42 Add feature X>",
  "attachments": [{
    "fields": [
      {"title": "Author", "value": "Alice"},
      {"title": "Branch", "value": "feature/add-x"},
      {"title": "Status", "value": "Open"}
    ],
    "actions": [
      {"type": "button", "text": "Approve", "url": "..."},
      {"type": "button", "text": "View Diff", "url": "..."}
    ]
  }]
}
```

點擊 button 直接 approve MR。

**部署通知（互動式）**：
```python
@app.action("approve_deploy")
def approve_deploy(ack, body, say):
    ack()
    
    # 使用者點擊 "Approve"
    pipeline_id = body['actions'][0]['value']
    
    # 觸發部署
    trigger_pipeline(pipeline_id)
    
    say(f"Deployment approved by <@{body['user']['id']}>")
```

## Dashboard 整合

### Grafana 統一面板

整合所有資料來源到單一 Dashboard。

**資料來源**：
- Prometheus（指標）
- Loki（日誌）
- Jaeger（追蹤）
- PostgreSQL（業務資料）

**Dashboard 範例**：

```json
{
  "dashboard": {
    "title": "Order Service Overview",
    "panels": [
      {
        "title": "RPS",
        "targets": [{
          "datasource": "Prometheus",
          "expr": "sum(rate(http_requests_total{job=\"order-service\"}[5m]))"
        }]
      },
      {
        "title": "Error Rate",
        "targets": [{
          "datasource": "Prometheus",
          "expr": "sum(rate(http_requests_total{job=\"order-service\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{job=\"order-service\"}[5m])) * 100"
        }],
        "alert": {
          "conditions": [{
            "evaluator": {"params": [5], "type": "gt"},
            "operator": {"type": "and"},
            "query": {"datasourceId": 1, "model": {...}},
            "reducer": {"params": [], "type": "avg"},
            "type": "query"
          }],
          "executionErrorState": "alerting",
          "frequency": "1m",
          "handler": 1,
          "name": "High Error Rate Alert",
          "noDataState": "no_data",
          "notifications": [{"uid": "slack-channel"}]
        }
      },
      {
        "title": "Recent Logs",
        "targets": [{
          "datasource": "Loki",
          "expr": "{job=\"order-service\"} |= \"ERROR\""
        }]
      },
      {
        "title": "Slow Traces",
        "targets": [{
          "datasource": "Jaeger",
          "query": "service=order-service AND duration>1s"
        }]
      },
      {
        "title": "Orders Today",
        "targets": [{
          "datasource": "PostgreSQL",
          "rawSql": "SELECT COUNT(*) FROM orders WHERE DATE(created_at) = CURRENT_DATE"
        }]
      }
    ]
  }
}
```

單一畫面看到：
- 即時指標（RPS、延遲、錯誤率）
- 最新錯誤日誌
- 慢請求追蹤
- 業務數據（訂單數）

## 心得

工具鏈整合讓我們的效率大幅提升：

**以前**：
- 開發寫完程式碼 → 手動通知 QA → QA 測試 → 手動通知 Ops → Ops 部署 → 出問題再手動通知開發
- 耗時：1-2 天

**現在**：
- 開發 push 程式碼 → CI/CD 自動建置測試部署 → Slack 通知結果 → 出問題自動 rollback 並告警
- 耗時：10-15 分鐘

關鍵是：
1. **自動化**：減少人工操作
2. **通知**：重要事件即時知道
3. **可視化**：所有資訊在 Dashboard
4. **追蹤**：從需求到部署全程可追蹤

建議：
- 先整合最常用的工具（Git、CI/CD、Slack）
- 逐步加入其他工具
- 文件化工作流程
- 定期檢視和優化

工具鏈整合是個持續改進的過程，目標是讓團隊更專注於創造價值，而不是手動操作工具。

下週是這個系列的最後一篇，來總結這半年來的 DevOps 學習心得。
