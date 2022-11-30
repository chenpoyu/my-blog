---
layout: post
title: "Prometheus 和 Grafana 監控 Kubernetes"
date: 2019-01-21 10:00:00 +0800
categories: [DevOps, Monitoring]
tags: [Prometheus, Grafana, Kubernetes, Monitoring, Metrics]
---

上週建立了 Kubernetes Ingress（參考 [Kubernetes Ingress 路由和負載平衡](/posts/2019/01/14/kubernetes-ingress-routing/)），整個應用都跑在 K8s 上了。但遇到一個問題：不知道系統運行狀況如何。

上個禮拜五下午，使用者反應「網站很慢」。我登入主機看，CPU 使用率 95%，但不知道是哪個 Pod 造成的，也不知道什麼時候開始變慢的。花了一個小時才找到問題。

這週建立監控系統，使用 Prometheus 收集指標，Grafana 視覺化呈現。

> 使用版本：Prometheus 2.6、Grafana 5.4

## 為什麼需要監控

沒有監控就像盲人開車：

1. **無法及時發現問題**：系統掛了才知道
2. **不知道歷史趨勢**：為什麼上週五很慢？不知道
3. **無法容量規劃**：什麼時候需要擴充？不知道
4. **除錯困難**：錯誤發生時的狀態無法重現

需要監控的指標：
- **系統層**：CPU、記憶體、磁碟、網路
- **容器層**：Pod 數量、重啟次數、資源使用
- **應用層**：API 回應時間、錯誤率、QPS
- **業務層**：訂單數、使用者數、交易金額

## Prometheus 是什麼

Prometheus 是開源的監控和告警系統，由 SoundCloud 開發，現在是 CNCF 畢業專案。

### 特色

1. **多維度資料模型**：時間序列由 metric name 和 labels 識別
2. **強大的查詢語言**：PromQL
3. **Pull 模式**：Prometheus 主動去抓取指標
4. **不依賴分散式儲存**：單機運行
5. **豐富的生態系**：各種 exporter

### 架構

```
[應用程式] → 暴露 /metrics endpoint
     ↓
[Prometheus Server] → 定期抓取指標並儲存
     ↓
[Grafana] → 查詢 Prometheus 並視覺化
     ↓
[使用者] → 看儀表板
```

## 安裝 Prometheus

### 使用 Helm 安裝

Helm 是 Kubernetes 的套件管理工具。

安裝 Helm：
```bash
# macOS
brew install kubernetes-helm

# 初始化
helm init
```

安裝 Prometheus：
```bash
# 加入 stable repo
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

# 安裝 Prometheus Operator
helm install stable/prometheus-operator --name prometheus --namespace monitoring
```

Prometheus Operator 會自動部署：
- Prometheus Server
- Grafana
- Alertmanager
- Node Exporter（收集 Node 指標）
- Kube State Metrics（收集 K8s 資源指標）

查看 Pod：
```bash
kubectl get pods -n monitoring

# NAME                                                  READY   STATUS
# prometheus-prometheus-oper-operator-xxxxx             1/1     Running
# prometheus-prometheus-oper-prometheus-0               3/3     Running
# prometheus-grafana-xxxxx                              2/2     Running
# prometheus-kube-state-metrics-xxxxx                   1/1     Running
# prometheus-prometheus-node-exporter-xxxxx             1/1     Running
```

### 訪問 Prometheus

```bash
kubectl port-forward -n monitoring svc/prometheus-prometheus-oper-prometheus 9090:9090
```

打開瀏覽器：`http://localhost:9090`

### 訪問 Grafana

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

打開瀏覽器：`http://localhost:3000`

預設帳號：
- Username: `admin`
- Password: `prom-operator`（可能不同，查看文件）

## Prometheus 基本操作

### 查詢介面

在 Prometheus UI 的「Graph」頁面可以執行 PromQL 查詢。

### 簡單查詢

查看所有 Node 的 CPU 使用率：
```promql
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

查看所有 Pod 的記憶體使用：
```promql
container_memory_usage_bytes{namespace="default"}
```

查看 API Server 請求總數：
```promql
apiserver_request_total
```

### PromQL 基礎

#### Metric Types

1. **Counter**：只增不減的計數器（例如：請求總數）
2. **Gauge**：可增可減的數值（例如：當前記憶體使用）
3. **Histogram**：統計分佈（例如：請求延遲分佈）
4. **Summary**：類似 Histogram，但計算在客戶端

#### Selectors

```promql
# 選擇特定 metric
http_requests_total

# 加上 label 篩選
http_requests_total{job="api-server"}

# 多個條件
http_requests_total{job="api-server", method="GET"}

