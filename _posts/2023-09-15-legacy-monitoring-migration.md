---
layout: post
title: "從 Legacy Monitoring 遷移到現代可觀測性"
date: 2023-09-15 09:00:00 +0800
categories: [Observability, Migration]
tags: [Migration, Legacy, Modernization, Strategy]
---

很多團隊還在用 10 年前的監控工具：Nagios、Zabbix、ELK 4.x...

今天我們來談談如何**平滑地遷移到現代可觀測性架構**。

## Legacy Monitoring 的問題

### 典型的 Legacy 架構

```
Nagios → Email 告警
Zabbix → 監控 CPU、Memory
ELK 4.x → 集中日誌
自建的 APM → 追蹤效能
```

### 問題

1. **孤島效應**：Logs、Metrics、Traces 分散在不同系統
2. **維護成本高**：需要維護 4-5 套系統
3. **查詢困難**：要在多個系統之間跳轉
4. **缺少關聯**：無法從 Metric 跳到 Trace

## 遷移策略

### 策略 1：Big Bang（不建議）

一次性關閉舊系統，啟用新系統。

**風險**：
- 新系統可能有 bug
- 團隊不熟悉新系統
- 無法回滾

### 策略 2：Strangler Fig Pattern（建議）

逐步遷移，新舊系統並存。

```
第 1 週：新服務使用 OpenTelemetry，舊服務繼續用 Nagios
第 2 週：遷移 20% 的舊服務
第 4 週：遷移 50% 的舊服務
第 8 週：遷移 100% 的舊服務
第 12 週：關閉 Nagios
```

## 階段 1：建立新的可觀測性平台

### 1. 部署 OpenTelemetry Collector

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
spec:
  template:
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.81.0
        volumeMounts:
        - name: config
          mountPath: /etc/otel-collector
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
```

### 2. 部署 Jaeger

```bash
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.47.0/jaeger-operator.yaml

kubectl apply -f - <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
EOF
```

### 3. 升級 ELK 到最新版本

```bash
# 備份現有的 Elasticsearch
elasticdump --input=http://old-elasticsearch:9200 --output=http://new-elasticsearch:9200

# 升級 Kibana
kubectl set image deployment/kibana kibana=docker.elastic.co/kibana/kibana:8.9.0
```

### 4. 部署 Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana \
  --set persistence.enabled=true \
  --set adminPassword=admin
```

## 階段 2：遷移 Metrics

### 從 Zabbix 遷移到 Prometheus

#### 1. 保留 Zabbix，同時啟用 Prometheus

```yaml
# 在每台機器上安裝 node_exporter
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  template:
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        ports:
        - containerPort: 9100
```

#### 2. 對比資料

在 Grafana 中建立 Dashboard，同時顯示 Zabbix 和 Prometheus 的資料：

```json
{
  "panels": [
    {
      "title": "CPU Usage (Zabbix vs Prometheus)",
      "targets": [
        {
          "datasource": "Zabbix",
          "expr": "system.cpu.util"
        },
        {
          "datasource": "Prometheus",
          "expr": "rate(node_cpu_seconds_total{mode=\"idle\"}[5m])"
        }
      ]
    }
  ]
}
```

#### 3. 遷移告警規則

**Zabbix 告警**：

```
CPU > 80% for 5 minutes
```

**Prometheus 告警**：

```yaml
groups:
  - name: host
    rules:
      - alert: HighCPU
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
```

#### 4. 逐步關閉 Zabbix 監控

```
第 1 週：測試環境使用 Prometheus
第 2 週：10% 的生產環境使用 Prometheus
第 4 週：50% 的生產環境使用 Prometheus
第 8 週：100% 的生產環境使用 Prometheus
第 12 週：關閉 Zabbix
```

## 階段 3：遷移 Logs

### 從 ELK 4.x 遷移到 ELK 8.x

#### 1. 建立新的 Elasticsearch 8.x Cluster

```bash
helm install elasticsearch elastic/elasticsearch \
  --version 8.9.0 \
  --set replicas=3 \
  --set volumeClaimTemplate.resources.requests.storage=100Gi
```

#### 2. 雙寫（新舊 Elasticsearch 同時寫入）

**Filebeat 配置**：

```yaml
output.elasticsearch:
  hosts: ["old-elasticsearch:9200", "new-elasticsearch:9200"]
```

或者用 Logstash：

```ruby
output {
  elasticsearch {
    hosts => ["old-elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  elasticsearch {
    hosts => ["new-elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

#### 3. 對比資料

在 Kibana 中，同時查詢新舊 Elasticsearch：

```
Index Pattern 1: logs-old-*
Index Pattern 2: logs-new-*
```

確認資料一致。

#### 4. 切換到新 Elasticsearch

```yaml
output.elasticsearch:
  hosts: ["new-elasticsearch:9200"]  # 只寫入新的
```

#### 5. 遷移歷史資料（可選）

```bash
# 用 Reindex API 遷移歷史資料
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://old-elasticsearch:9200"
    },
    "index": "logs-2023-*"
  },
  "dest": {
    "index": "logs-2023-*"
  }
}
```

## 階段 4：遷移 Tracing

### 從自建 APM 遷移到 OpenTelemetry

#### 1. 識別需要遷移的服務

```bash
# 列出所有服務
kubectl get services

