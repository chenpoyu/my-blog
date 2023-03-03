---
layout: post
title: "Exporter 生態系：讓萬物皆可監控"
date: 2023-03-03 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Prometheus, Exporter, Integration, Monitoring]
---

前面幾週我們學會了監控應用程式和系統資源，但現實世界遠比這複雜：

- 資料庫（MySQL、PostgreSQL、Redis）
- 訊息佇列（Kafka、RabbitMQ）
- 負載均衡器（Nginx、HAProxy）
- ...

難道每個都要自己寫程式碼嗎？**不用，這就是 Exporter 的價值。**

## 什麼是 Exporter？

Exporter 是一個「翻譯官」，它把各種系統的指標轉換成 Prometheus 格式。

```
┌──────────┐
│  MySQL   │
└─────┬────┘
      │ (MySQL Protocol)
      ↓
┌──────────────┐
│MySQL Exporter│ ← 翻譯官
└─────┬────────┘
      │ (Prometheus Format)
      ↓
┌──────────────┐
│ Prometheus   │
└──────────────┘
```

## 必裝的 Exporter

### 1. Node Exporter（系統資源）

> **安裝說明**：我們在《Week 2: Prometheus 入門》已經介紹過 Node Exporter 的基本安裝。這裡展示更完整的配置方式。

**Docker 安裝（完整版）**：

```bash
docker run -d \
  --name node-exporter \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  prom/node-exporter:latest \
  --path.rootfs=/host
```

**收集的指標**：
- CPU、記憶體、磁碟、網路
- 檔案系統、Load Average
- 網路連線狀態、TCP/UDP 統計
- 系統啟動時間、執行緒數

**實用查詢**：

```promql
# 磁碟預測何時會滿
predict_linear(node_filesystem_free_bytes{mountpoint="/"}[1h], 24*3600) < 0

# 網路流量
rate(node_network_receive_bytes_total[5m]) * 8 / 1024 / 1024  # Mbps

# 系統運行時間
(time() - node_boot_time_seconds) / 86400  # 天數
```

### 2. MySQL Exporter（資料庫）

```bash
docker run -d \
  --name mysql-exporter \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="exporter:password@(mysql:3306)/" \
  prom/mysqld-exporter:latest
```

**收集的指標**：
- 查詢數、慢查詢數
- 連線數、執行緒數
- InnoDB Buffer Pool 使用率
- Replication 延遲

**關鍵告警**：

```yaml
- alert: MySQLDown
  expr: mysql_up == 0
  for: 1m

- alert: MySQLTooManyConnections
  expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
  for: 5m

- alert: MySQLSlowQueries
  expr: rate(mysql_global_status_slow_queries[5m]) > 10
  for: 5m

- alert: MySQLReplicationLag
  expr: mysql_slave_status_seconds_behind_master > 30
  for: 5m
```

### 3. Redis Exporter（快取）

```bash
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  oliver006/redis_exporter:latest \
  --redis.addr=redis://redis:6379
```

**關鍵指標**：

```promql
# 記憶體使用率
redis_memory_used_bytes / redis_memory_max_bytes * 100

# 命中率
rate(redis_keyspace_hits_total[5m]) / 
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) * 100

# 連線數
redis_connected_clients

# 過期 key 數量
rate(redis_expired_keys_total[5m])
```

### 4. PostgreSQL Exporter

```bash
docker run -d \
  --name postgres-exporter \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://postgres:password@postgres:5432/postgres?sslmode=disable" \
  prometheuscommunity/postgres-exporter:latest
```

**獨特指標**：

```promql
# 連線池使用率
pg_stat_database_numbackends / pg_settings_max_connections * 100

# 資料庫大小
pg_database_size_bytes

# 慢查詢
pg_stat_statements_mean_exec_time_seconds > 1

# 死鎖次數
rate(pg_stat_database_deadlocks[5m])
```

### 5. Nginx Exporter（Web 伺服器）

首先要在 Nginx 啟用 stub_status：

```nginx
server {
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}
```

啟動 Exporter：

```bash
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:latest \
  -nginx.scrape-uri=http://nginx:80/nginx_status
```

**關鍵指標**：

```promql
# 每秒請求數
rate(nginx_http_requests_total[5m])

# 活躍連線數
nginx_connections_active

# 連線等待數
nginx_connections_waiting
```

### 6. Blackbox Exporter（外部監控）

這是最特別的 Exporter，它可以監控「外部服務」的可用性。

```bash
docker run -d \
  --name blackbox-exporter \
  -p 9115:9115 \
  -v $(pwd)/blackbox.yml:/etc/blackbox_exporter/config.yml \
  prom/blackbox-exporter:latest
```

建立 `blackbox.yml`：

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET
  
  tcp_connect:
    prober: tcp
    timeout: 5s
  
  icmp:
    prober: icmp
    timeout: 5s
