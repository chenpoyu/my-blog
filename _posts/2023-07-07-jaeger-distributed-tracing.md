---
layout: post
title: "Jaeger 實戰：建立分散式追蹤系統"
date: 2023-07-07 09:00:00 +0800
categories: [Observability, Tracing]
tags: [Jaeger, Distributed Tracing, OpenTelemetry, Performance Analysis]
---

上週我們理解了 Trace 和 Span 的概念，今天我們來實際部署一套完整的追蹤系統。

**Jaeger** 是 Uber 開源的分散式追蹤平台，也是 CNCF 的畢業專案，與 Kubernetes、Prometheus 同等級。

## Jaeger 架構

```
應用程式 → Jaeger Agent → Jaeger Collector → Storage (Cassandra/Elasticsearch) → Jaeger Query → Jaeger UI
```

### 元件說明

| 元件 | 作用 | 部署位置 |
|------|------|----------|
| **Jaeger Agent** | 接收 Traces，批次傳送到 Collector | 每個主機一個（sidecar 或 daemonset） |
| **Jaeger Collector** | 驗證、處理、儲存 Traces | 集中部署（可水平擴展） |
| **Storage** | 儲存 Traces | Cassandra / Elasticsearch / Kafka |
| **Jaeger Query** | 查詢 API | 集中部署 |
| **Jaeger UI** | Web 介面 | 集中部署 |

## 快速開始：All-in-One

最簡單的方式是用 **All-in-One** 映像（包含所有元件）。

```bash
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

**Port 說明**：
- `16686`: Jaeger UI
- `4317`: OTLP gRPC（推薦）
- `4318`: OTLP HTTP

存取 Jaeger UI：`http://localhost:16686`

## 生產環境部署：Kubernetes

### 1. 使用 Helm 安裝

```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm install jaeger jaegertracing/jaeger \
  --set collector.replicaCount=3 \
  --set query.replicaCount=2 \
  --set storage.type=elasticsearch \
  --set storage.elasticsearch.host=elasticsearch.default.svc.cluster.local
```

### 2. 使用 Jaeger Operator

```bash
kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.45.0/jaeger-operator.yaml -n observability
```

建立 Jaeger 實例：

```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-prod
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        index-prefix: jaeger
  collector:
    replicas: 3
    resources:
      limits:
        cpu: 1000m
        memory: 2Gi
  query:
    replicas: 2
    resources:
      limits:
        cpu: 500m
        memory: 1Gi
  ingress:
    enabled: true
    hosts:
      - jaeger.example.com
```

## 儲存後端選擇

### Cassandra（推薦用於大規模）

**優點**：
- 高可用性
- 水平擴展
- 適合寫入密集型工作負載

**缺點**：
- 運維複雜
- 資源消耗大

**配置**：

```yaml
storage:
  type: cassandra
  options:
    cassandra:
      servers: cassandra-0.cassandra.default.svc.cluster.local,cassandra-1.cassandra.default.svc.cluster.local
      keyspace: jaeger_v1_prod
      replication-factor: 3
```

### Elasticsearch（推薦用於中小規模）

**優點**：
- 如果你已經有 ELK Stack，可以共用
- 強大的查詢能力
- 容易運維

**缺點**：
- 寫入效能不如 Cassandra

**配置**：

```yaml
storage:
  type: elasticsearch
  options:
    es:
      server-urls: http://elasticsearch:9200
      index-prefix: jaeger
      username: elastic
      password: changeme
```

### Kafka（用於高吞吐量緩衝）

如果你的 Traces 量非常大，可以在 Collector 和 Storage 之間加入 Kafka。

```yaml
storage:
  type: kafka
  options:
    kafka:
      brokers: kafka-0:9092,kafka-1:9092,kafka-2:9092
      topic: jaeger-spans
```

然後用 **Jaeger Ingester** 從 Kafka 讀取並寫入最終儲存。

## Java 應用程式整合

### 方法 1：使用 OpenTelemetry（推薦）

上週我們已經介紹過，這裡快速回顧：

```yaml
# application.yml
otel:
  service:
    name: order-service
  traces:
    exporter: otlp
  exporter:
    otlp:
      endpoint: http://jaeger-collector:4317
```

### 方法 2：使用 Jaeger Java Client

如果你不想用 OpenTelemetry，也可以直接用 Jaeger SDK。

```xml
<dependency>
    <groupId>io.jaegertracing</groupId>
    <artifactId>jaeger-client</artifactId>
    <version>1.8.1</version>
</dependency>
```

```java
import io.jaegertracing.Configuration;
import io.opentracing.Tracer;

@Bean
public Tracer jaegerTracer() {
    Configuration config = Configuration.fromEnv("order-service");
    return config.getTracer();
}
```

環境變數：

```bash
JAEGER_AGENT_HOST=localhost
JAEGER_AGENT_PORT=6831
JAEGER_SAMPLER_TYPE=probabilistic
JAEGER_SAMPLER_PARAM=0.1  # 取樣 10%
```

## .NET Core 應用程式整合

```csharp
using OpenTelemetry.Exporter;
using OpenTelemetry.Trace;

builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProviderBuilder =>
        tracerProviderBuilder
            .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("order-service"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri("http://jaeger-collector:4317");
                options.Protocol = OtlpExportProtocol.Grpc;
            }));
```

## Jaeger UI 使用指南

### 1. 搜尋 Traces

**基本搜尋**：
- **Service**: 選擇服務（如 `order-service`）
- **Operation**: 選擇操作（如 `GET /api/orders`）
- **Tags**: 加入過濾條件（如 `http.status_code=500`）
- **Lookback**: 時間範圍（如 Last 1 hour）
- **Min/Max Duration**: 過濾執行時間