# 正則表達式
http_requests_total{status=~"5.."}  # 5xx 錯誤
```

#### Functions

```promql
# 速率（每秒增長）
rate(http_requests_total[5m])

# 瞬時速率
irate(http_requests_total[5m])

# 總和
sum(http_requests_total)

# 平均
avg(node_memory_usage_bytes)

# 最大值
max(http_request_duration_seconds)

# 分位數
histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

## 監控 Kubernetes 叢集

Prometheus Operator 已經預先設定好監控 K8s 叢集。

### Node 指標

```promql
# CPU 使用率
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 記憶體使用率
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 磁碟使用率
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100

# 網路流量
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

### Pod 指標

```promql
# Pod 數量
count(kube_pod_info)

# Pod 重啟次數
kube_pod_container_status_restarts_total

# Pod CPU 使用
rate(container_cpu_usage_seconds_total{namespace="default"}[5m])

# Pod 記憶體使用
container_memory_usage_bytes{namespace="default"}
```

### API Server 指標

```promql
# API 請求率
rate(apiserver_request_total[5m])

# API 請求延遲
histogram_quantile(0.95, rate(apiserver_request_duration_seconds_bucket[5m]))
```

## 監控 Spring Boot 應用

Spring Boot 有 Actuator，可以暴露監控指標給 Prometheus。

### 加入依賴

`pom.xml`：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### 設定

`application.properties`：
```properties
# 開啟 Prometheus endpoint
management.endpoints.web.exposure.include=health,info,prometheus
management.metrics.export.prometheus.enabled=true

# 加入自訂 tags
management.metrics.tags.application=${spring.application.name}
management.metrics.tags.environment=${ENVIRONMENT:dev}
```

### 驗證

啟動應用後訪問：
```bash
curl http://localhost:8080/actuator/prometheus

# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
# jvm_memory_used_bytes{area="heap",id="PS Eden Space",} 1.23456789E8
# ...
```

### 讓 Prometheus 抓取

建立 ServiceMonitor：

`servicemonitor.yaml`：
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-shop-api
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-shop-api
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
```

部署：
```bash
kubectl apply -f servicemonitor.yaml
```

確保你的 Service 有正確的 label：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-shop-api-service
  labels:
    app: my-shop-api
spec:
  selector:
    app: my-shop-api
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

### 查詢應用指標

在 Prometheus 查詢：

```promql
# HTTP 請求總數
http_server_requests_seconds_count{application="my-shop-api"}

# 平均回應時間
rate(http_server_requests_seconds_sum{application="my-shop-api"}[5m]) 
/ 
rate(http_server_requests_seconds_count{application="my-shop-api"}[5m])

# 95th 百分位回應時間
histogram_quantile(0.95, 
  rate(http_server_requests_seconds_bucket{application="my-shop-api"}[5m])
)

# 錯誤率
rate(http_server_requests_seconds_count{application="my-shop-api", status=~"5.."}[5m])

# JVM 記憶體使用
jvm_memory_used_bytes{application="my-shop-api", area="heap"}
```

## Grafana 儀表板

Prometheus 的查詢介面很陽春，Grafana 提供專業的視覺化。

### 登入 Grafana

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

訪問 `http://localhost:3000`，使用 `admin` / `prom-operator` 登入。

### 匯入預設儀表板

Grafana 社群有很多現成的儀表板。

1. 點左側「+」→「Import」
2. 輸入儀表板 ID

推薦的儀表板：
- **Kubernetes Cluster Monitoring (Dashboard ID: 7249)**：叢集總覽
- **Kubernetes Capacity Planning (Dashboard ID: 5228)**：容量規劃
- **Node Exporter Full (Dashboard ID: 1860)**：Node 詳細指標
- **Spring Boot Statistics (Dashboard ID: 6756)**：Spring Boot 應用

3. 選擇 Prometheus 資料來源
4. 點「Import」

### 建立自訂儀表板

#### 新增儀表板

1. 點「+」→「Dashboard」
2. 點「Add Query」

#### CPU 使用率面板

- Query: 
  ```promql
  100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  ```
- Visualization: Graph
- Legend: `{{instance}}`

#### API 請求率面板

- Query:
  ```promql
  sum(rate(http_server_requests_seconds_count{application="my-shop-api"}[5m])) by (uri)
  ```
- Visualization: Graph
- Legend: `{{uri}}`

#### 錯誤率面板

- Query:
  ```promql
  sum(rate(http_server_requests_seconds_count{application="my-shop-api", status=~"5.."}[5m])) 
  / 
  sum(rate(http_server_requests_seconds_count{application="my-shop-api"}[5m])) * 100
  ```
- Visualization: Singlestat
- Unit: percent (0-100)
- Thresholds: 1, 5（黃色和紅色警告）

#### JVM 記憶體面板

