---
layout: post
title: "Service Mesh 的可觀測性：Istio 與 Linkerd"
date: 2023-08-04 09:00:00 +0800
categories: [Observability, Service Mesh]
tags: [Istio, Linkerd, Service Mesh, Tracing, Metrics]
---

當你的微服務數量達到 20+，你會發現一個問題：**每個服務都要自己實作 Tracing、Metrics、Retry、Circuit Breaker...**

這時候，Service Mesh 就派上用場了。

今天我們來看看 Istio 和 Linkerd 如何自動提供可觀測性。

## Service Mesh 的可觀測性優勢

### 傳統方式 vs Service Mesh

**傳統方式**：

```java
@RestController
public class OrderController {
    @Autowired
    private Tracer tracer;  // 手動注入
    
    @Autowired
    private MeterRegistry registry;  // 手動注入
    
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        Span span = tracer.nextSpan().name("get-order").start();  // 手動建立 Span
        Timer.Sample sample = Timer.start(registry);  // 手動計時
        
        try {
            Order order = orderService.findById(id);
            return order;
        } finally {
            span.finish();
            sample.stop(Timer.builder("http.request.duration").register(registry));
        }
    }
}
```

**Service Mesh 方式**：

```java
@RestController
public class OrderController {
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);  // 完全不用管 Tracing 和 Metrics
    }
}
```

Service Mesh 的 Sidecar Proxy 會自動處理：
- 分散式追蹤（Distributed Tracing）
- Metrics 收集（RED 指標）
- 服務間的重試、超時、熔斷

## Istio 的可觀測性

### 自動注入 Envoy Sidecar

```bash
kubectl label namespace default istio-injection=enabled
kubectl apply -f deployment.yaml
```

Istio 會自動在每個 Pod 中注入一個 Envoy Proxy：

```
┌─────────────────────┐
│       Pod           │
│  ┌──────────────┐   │
│  │ App Container│   │
│  └──────┬───────┘   │
│         │           │
│  ┌──────▼───────┐   │
│  │Envoy Sidecar │   │
│  └──────────────┘   │
└─────────────────────┘
```

所有的流量都會經過 Envoy，所以 Envoy 可以自動產生 Traces 和 Metrics。

### 自動產生的 Metrics

Istio 會自動產生這些 Metrics：

```promql
# 請求總數
istio_requests_total{
  source_workload="order-service",
  destination_workload="payment-service",
  response_code="200"
}

# 請求時長
istio_request_duration_milliseconds_bucket{
  source_workload="order-service",
  destination_workload="payment-service"
}

# TCP 連線數
istio_tcp_connections_opened_total{
  source_workload="order-service",
  destination_workload="database"
}
```

### 自動產生的 Traces

Istio 會自動產生 Traces，並發送到 Jaeger 或 Zipkin。

**配置 Istio 發送 Traces**：

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    defaultConfig:
      tracing:
        sampling: 100  # 採樣率 100%（開發環境），生產環境建議 1-5%
        zipkin:
          address: jaeger-collector.istio-system:9411
```

**產生的 Trace**：

```
order-service (ingress)
├── order-service: GET /api/orders/123 (125ms)
│   └── payment-service: POST /api/payment (78ms)
│       └── stripe-api: POST /v1/charges (65ms)
└── order-service: GET /api/inventory (23ms)
```

這些 Trace 是**自動產生的**，你的應用程式不需要做任何修改。

### Kiali：Istio 的可觀測性 UI

Kiali 是 Istio 的官方可觀測性工具，提供：
- 服務拓撲圖
- 流量監控
- 追蹤整合
- 配置驗證

**安裝 Kiali**：

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/kiali.yaml
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

打開 http://localhost:20001，你會看到：

```
┌─────────────┐
│order-service│─────┐
└─────────────┘     │
                    │ 150 req/s
                    │ P95: 78ms
                    ▼
            ┌──────────────────┐
            │payment-service   │
            └──────────────────┘
                    │
                    │ 150 req/s
                    │ P95: 65ms
                    ▼
            ┌──────────────────┐
            │stripe-api        │
            └──────────────────┘
```

點選任何一條線，可以看到：
- 成功率
- 延遲分佈（P50、P95、P99）
- 錯誤率

點選「View in Jaeger」，可以直接跳到 Jaeger 看 Traces。

## Linkerd 的可觀測性

Linkerd 比 Istio 更輕量，但可觀測性功能一樣強大。

### 安裝 Linkerd

```bash
curl -sL https://run.linkerd.io/install | sh
linkerd install | kubectl apply -f -
linkerd check
```

### 注入 Linkerd Proxy

```bash
kubectl get deploy order-service -o yaml | linkerd inject - | kubectl apply -f -
```

或者在部署時自動注入：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  annotations:
    linkerd.io/inject: enabled  # 自動注入
```

### 自動產生的 Metrics

Linkerd 會產生類似的 Metrics：

```promql
# 成功率
request_total{
  direction="outbound",
  dst_namespace="default",
  dst_service="payment-service",
  classification="success"
}

# 請求時長
response_latency_ms_bucket{
  direction="outbound",
  dst_service="payment-service"
}
```

### Linkerd Viz：內建的可觀測性 UI