```

在 `prometheus.yml` 加入：

```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://www.google.com
          - https://your-api.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

**實用查詢**：

```promql
# 服務是否可達
probe_success == 0

# SSL 憑證何時過期
(probe_ssl_earliest_cert_expiry - time()) / 86400  # 剩餘天數

# HTTP 回應時間
probe_http_duration_seconds

# DNS 查詢時間
probe_dns_lookup_time_seconds
```

**告警範例**：

```yaml
- alert: SiteDown
  expr: probe_success == 0
  for: 2m
  annotations:
    summary: "{{ $labels.instance }} 無法連線"

- alert: SSLCertExpiringSoon
  expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
  for: 1h
  annotations:
    summary: "{{ $labels.instance }} SSL 憑證將在 {{ $value }} 天後過期"

- alert: SlowResponse
  expr: probe_http_duration_seconds > 5
  for: 5m
  annotations:
    summary: "{{ $labels.instance }} 回應時間 > 5s"
```

## JMX Exporter（Java 應用程式）

如果你的應用程式是 Java，可以用 JMX Exporter 暴露 JVM 指標。

下載 JMX Exporter JAR：

```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar
```

啟動應用程式時加入參數：

```bash
java -javaagent:./jmx_prometheus_javaagent-0.19.0.jar=8080:config.yaml -jar your-app.jar
```

`config.yaml`：

```yaml
rules:
  - pattern: ".*"
```

訪問 `http://localhost:8080/metrics`，你會看到所有 JMX MBeans 的指標。

**關鍵指標**：

```promql
# Heap 記憶體使用率
jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"} * 100

# GC 暫停時間
rate(jvm_gc_collection_seconds_sum[5m])

# 執行緒數量
jvm_threads_current

# Class 載入數量
jvm_classes_loaded
```

## 自定義 Exporter：當沒有現成的怎麼辦？

假設你有一個內部系統，沒有現成的 Exporter，怎麼辦？

### 方法 1：用 Python 寫一個簡單的 Exporter

```python
from prometheus_client import start_http_server, Gauge
import time
import requests

# 定義指標
active_users = Gauge('myapp_active_users', 'Number of active users')
pending_orders = Gauge('myapp_pending_orders', 'Number of pending orders')

def collect_metrics():
    # 從你的 API 或資料庫取得數據
    response = requests.get('http://your-app/api/stats')
    data = response.json()
    
    active_users.set(data['active_users'])
    pending_orders.set(data['pending_orders'])

if __name__ == '__main__':
    start_http_server(8000)  # 在 port 8000 暴露 /metrics
    
    while True:
        collect_metrics()
        time.sleep(15)  # 每 15 秒更新一次
```

### 方法 2：用 Pushgateway（短期任務）

有些任務是「短期執行」的（例如 Cron Job、Batch Job），無法被 Prometheus 抓取。

這時用 Pushgateway：

```bash
docker run -d \
  --name pushgateway \
  -p 9091:9091 \
  prom/pushgateway:latest
```

在你的 Batch Job 中：

```python
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

registry = CollectorRegistry()
job_duration = Gauge('batch_job_duration_seconds', 'Job duration', registry=registry)

# 執行任務
start = time.time()
do_batch_job()
duration = time.time() - start

job_duration.set(duration)

# 推送到 Pushgateway
push_to_gateway('pushgateway:9091', job='batch_import', registry=registry)
```

在 `prometheus.yml` 加入：

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['pushgateway:9091']
```

## Exporter 的效能考量

Exporter 本身也會消耗資源，要注意：

### 1. 不要設定太短的 scrape_interval

錯誤：
```yaml
scrape_configs:
  - job_name: 'mysql'
    scrape_interval: 1s  # 太頻繁！
```

正確：
```yaml
scrape_configs:
  - job_name: 'mysql'
    scrape_interval: 15s  # 合理
```

### 2. 過濾不需要的指標

有些 Exporter 會暴露幾百個指標，但你可能只需要幾十個。

用 `metric_relabel_configs` 過濾：

```yaml
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']
    metric_relabel_configs:
      # 只保留我需要的指標
      - source_labels: [__name__]
        regex: 'mysql_global_status_.*|mysql_global_variables_.*|mysql_up'
        action: keep
```

### 3. 監控 Exporter 本身

```promql
# Exporter 是否正常
up{job="mysql-exporter"}

# Scrape 失敗次數
rate(prometheus_target_scrapes_exceeded_sample_limit_total[5m])

# Scrape 延遲
prometheus_target_interval_length_seconds
```

---

**Prometheus 生態系的強大，在於「萬物皆可監控」。**

當你能監控從應用程式到資料庫、從 SSL 憑證到外部 API，你就掌握了整個系統的健康狀態。
