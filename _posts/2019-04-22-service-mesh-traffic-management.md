---
layout: post
title: "服務網格流量管理進階"
date: 2019-04-22 10:00:00 +0800
categories: [DevOps, Service Mesh]
tags: [Istio, Traffic Management, Circuit Breaker, Retry, Timeout]
---

上週研究了配置中心（參考 [Spring Cloud Config](/posts/2019/04/15/spring-cloud-config/)），這週回到 Istio，深入研究流量管理的進階功能。

之前我們學過 Istio 的基礎（參考 [Service Mesh 入門](/posts/2019/03/18/service-mesh-istio-intro/)），這週要研究更細緻的流量控制：重試、超時、熔斷器、連線池管理等。

我們的問題：

上週五晚上，payment-service 突然變慢（回應時間從 100ms 變成 5 秒），導致整個系統都變慢。更糟的是，大量請求堆積，把 payment-service 徹底壓垮，完全無法回應。

需要更智慧的流量管理！

> 使用版本：Istio 1.0.6

## 超時控制 (Timeout)

設定請求超時，避免無限等待。

`virtualservice-timeout.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
    timeout: 2s  # 2 秒超時
```

套用：
```bash
kubectl apply -f virtualservice-timeout.yaml
```

現在呼叫 payment-service，如果 2 秒內沒回應，自動失敗，不會一直等。

### 測試超時

模擬慢服務：
```java
@GetMapping("/payment/{id}")
public Payment getPayment(@PathVariable String id) throws InterruptedException {
    Thread.sleep(3000);  // 睡 3 秒
    return paymentRepository.findById(id);
}
```

呼叫：
```bash
curl http://payment-service/payment/123

# 2 秒後回傳
# upstream request timeout
```

保護了呼叫方，不會無限等待。

## 重試機制 (Retry)

暫時性錯誤自動重試。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
    timeout: 2s
    retries:
      attempts: 3
      perTryTimeout: 1s
      retryOn: 5xx,reset,connect-failure,refused-stream
```

參數說明：
- `attempts: 3`：最多重試 3 次
- `perTryTimeout: 1s`：每次嘗試超時 1 秒
- `retryOn`：什麼情況下重試
  - `5xx`：伺服器錯誤
  - `reset`：連線重置
  - `connect-failure`：連線失敗
  - `refused-stream`：串流被拒絕

### 指數退避

預設重試立即執行，可能雪上加霜。使用指數退避：

```yaml
retries:
  attempts: 3
  perTryTimeout: 1s
  retryOn: 5xx
  exponentialBackoff:
    baseInterval: 100ms
    maxInterval: 10s
```

重試間隔：100ms → 200ms → 400ms...

## 熔斷器 (Circuit Breaker)

服務異常時，停止呼叫，避免雪崩。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5        # 連續 5 次錯誤
      interval: 30s               # 檢測間隔 30 秒
      baseEjectionTime: 30s       # 驅逐時間 30 秒
      maxEjectionPercent: 50      # 最多驅逐 50% 的 Pod
      minHealthPercent: 30        # 至少保持 30% 健康 Pod
```

運作方式：
1. 監測每個 Pod 的錯誤率
2. Pod 連續 5 次錯誤 → 驅逐（不再發送流量）
3. 30 秒後，嘗試恢復該 Pod
4. 如果還是有問題，繼續驅逐

就像電路保險絲，問題 Pod 被「切斷」。

### 測試熔斷器

模擬錯誤 Pod：
```java
private AtomicInteger counter = new AtomicInteger(0);

@GetMapping("/payment/{id}")
public ResponseEntity<Payment> getPayment(@PathVariable String id) {
    // 每 2 次請求中有 1 次失敗
    if (counter.incrementAndGet() % 2 == 0) {
        return ResponseEntity.status(500).build();
    }
    return ResponseEntity.ok(paymentRepository.findById(id));
}
```

發送請求：
```bash
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://payment-service/payment/123
done
```

