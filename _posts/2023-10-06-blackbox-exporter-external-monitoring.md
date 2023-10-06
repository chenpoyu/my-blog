---
layout: post
title: "Blackbox Exporter：外部端點監控"
date: 2023-10-06 09:00:00 +0800
categories: [Observability, Synthetic Monitoring]
tags: [Blackbox Exporter, Synthetic Monitoring, Uptime, Prometheus]
---

你的服務可能在內部運作正常，但**使用者能訪問嗎**？

今天我們來談談如何用 **Blackbox Exporter** 監控外部端點的可用性。

## Blackbox Exporter 簡介

Blackbox Exporter 可以從**外部**監控服務，模擬使用者的訪問：
- HTTP/HTTPS 端點監控
- TCP 端點監控
- ICMP (Ping) 監控
- DNS 查詢監控

## 安裝 Blackbox Exporter

### Docker

```bash
docker run -d \
  --name blackbox-exporter \
  -p 9115:9115 \
  -v $(pwd)/blackbox.yml:/config/blackbox.yml \
  prom/blackbox-exporter:v0.24.0 \
  --config.file=/config/blackbox.yml
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: blackbox-exporter
        image: prom/blackbox-exporter:v0.24.0
        ports:
        - containerPort: 9115
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: blackbox-config
---
apiVersion: v1
kind: Service
metadata:
  name: blackbox-exporter
spec:
  ports:
  - port: 9115
  selector:
    app: blackbox-exporter
```

## 配置 Blackbox Exporter

### 基本配置

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      valid_status_codes: [200]
      preferred_ip_protocol: "ip4"
  
  http_post:
    prober: http
    http:
      method: POST
      body: '{"key": "value"}'
      headers:
        Content-Type: application/json
  
  tcp_connect:
    prober: tcp
    timeout: 5s
  
  icmp:
    prober: icmp
    timeout: 5s
  
  dns:
    prober: dns
    dns:
      query_name: "example.com"
      query_type: "A"
```

## Prometheus 整合

### 配置 Prometheus

```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # 使用 http_2xx 模組
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com/health
          - https://blog.example.com
    relabel_configs:
      # 把 target 的 URL 放到 __param_target
      - source_labels: [__address__]
        target_label: __param_target
      
      # 把 blackbox-exporter 的地址放到 __address__
      - target_label: __address__
        replacement: blackbox-exporter:9115
      
      # 把 target 的 URL 放到 instance 標籤
      - source_labels: [__param_target]
        target_label: instance
```

### 查詢

```promql
# 端點是否可用（1=可用，0=不可用）
probe_success{instance="https://example.com"}

# 回應時間（秒）
probe_duration_seconds{instance="https://example.com"}

# HTTP 狀態碼
probe_http_status_code{instance="https://example.com"}

# SSL 證書到期時間（秒）
probe_ssl_earliest_cert_expiry{instance="https://example.com"}
```

## 實戰場景

### 場景 1：監控多個網站的可用性

```yaml
static_configs:
  - targets:
      - https://www.example.com
      - https://api.example.com
      - https://cdn.example.com
    labels:
      env: production
  
  - targets:
      - https://staging.example.com
    labels:
      env: staging
```

### 場景 2：監控 API 回應時間

```promql
# P95 回應時間
histogram_quantile(0.95, 
  sum by (instance, le) (
    rate(probe_http_duration_seconds_bucket[5m])
  )
)

# 平均回應時間
rate(probe_duration_seconds_sum[5m]) / rate(probe_duration_seconds_count[5m])
```

### 場景 3：監控 SSL 證書到期

```promql
# 證書還有幾天到期
(probe_ssl_earliest_cert_expiry - time()) / 86400

# 告警：證書 30 天內到期
probe_ssl_earliest_cert_expiry - time() < 86400 * 30
```

## 進階配置

### HTTP 認證

```yaml
modules:
  http_basic_auth:
    prober: http
    http:
      method: GET
      basic_auth:
        username: admin
        password: secret
```

### 自訂 Headers

```yaml
modules:
  http_custom_headers:
    prober: http
    http:
      headers:
        Authorization: Bearer token123
        X-Custom-Header: value
