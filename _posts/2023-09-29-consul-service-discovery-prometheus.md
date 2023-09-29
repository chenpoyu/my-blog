---
layout: post
title: "Consul 服務發現與 Prometheus 整合"
date: 2023-09-29 09:00:00 +0800
categories: [Observability, Service Discovery]
tags: [Consul, Service Discovery, Prometheus, Automation]
---

當你的微服務數量達到 50+，手動配置 Prometheus 的 Scrape Targets 會變成噩夢。

今天我們來談談如何用 **Consul** 實現服務自動發現。

## Consul 簡介

Consul 是 HashiCorp 推出的服務網格解決方案，主要功能：
- **服務註冊與發現**
- **健康檢查**
- **Key/Value 存儲**
- **Multi-Datacenter 支援**

## 安裝 Consul

### Docker Compose

```yaml
version: '3'
services:
  consul:
    image: consul:1.16
    command: agent -server -ui -bootstrap-expect=1 -client=0.0.0.0
    ports:
      - "8500:8500"  # HTTP API
      - "8600:8600/udp"  # DNS
    environment:
      - CONSUL_BIND_INTERFACE=eth0
```

### Kubernetes (Helm)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install consul hashicorp/consul \
  --set global.name=consul \
  --set ui.enabled=true \
  --set connectInject.enabled=true
```

## 服務註冊到 Consul

### 方法 1：手動註冊

```bash
curl -X PUT http://consul:8500/v1/agent/service/register \
  -d '{
  "ID": "order-service-1",
  "Name": "order-service",
  "Address": "10.0.1.5",
  "Port": 8080,
  "Tags": ["production", "v1.2.3"],
  "Meta": {
    "metrics_path": "/metrics",
    "version": "1.2.3"
  },
  "Check": {
    "HTTP": "http://10.0.1.5:8080/health",
    "Interval": "10s",
    "Timeout": "5s"
  }
}'
```

### 方法 2：用 Consul Client 自動註冊

**Java + Spring Boot**：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: order-service
  cloud:
    consul:
      host: consul
      port: 8500
      discovery:
        enabled: true
        health-check-path: /actuator/health
        health-check-interval: 10s
        tags:
          - production
          - v1.2.3
        metadata:
          metrics_path: /actuator/prometheus
```

**應用程式啟動時**，會自動註冊到 Consul。

**.NET + Consul**：

```csharp
using Consul;

var consulClient = new ConsulClient(config =>
{
    config.Address = new Uri("http://consul:8500");
});

var registration = new AgentServiceRegistration
{
    ID = "order-service-1",
    Name = "order-service",
    Address = "10.0.1.5",
    Port = 8080,
    Tags = new[] { "production", "v1.2.3" },
    Meta = new Dictionary<string, string>
    {
        { "metrics_path", "/metrics" },
        { "version", "1.2.3" }
    },
    Check = new AgentServiceCheck
    {
        HTTP = "http://10.0.1.5:8080/health",
        Interval = TimeSpan.FromSeconds(10),
        Timeout = TimeSpan.FromSeconds(5)
    }
};

await consulClient.Agent.ServiceRegister(registration);
```

### 方法 3：Kubernetes + Consul Connect Inject

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        consul.hashicorp.com/connect-inject: "true"  # 自動註冊
        consul.hashicorp.com/service-meta-metrics_path: "/metrics"
    spec:
      containers:
      - name: order-service
        image: order-service:1.2.3
```

## Prometheus + Consul 整合

### 配置 Prometheus

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'consul:8500'
        services: []  # 空的話，會發現所有服務
    
    relabel_configs:
      # 使用 Consul 的 Meta 中的 metrics_path
      - source_labels: [__meta_consul_service_metadata_metrics_path]
        target_label: __metrics_path__
        regex: (.+)
        replacement: $1
      
      # 使用服務名稱作為 job 標籤
      - source_labels: [__meta_consul_service]
        target_label: job
      
      # 使用服務的 Tags 作為標籤
      - source_labels: [__meta_consul_tags]
        regex: ,(?:[^,]+,)*(production|staging|development),.*
        target_label: env
        replacement: $1
      
      # 過濾：只抓取有 "metrics" Tag 的服務
      - source_labels: [__meta_consul_tags]
        regex: .*,metrics,.*
        action: keep
```

### 標籤說明

Consul Service Discovery 會產生這些標籤：

| 標籤                                  | 說明                     | 範例                  |
|---------------------------------------|--------------------------|----------------------|
| `__meta_consul_address`               | Consul Agent 的地址      | `10.0.1.1:8500`      |
| `__meta_consul_node`                  | 服務所在的節點           | `node-1`             |
| `__meta_consul_service`               | 服務名稱                 | `order-service`      |
| `__meta_consul_service_address`       | 服務的 IP                | `10.0.1.5`           |
| `__meta_consul_service_port`          | 服務的 Port              | `8080`               |
| `__meta_consul_service_id`            | 服務的 ID                | `order-service-1`    |
| `__meta_consul_tags`                  | 服務的 Tags              | `,production,v1.2.3,`|
| `__meta_consul_service_metadata_<key>`| 服務的 Meta（自訂）      | `metrics_path=/metrics` |

## 實戰：動態服務發現

### 場景

你有 3 個 `order-service` 實例：

```
order-service-1: 10.0.1.5:8080
order-service-2: 10.0.1.6:8080
order-service-3: 10.0.1.7:8080
```

### 註冊到 Consul