觀察：
```
200
500
200
500
200
500  ← 第 5 次錯誤
(這個 Pod 被驅逐)
200  ← 流量轉到其他 Pod
200
200
```

有問題的 Pod 被隔離，保護整體系統。

## 連線池管理

限制對後端服務的並發連線數。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100      # 最多 100 個 TCP 連線
      http:
        http1MaxPendingRequests: 50   # 最多 50 個等待請求
        http2MaxRequests: 100          # HTTP/2 最多 100 個請求
        maxRequestsPerConnection: 2    # 每個連線最多 2 個請求
```

超過限制的請求會被拒絕（返回 503），保護後端服務不被壓垮。

## 流量鏡像 (Traffic Mirroring)

複製流量到另一個版本，測試新版本但不影響使用者。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 100
    mirror:
      host: payment-service
      subset: v2
    mirror_percent: 100  # 鏡像 100% 流量
```

流量處理：
1. 請求打到 v1，正常回應給使用者
2. 同時複製請求到 v2（使用者不感知）
3. v2 的回應被丟棄（只用來測試）

用途：
- 測試新版本的效能
- 驗證新版本沒有錯誤
- 比對新舊版本的回應差異

## 故障注入進階

### 延遲注入（特定條件）

只對特定使用者注入延遲：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        user-id:
          exact: "test-user-123"
    fault:
      delay:
        fixedDelay: 5s
        percentage:
          value: 100
    route:
    - destination:
        host: payment-service
  - route:
    - destination:
        host: payment-service
```

測試帳號會遇到延遲，正常使用者不受影響。

### 隨機錯誤

模擬不穩定的服務：

```yaml
fault:
  abort:
    httpStatus: 503
    percentage:
      value: 10  # 10% 機率回傳 503
```

測試系統的容錯能力。

## 請求路由進階

### 基於權重的多版本路由

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 80
    - destination:
        host: payment-service
        subset: v2
      weight: 15
    - destination:
        host: payment-service
        subset: v3-canary
      weight: 5
```

同時測試三個版本！

### 基於請求內容路由

根據 HTTP Header 路由：

```yaml
http:
- match:
  - headers:
      x-api-version:
        exact: "v2"
  route:
  - destination:
      host: payment-service
      subset: v2
- match:
  - headers:
      x-api-version:
        exact: "v1"
  route:
  - destination:
      host: payment-service
      subset: v1
- route:  # 預設
  - destination:
      host: payment-service
      subset: v1
```

客戶端可以選擇使用哪個版本：
```bash
curl -H "x-api-version: v2" http://payment-service/payment/123
```

根據 URI 參數路由：

```yaml
http:
- match:
  - uri:
      prefix: "/api/v2"
  route:
  - destination:
      host: payment-service
      subset: v2
- match:
  - uri:
      prefix: "/api/v1"
  route:
  - destination:
      host: payment-service
      subset: v1
```

## 區域感知路由