# 識別哪些服務有 APM Agent
kubectl get pods -o json | jq '.items[].spec.containers[].env[] | select(.name == "APM_AGENT")'
```

#### 2. 逐步遷移

**方法 1：雙 Agent（不建議，開銷大）**

```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/opt/apm-agent.jar -javaagent:/opt/opentelemetry-javaagent.jar"
```

**方法 2：Feature Flag 控制**

```java
@Service
public class OrderService {
    @Value("${feature.opentelemetry.enabled:false}")
    private boolean openTelemetryEnabled;
    
    public void createOrder(OrderRequest request) {
        if (openTelemetryEnabled) {
            // 使用 OpenTelemetry
            Span span = tracer.spanBuilder("create-order").startSpan();
            try {
                // ...
            } finally {
                span.end();
            }
        } else {
            // 使用舊的 APM Agent（自動）
            // ...
        }
    }
}
```

**方法 3：按服務遷移（建議）**

```
第 1 週：遷移 1 個低流量服務
第 2 週：遷移 3 個低流量服務
第 4 週：遷移 10 個中流量服務
第 8 週：遷移所有高流量服務
```

#### 3. 對比資料

在 Grafana 中，同時顯示舊 APM 和 Jaeger 的資料：

```json
{
  "panels": [
    {
      "title": "Request Rate (Old APM vs Jaeger)",
      "targets": [
        {
          "datasource": "Old APM",
          "expr": "request_rate"
        },
        {
          "datasource": "Jaeger",
          "expr": "rate(spans_total[5m])"
        }
      ]
    }
  ]
}
```

## 階段 5：遷移告警

### 從 Email 告警遷移到 AlertManager

#### 1. 匯出舊的告警規則

**Nagios 告警**：

```
define service {
  service_description CPU
  check_command check_cpu!80!90
  contact_groups admins
  notification_interval 30
}
```

#### 2. 轉換成 Prometheus 告警

```yaml
groups:
  - name: host
    rules:
      - alert: HighCPU
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
```

#### 3. 配置 AlertManager

```yaml
route:
  receiver: email
  routes:
    - match:
        severity: critical
      receiver: pagerduty
    - match:
        severity: warning
      receiver: slack

receivers:
  - name: email
    email_configs:
      - to: 'admins@example.com'
  
  - name: slack
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#alerts'
  
  - name: pagerduty
    pagerduty_configs:
      - service_key: '...'
```

#### 4. 雙發告警（過渡期）

過渡期間，同時發送到舊系統（Email）和新系統（Slack）：

```yaml
receivers:
  - name: both
    email_configs:
      - to: 'admins@example.com'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#alerts'
```

## 常見陷阱

### 陷阱 1：過早關閉舊系統

**建議**：保留舊系統至少 1 個月，確保新系統穩定。

### 陷阱 2：沒有對比資料

**建議**：在 Grafana 中建立對比 Dashboard，確認新舊系統的資料一致。

### 陷阱 3：忽略歷史資料

**建議**：決定是否需要遷移歷史資料，還是只保留未來的資料。

### 陷阱 4：團隊不熟悉新工具

**建議**：在遷移前，先培訓團隊，並建立文件。

## 遷移檢查清單

### 準備階段
- [ ] 評估現有監控系統的問題
- [ ] 確定遷移的範圍（哪些服務、哪些指標）
- [ ] 建立新的可觀測性平台
- [ ] 培訓團隊

### 遷移階段
- [ ] 選擇一個低風險的服務作為試點
- [ ] 雙寫（新舊系統同時寫入）
- [ ] 對比資料（確認一致性）
- [ ] 逐步遷移其他服務
- [ ] 遷移告警規則

### 收尾階段
- [ ] 關閉舊系統的寫入
- [ ] 保留舊系統 1 個月（唯讀）
- [ ] 完全關閉舊系統
- [ ] 更新文件
- [ ] 回顧遷移過程

## 成本分析

### 舊系統

```
Nagios: 自建，需要 1 個工程師維護
Zabbix: 自建，需要 1 個工程師維護
ELK 4.x: 自建，3 台 VM，需要 1 個工程師維護
自建 APM: 自建，需要 2 個工程師維護

總成本: 5 個工程師 × $100,000/year = $500,000/year
```

### 新系統

```
Prometheus: 自建或託管（Grafana Cloud）
Grafana: 自建或託管
Jaeger: 自建或託管
ELK 8.x: 託管（Elastic Cloud）

總成本: 1 個工程師 × $100,000/year + $50,000 (Elastic Cloud) = $150,000/year
```

**節省**：$350,000/year

## 實戰案例

### 案例：大型電商公司

**背景**：
- 500+ 個微服務
- Nagios + Zabbix + ELK 4.x + 自建 APM

**遷移計劃**：
- 第 1 個月：建立新平台，遷移 10 個服務
- 第 2-3 個月：遷移 100 個服務
- 第 4-6 個月：遷移剩餘服務
- 第 7 個月：關閉舊系統

**結果**：
- 維護成本降低 70%
- MTTR（平均修復時間）降低 50%
- 工程師滿意度提升（不用維護 4 套系統）

---

**遷移不是一蹴而就的，需要仔細規劃和逐步執行。**

但是，一旦完成遷移，你會發現維護成本大幅降低，排查問題的速度大幅提升。