- Query:
  ```promql
  jvm_memory_used_bytes{application="my-shop-api"}
  ```
- Visualization: Graph
- Legend: `{{area}} - {{id}}`
- Unit: bytes (IEC)

### 設定變數

讓儀表板更靈活，可以切換不同環境或應用。

1. 儀表板設定 → Variables → Add variable
2. Name: `app`
3. Type: Query
4. Query: `label_values(http_server_requests_seconds_count, application)`

現在儀表板上方會有下拉選單，可以切換不同應用。

在 Query 中使用：
```promql
http_server_requests_seconds_count{application="$app"}
```

## 告警設定

監控不只是看儀表板，還要主動通知。

### Alertmanager

Prometheus Operator 已經安裝了 Alertmanager。

查看：
```bash
kubectl get svc -n monitoring | grep alertmanager
```

### 建立告警規則

`prometheus-rule.yaml`：
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-shop-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
  - name: my-shop-api
    interval: 30s
    rules:
    # API 錯誤率過高
    - alert: HighErrorRate
      expr: |
        sum(rate(http_server_requests_seconds_count{application="my-shop-api", status=~"5.."}[5m])) 
        / 
        sum(rate(http_server_requests_seconds_count{application="my-shop-api"}[5m])) > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "API 錯誤率過高"
        description: "{{ $labels.application }} 錯誤率 {{ $value | humanizePercentage }}"
    
    # API 回應時間過長
    - alert: HighLatency
      expr: |
        histogram_quantile(0.95, 
          rate(http_server_requests_seconds_bucket{application="my-shop-api"}[5m])
        ) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "API 回應時間過長"
        description: "{{ $labels.application }} 95th 百分位回應時間 {{ $value }}s"
    
    # Pod 不斷重啟
    - alert: PodRestarting
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace="default"}[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod 不斷重啟"
        description: "{{ $labels.pod }} 在過去 15 分鐘重啟了 {{ $value }} 次"
    
    # Node 記憶體不足
    - alert: NodeMemoryPressure
      expr: |
        (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node 記憶體不足"
        description: "{{ $labels.instance }} 記憶體使用率 {{ $value }}%"
```

部署：
```bash
kubectl apply -f prometheus-rule.yaml
```

在 Prometheus UI 的「Alerts」頁面可以看到告警規則。

### 設定通知

Alertmanager 支援多種通知方式：Email、Slack、PagerDuty、Webhook。

這裡設定 Slack 通知。

建立 Slack Incoming Webhook：
1. 到 Slack App Directory
2. 搜尋「Incoming WebHooks」
3. 選擇頻道（例如 `#alerts`）
4. 複製 Webhook URL

設定 Alertmanager：

`alertmanager-config.yaml`：
```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  group_by: ['alertname', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'slack'
  
receivers:
- name: 'slack'
  slack_configs:
  - channel: '#alerts'
    title: '{{ .GroupLabels.alertname }}'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    send_resolved: true
```

更新 Alertmanager：
```bash
kubectl create secret generic alertmanager-prometheus-prometheus-oper-alertmanager \
  --from-file=alertmanager.yaml=alertmanager-config.yaml \
  --dry-run -o yaml | kubectl apply -n monitoring -f -
```

## 遇到的問題

### 問題一：ServiceMonitor 無法抓取指標

Prometheus targets 頁面顯示「DOWN」。

原因：
1. Service 的 label 不符合 ServiceMonitor 的 selector
2. endpoint 的 port name 不對
3. 應用沒有真的暴露 `/metrics`

解決方法：檢查 label 和 port，確認應用正常運行。

### 問題二：Grafana 顯示「No data」

Prometheus 有資料，但 Grafana 查不到。

原因：時間範圍設定錯誤，或 query 語法錯誤。

解決方法：先在 Prometheus UI 測試 query，確認有資料後再到 Grafana。

### 問題三：告警風暴

某個告警不斷觸發，Slack 被淹沒。

解決方法：
1. 調整告警閾值（太敏感）
2. 增加 `for` 時間（持續一段時間才告警）
3. 設定 `repeat_interval`（降低重複通知頻率）

## 心得

有了監控系統後，對系統運行狀況更有信心了。現在每天早上第一件事就是打開 Grafana 儀表板，看看昨晚有沒有異常。

上週又遇到「網站很慢」的問題，這次馬上打開 Grafana，發現是某個 API 的 95th 回應時間暴增。點進去看 Pod 日誌，發現是資料庫連線池用完了。10 分鐘就找到問題，調整連線池大小後解決。

下週要研究 ELK Stack（Elasticsearch + Logstash + Kibana），集中管理所有服務的日誌。現在要看日誌還要 `kubectl logs` 一個一個 Pod 查，太麻煩了。