```bash
for i in {1..3}; do
  curl -X PUT http://consul:8500/v1/agent/service/register -d '{
    "ID": "order-service-'$i'",
    "Name": "order-service",
    "Address": "10.0.1.'$((4+i))'",
    "Port": 8080,
    "Tags": ["production", "metrics"],
    "Meta": {
      "metrics_path": "/metrics"
    }
  }'
done
```

### Prometheus 自動發現

在 Prometheus 的 Targets 頁面（http://prometheus:9090/targets），你會看到：

```
job="order-service"
  10.0.1.5:8080 (order-service-1) UP
  10.0.1.6:8080 (order-service-2) UP
  10.0.1.7:8080 (order-service-3) UP
```

### 新增實例

當你部署第 4 個實例：

```bash
curl -X PUT http://consul:8500/v1/agent/service/register -d '{
  "ID": "order-service-4",
  "Name": "order-service",
  "Address": "10.0.1.8",
  "Port": 8080,
  "Tags": ["production", "metrics"]
}'
```

Prometheus 會**自動**發現新實例（無需重啟）。

### 移除實例

```bash
curl -X PUT http://consul:8500/v1/agent/service/deregister/order-service-4
```

Prometheus 會**自動**移除這個實例。

## Consul 健康檢查

### HTTP 健康檢查

```json
{
  "Check": {
    "HTTP": "http://10.0.1.5:8080/health",
    "Interval": "10s",
    "Timeout": "5s",
    "DeregisterCriticalServiceAfter": "1m"
  }
}
```

### TCP 健康檢查

```json
{
  "Check": {
    "TCP": "10.0.1.5:8080",
    "Interval": "10s",
    "Timeout": "5s"
  }
}
```

### Script 健康檢查

```json
{
  "Check": {
    "Script": "/usr/local/bin/check_order_service.sh",
    "Interval": "10s"
  }
}
```

### Prometheus 只抓取健康的實例

Prometheus 的 Consul Service Discovery 會自動過濾掉不健康的實例。

## Multi-Datacenter

### 場景

你有兩個資料中心：
- `dc1`：北京
- `dc2`：上海

### 配置 Consul

```yaml
# dc1
consul agent -server -datacenter=dc1 -ui

# dc2
consul agent -server -datacenter=dc2 -ui -join=<dc1-ip>
```

### Prometheus 發現多個資料中心的服務

```yaml
scrape_configs:
  - job_name: 'consul-dc1'
    consul_sd_configs:
      - server: 'consul-dc1:8500'
        datacenter: 'dc1'
  
  - job_name: 'consul-dc2'
    consul_sd_configs:
      - server: 'consul-dc2:8500'
        datacenter: 'dc2'
```

## Grafana + Consul

### 自動建立 Dashboard

當新服務註冊到 Consul 時，自動建立 Grafana Dashboard。

**實作**：

1. 監聽 Consul 的事件
2. 當有新服務註冊時，呼叫 Grafana API 建立 Dashboard

```python
import consul
import requests
import json

c = consul.Consul(host='consul', port=8500)

# 監聽服務變更
index = None
while True:
    index, services = c.health.service('order-service', index=index)
    
    for service in services:
        service_id = service['Service']['ID']
        service_name = service['Service']['Service']
        
        # 檢查是否已經有 Dashboard
        response = requests.get(f'http://grafana:3000/api/search?query={service_name}')
        if len(response.json()) == 0:
            # 建立 Dashboard
            {% raw %}
            dashboard = {
                'dashboard': {
                    'title': f'{service_name} Dashboard',
                    'panels': [
                        {
                            'title': 'Request Rate',
                            'targets': [{'expr': f'rate(http_requests_total{{service="{service_name}"}}[5m])'}]
                        }
                    ]
                }
            }
            {% endraw %}
            requests.post('http://grafana:3000/api/dashboards/db', json=dashboard)
```

## Consul Template

Consul Template 可以監聽 Consul 的變更，自動更新配置檔案。

### 範例：動態更新 Nginx Upstream

**template.ctmpl**：

```nginx
upstream backend {
{{- range service "order-service" }}
  server {{ .Address }}:{{ .Port }};
{{- end }}
}

server {
  listen 80;
  location / {
    proxy_pass http://backend;
  }
}
```

**執行 Consul Template**：

```bash
consul-template \
  -consul-addr=consul:8500 \
  -template="template.ctmpl:/etc/nginx/nginx.conf:nginx -s reload"
```

當 Consul 中的服務變更時，Nginx 的配置會**自動更新**並重新載入。

## 實戰建議

### 1. 在服務中加入 Graceful Shutdown

當服務關閉時，先從 Consul 註銷，再關閉服務。

```java
@PreDestroy
public void deregister() {
    consulClient.agentServiceDeregister(serviceId);
    Thread.sleep(5000);  // 等待 Prometheus 移除這個實例
}
```

### 2. 使用 Meta 傳遞自訂資訊

```json
{
  "Meta": {
    "metrics_path": "/actuator/prometheus",
    "version": "1.2.3",
    "team": "order-team"
  }
}
```

在 Prometheus 中，可以用這些 Meta 作為標籤。

### 3. 定期檢查 Consul 的健康狀態

```bash
curl http://consul:8500/v1/health/state/critical
```

---

**Service Discovery 是可觀測性自動化的基礎。**

當你的服務能自動註冊到 Consul，Prometheus 就能自動發現並監控它們，不需要任何手動配置。