```

### 驗證回應內容

```yaml
modules:
  http_2xx_with_body:
    prober: http
    http:
      valid_status_codes: [200]
      fail_if_not_matches_regexp:
        - "Welcome"
        - "status.*ok"
```

### 驗證 JSON 回應

```yaml
modules:
  http_json:
    prober: http
    http:
      valid_status_codes: [200]
      fail_if_body_not_matches_regexp:
        - '"status":\s*"healthy"'
```

## Grafana Dashboard

### Panel 1：可用性

```promql
avg_over_time(probe_success{job="blackbox"}[24h]) * 100
```

顯示過去 24 小時的平均可用性（%）。

### Panel 2：回應時間

```promql
probe_duration_seconds{job="blackbox"}
```

### Panel 3：SSL 證書到期時間

```promql
(probe_ssl_earliest_cert_expiry{job="blackbox"} - time()) / 86400
```

### 完整 Dashboard

```json
{
  "dashboard": {
    "title": "Blackbox Exporter Dashboard",
    "panels": [
      {
        "title": "Uptime (24h)",
        "targets": [{
          "expr": "avg_over_time(probe_success[24h]) * 100"
        }],
        "type": "stat",
        "unit": "percent"
      },
      {
        "title": "Response Time",
        "targets": [{
          "expr": "probe_duration_seconds"
        }],
        "type": "graph"
      },
      {
        "title": "SSL Certificate Expiry",
        "targets": [{
          "expr": "(probe_ssl_earliest_cert_expiry - time()) / 86400"
        }],
        "type": "table"
      }
    ]
  }
}
```

## 告警規則

### 告警 1：服務不可用

```yaml
groups:
  - name: blackbox
    rules:
      - alert: ServiceDown
        expr: probe_success{job="blackbox"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
```

### 告警 2：回應時間過慢

```yaml
- alert: SlowResponse
  expr: probe_duration_seconds{job="blackbox"} > 5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Service {{ $labels.instance }} is slow ({{ $value }}s)"
```

### 告警 3：SSL 證書即將到期

```yaml
- alert: SSLCertificateExpiringSoon
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
  labels:
    severity: warning
  annotations:
    summary: "SSL certificate for {{ $labels.instance }} expires in {{ $value }} days"
```

## 多地點監控

### 架構

```
Blackbox Exporter (北京) → 監控所有端點
Blackbox Exporter (上海) → 監控所有端點
Blackbox Exporter (廣州) → 監控所有端點
```

### Prometheus 配置

```yaml
scrape_configs:
  - job_name: 'blackbox-beijing'
    static_configs:
      - targets: [https://example.com]
    relabel_configs:
      - target_label: __address__
        replacement: blackbox-beijing:9115
      - target_label: region
        replacement: beijing
  
  - job_name: 'blackbox-shanghai'
    static_configs:
      - targets: [https://example.com]
    relabel_configs:
      - target_label: __address__
        replacement: blackbox-shanghai:9115
      - target_label: region
        replacement: shanghai
```

### 查詢

```promql
# 各地點的可用性
probe_success{instance="https://example.com"} by (region)

# 各地點的回應時間
probe_duration_seconds{instance="https://example.com"} by (region)
```

## 實戰建議

### 1. 監控關鍵端點

不要監控所有端點，只監控關鍵的：
- 首頁
- API 健康檢查端點
- 登入頁面
- 結帳流程

### 2. 設定合理的 Timeout

```yaml
timeout: 10s  # 不要太短，避免誤報
```

### 3. 使用多地點監控

從多個地點監控，可以發現地區性的問題。

### 4. 整合到 Slack/PagerDuty

```yaml
receivers:
  - name: 'slack'
    slack_configs:
      - channel: '#alerts'
        text: "Service {{ .CommonLabels.instance }} is down!"
```

---

**Blackbox Exporter 可以讓你從使用者的角度監控服務。**

當你的服務在內部運作正常，但外部無法訪問時，Blackbox Exporter 可以第一時間發現問題。