**進階搜尋**：

```
http.status_code=500 error=true user.id=12345
```

### 2. Trace Timeline View

點選任一個 Trace，會看到：

```
Timeline View:
├── order-service: GET /api/orders (523ms) ████████████████████
│   ├── validate_user (45ms) ███
│   ├── inventory-service: GET /api/inventory (234ms) ███████████
│   │   └── database: SELECT (189ms) ████████
│   ├── payment-service: POST /api/payment (198ms) ████████
│   │   └── stripe-api: charge (156ms) ██████
│   └── database: INSERT (23ms) █
```

**顏色說明**：
- 綠色：正常
- 黃色：警告（較慢）
- 紅色：錯誤

### 3. Span Details

點選任一個 Span，可以看到：

**Tags（標籤）**：
```
http.method: GET
http.url: /api/orders/123
http.status_code: 200
user.id: john@example.com
order.amount: 1234.56
```

**Process（程序資訊）**：
```
service.name: order-service
service.version: 1.2.3
hostname: pod-abc123
ip: 10.0.1.5
```

**Logs（事件）**：
```
10:15:23.123 - Validating user
10:15:23.234 - User validated successfully
10:15:23.345 - Checking inventory
10:15:23.567 - Inventory available
```

### 4. 比較 Traces

選擇多個 Trace，點選 **Compare**，可以看到：

```
Trace 1 (523ms):
├── validate_user (45ms)
├── inventory-service (234ms)
├── payment-service (198ms)
└── database (23ms)

Trace 2 (1234ms):  ← 慢很多！
├── validate_user (43ms)
├── inventory-service (987ms)  ← 瓶頸在這裡
├── payment-service (189ms)
└── database (15ms)
```

### 5. 依賴關係圖（Dependency Graph）

在 Jaeger UI 中選擇 **System Architecture**，可以看到服務之間的依賴關係：

```
API Gateway → Order Service → Inventory Service
           ↓                 ↓
           Payment Service   Database
           ↓
           Stripe API
```

**指標**：
- 每條線的粗細代表呼叫頻率
- 顏色代表錯誤率

## 效能分析實戰

### 案例：API 回應時間從 200ms 暴增到 2s

#### 步驟 1：找出慢的 Traces

在 Jaeger UI：
- **Service**: `order-service`
- **Operation**: `GET /api/orders`
- **Min Duration**: `2s`
- **Lookback**: Last 1 hour

找到 100 個慢請求。

#### 步驟 2：分析 Timeline

點選其中一個 Trace，發現：

```
order-service: GET /api/orders (2.1s)
├── validate_user (45ms)
├── inventory-service: GET /api/inventory (1.9s)  ← 問題在這裡
│   └── database: SELECT (1.85s)  ← 慢查詢
└── database: INSERT (23ms)
```

#### 步驟 3：檢查 Database Span

點選 `database: SELECT`，看到：

```
Tags:
db.statement: SELECT * FROM inventory WHERE product_id IN (1,2,3,...,100)
db.rows_returned: 100
```

#### 步驟 4：找到根因

原來是 **N+1 查詢問題**：

```java
// 問題程式碼
List<Order> orders = orderRepository.findAll();  // 1 次查詢
for (Order order : orders) {
    Inventory inventory = inventoryRepository.findByProductId(order.getProductId());  // N 次查詢
}
```

#### 步驟 5：修正

```java
// 修正後
List<Long> productIds = orders.stream().map(Order::getProductId).collect(Collectors.toList());
Map<Long, Inventory> inventories = inventoryRepository.findByProductIdIn(productIds);  // 1 次查詢
```

#### 結果

回應時間從 2s 降到 200ms。

## 告警設定

Jaeger 本身不提供告警功能，但你可以用 **Jaeger Metrics** + **Prometheus** + **Alertmanager**。

### 1. 啟用 Jaeger Metrics

```yaml
spec:
  query:
    metricsBackend: prometheus
```

### 2. Prometheus 抓取 Metrics

```yaml
scrape_configs:
  - job_name: 'jaeger'
    static_configs:
      - targets: ['jaeger-query:16687']
```

### 3. 設定告警規則

```yaml
groups:
  - name: jaeger-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(jaeger_traces_error_total[5m]) > 0.05
        for: 5m
        annotations:
          summary: "High error rate in traces"
      
      - alert: SlowTraces
        expr: histogram_quantile(0.95, rate(jaeger_traces_duration_seconds_bucket[5m])) > 5
        for: 5m
        annotations:
          summary: "P95 latency > 5s"
```

## 成本優化

### 1. 調整 Sampling Rate

```yaml
sampler:
  type: probabilistic
  param: 0.01  # 只追蹤 1%（從 10% 降到 1%）
```

**成本降低**：90%

### 2. 設定 Retention

```yaml
storage:
  options:
    es:
      max-span-age: 168h  # 保留 7 天（從 30 天降到 7 天）
```

### 3. 使用 Adaptive Sampling

Jaeger 支援**自適應取樣**：自動調整取樣率。

```yaml
sampling:
  strategies-file: /etc/jaeger/strategies.json
```

`strategies.json`：

```json
{
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.01
  },
  "per_operation_strategies": [
    {
      "operation": "GET /health",
      "type": "probabilistic",
      "param": 0  // 不追蹤健康檢查
    },
    {
      "operation": "POST /api/orders",
      "type": "probabilistic",
      "param": 1  // 追蹤所有訂單請求
    }
  ]
}
```

---

**Jaeger 讓你「看見」分散式系統的行為，而不是「猜測」。**

當你能精確定位每個環節的執行時間，優化就不再是碰運氣。