```bash
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

打開 http://localhost:50750，你會看到類似 Kiali 的服務拓撲圖。

### Linkerd 的 Tap 功能

Linkerd 有一個很強大的功能：**實時抓包（Tap）**。

```bash
linkerd viz tap deploy/order-service
```

你會看到：

```
req id=1:1 proxy=in  src=10.1.0.5:52342 dst=10.1.0.8:8080 :method=GET :path=/api/orders/123
rsp id=1:1 proxy=in  src=10.1.0.5:52342 dst=10.1.0.8:8080 :status=200 latency=125ms
req id=2:1 proxy=out src=10.1.0.8:52343 dst=10.1.0.9:8080 :method=POST :path=/api/payment
rsp id=2:1 proxy=out src=10.1.0.8:52343 dst=10.1.0.9:8080 :status=200 latency=78ms
```

這等於是**實時的 Wireshark**，但只看應用層的 HTTP 流量。

你可以加上過濾條件：

```bash
# 只看錯誤的請求
linkerd viz tap deploy/order-service --path /api/orders --to deploy/payment-service | grep :status=5

# 只看特定的 trace_id
linkerd viz tap deploy/order-service --path /api/orders | grep trace_id=abc123
```

## Istio vs Linkerd

| 功能                  | Istio                  | Linkerd              |
|-----------------------|------------------------|----------------------|
| 自動 Tracing          | ✅                     | ✅                   |
| 自動 Metrics          | ✅                     | ✅                   |
| 自動 Logging          | ❌（需要手動配置）      | ❌                   |
| 服務拓撲圖            | ✅ Kiali               | ✅ Linkerd Viz       |
| 實時抓包              | ❌                     | ✅ Tap               |
| 資源消耗              | 較高（Envoy 較重）     | 較低（Rust Proxy）   |
| 學習曲線              | 陡峭                   | 平緩                 |
| 社群支援              | 非常活躍               | 活躍                 |

## 整合 Prometheus 和 Grafana

### Istio + Prometheus

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/prometheus.yaml
```

**Prometheus 配置**（自動生成）：

```yaml
scrape_configs:
  - job_name: 'istio-mesh'
    kubernetes_sd_configs:
    - role: endpoints
      namespaces:
        names:
        - istio-system
    relabel_configs:
    - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: istio-telemetry;prometheus
```

### Grafana Dashboard

Istio 官方提供了 Grafana Dashboard：

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/grafana.yaml
kubectl port-forward -n istio-system svc/grafana 3000:3000
```

打開 http://localhost:3000，選擇「Istio Service Dashboard」：

```
Service: payment-service

Success Rate:    99.8%  ━━━━━━━━━━━━━━━━━━━━ 99.8%
Request Volume:  1500 req/s
P50 Latency:     45ms
P95 Latency:     78ms
P99 Latency:     125ms

Inbound Success Rate by Source:
  order-service:    99.9%
  cart-service:     99.5%

Outbound Success Rate by Destination:
  stripe-api:       99.8%
  database:         100%
```

## 整合 Jaeger

### Istio + Jaeger

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/jaeger.yaml
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686
```

打開 http://localhost:16686，搜尋 `service=payment-service`，你會看到所有經過 Istio 的 Traces。

### Linkerd + Jaeger

Linkerd 也支援 Jaeger，但需要手動配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: linkerd-config
  namespace: linkerd
data:
  config: |
    tracing:
      collector_svc_addr: jaeger-collector.default:9411
      collector_svc_account: jaeger
```

```bash
linkerd upgrade --set tracing.enabled=true | kubectl apply -f -
```

## 從 Service Mesh 跳到應用程式的 Span

Service Mesh 只能看到**服務之間**的呼叫，看不到**服務內部**的細節。

要看到完整的 Trace，你的應用程式還是需要手動加入 Span。

**Java 範例**：

```java
@RestController
public class OrderController {
    @Autowired
    private Tracer tracer;
    
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // Istio 會自動產生一個 Span
        // 但我們可以加入更多細節
        
        Span dbSpan = tracer.buildSpan("db-query").start();
        try {
            Order order = orderRepository.findById(id);
            dbSpan.setTag("db.statement", "SELECT * FROM orders WHERE id = ?");
            return order;
        } finally {
            dbSpan.finish();
        }
    }
}
```

**產生的 Trace**：

```
istio-ingressgateway: GET /api/orders/123 (150ms)
└── order-service (envoy): GET /api/orders/123 (145ms)
    ├── order-service (app): db-query (35ms)  ← 你手動加的
    └── payment-service (envoy): POST /api/payment (78ms)
        └── payment-service (app): stripe-charge (65ms)  ← 手動加的
```

這樣就能看到**完整的呼叫鏈**。

## 實戰建議

### 1. 先用 Service Mesh，再補應用程式的 Span

一開始先用 Service Mesh，快速得到服務間的可觀測性。

等到需要更細的粒度，再手動加入應用程式的 Span。

### 2. 採樣率要調整

生產環境不要用 100% 採樣率，會產生太多資料。

建議：
- 1-5%：正常流量
- 100%：有錯誤的請求

**Istio 配置**：

```yaml
meshConfig:
  defaultConfig:
    tracing:
      sampling: 1  # 1%
      custom_tags:
        http.status_code:
          header:
            name: ":status"
```

### 3. 用 Service Mesh 取代應用程式的 Retry、Circuit Breaker

不要在應用程式中實作 Retry、Circuit Breaker，交給 Service Mesh 處理。

**Istio DestinationRule**：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      http:
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

這樣你的程式碼會更乾淨。

---

**Service Mesh 可以讓你的應用程式專注在業務邏輯，把可觀測性、重試、熔斷都交給基礎設施處理。**

如果你的微服務數量超過 10 個，強烈建議考慮 Service Mesh。