優先路由到同一區域的服務，減少跨區延遲。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
    loadBalancer:
      localityLbSetting:
        enabled: true
        distribute:
        - from: us-west/zone1/*
          to:
            "us-west/zone1/*": 80
            "us-west/zone2/*": 20
```

us-west/zone1 的請求：
- 80% 打到同 zone1 的 Pod
- 20% 打到 zone2（避免單點故障）

## 實際應用案例

### 案例一：保護資料庫

payment-service 呼叫資料庫，設定連線池和超時：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: payment-db
spec:
  hosts:
  - payment-db.example.com
  ports:
  - number: 3306
    name: mysql
    protocol: TCP
  location: MESH_EXTERNAL
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-db
spec:
  host: payment-db.example.com
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 50
        connectTimeout: 3s
    outlierDetection:
      consecutiveErrors: 3
      interval: 10s
      baseEjectionTime: 30s
```

保護資料庫：
- 最多 50 個連線
- 連線超時 3 秒
- 連續 3 次失敗就停止連線 30 秒

### 案例二：金絲雀自動化

結合 Prometheus 指標，自動調整金絲雀流量。

初始：
```yaml
# 5% 流量到 v2
- destination:
    host: payment-service
    subset: v1
  weight: 95
- destination:
    host: payment-service
    subset: v2
  weight: 5
```

監控 v2 的錯誤率：
```promql
rate(istio_requests_total{
  destination_service="payment-service",
  destination_version="v2",
  response_code=~"5.."
}[5m])
```

如果錯誤率 < 1%：
1. 增加到 20%
2. 繼續監控
3. 逐步增加到 100%

如果錯誤率 > 5%：
1. 立刻回滾到 0%
2. 告警通知

使用 Flagger 自動化這個流程。

### 案例三：跨地域災難恢復

主區域故障，自動切換到備援區域。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 10s
      baseEjectionTime: 5m
    loadBalancer:
      localityLbSetting:
        enabled: true
        failover:
        - from: us-west
          to: us-east  # 主區域故障時切到東岸
```

us-west 整個掛了，流量自動切到 us-east。

## 監控流量管理

### Grafana Dashboard

查看 Istio 流量指標：

**成功率**：
```promql
sum(rate(istio_requests_total{
  destination_service="payment-service",
  response_code!~"5.."
}[5m])) / 
sum(rate(istio_requests_total{
  destination_service="payment-service"
}[5m])) * 100
```

**P99 延遲**：
```promql
histogram_quantile(0.99,
  sum(rate(istio_request_duration_seconds_bucket{
    destination_service="payment-service"
  }[5m])) by (le)
)
```

**熔斷器觸發次數**：
```promql
sum(increase(istio_tcp_connections_closed_total{
  destination_service="payment-service",
  response_flags="UO"
}[5m]))
```

### Kiali 視覺化

開啟 Kiali，可以看到：
- 服務拓撲
- 流量分布（v1: 80%, v2: 20%）
- 錯誤率（紅色標示）
- 重試次數
- 超時次數

非常直觀！

## 遇到的問題

### 問題一：重試風暴

設定重試 3 次，當服務慢時，請求量變成 4 倍（原始 + 3 次重試），反而壓垮服務。

解決方法：
1. 設定合理的重試次數（2-3 次）
2. 使用指數退避
3. 只對冪等操作重試（GET、PUT，不要 POST）
4. 設定重試預算（全域最多重試比例）

```yaml
retries:
  attempts: 2
  perTryTimeout: 1s
  retryOn: 5xx
  exponentialBackoff:
    baseInterval: 100ms
    maxInterval: 2s
```

### 問題二：熔斷器太敏感

設定 `consecutiveErrors: 2`，偶爾一兩個錯誤就觸發熔斷，影響可用性。

解決方法：
1. 增加閾值（5-10 次）
2. 考慮錯誤率而不是絕對次數
3. 調整檢測間隔

```yaml
outlierDetection:
  consecutiveErrors: 7
  interval: 30s
  baseEjectionTime: 30s
```

### 問題三：超時設定不一致

服務 A 呼叫服務 B，A 的超時 5 秒，B 的超時 10 秒，導致混亂。

解決方法：
1. 統一管理超時設定
2. 下游超時應該小於上游
3. 文件化所有超時設定

```
API Gateway: 10s
├─ Order Service: 8s
│  ├─ User Service: 2s
│  ├─ Product Service: 3s
│  └─ Payment Service: 2s
```

## 心得

Istio 的流量管理功能真的很強大。以前要實現這些功能，要在每個服務寫一堆程式碼（重試、熔斷器、超時），而且容易出錯。

現在只要寫 YAML，Istio 自動處理，而且是經過驗證的實作。

特別喜歡熔斷器，上次 payment-service 有問題，自動隔離有問題的 Pod，其他 Pod 還能正常服務。以前整個服務都會掛掉。

流量鏡像也很實用，可以放心測試新版本，不會影響使用者。我們現在每次上線前都會先鏡像流量測試一週。

不過要注意配置不要太複雜，曾經設定了一堆規則，連自己都搞不清楚流量怎麼走。保持簡單，只用真正需要的功能。

下週要研究混沌工程，主動注入故障測試系統的韌性。
