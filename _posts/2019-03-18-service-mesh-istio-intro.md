---
layout: post
title: "Service Mesh 入門 - Istio"
date: 2019-03-18 10:00:00 +0800
categories: [DevOps, Microservices]
tags: [Istio, Service Mesh, Kubernetes, Microservices, Observability]
---

上週學了 Helm（參考 [Helm Charts 套件管理](/posts/2019/03/11/helm-charts-package-management/)），這週來研究微服務架構的新趨勢：Service Mesh。

我們的微服務架構遇到一些問題：

1. **服務間通訊複雜**：30 個微服務互相呼叫，很難追蹤請求路徑
2. **重試邏輯散落各處**：每個服務都要自己寫重試、timeout、circuit breaker
3. **監控困難**：不知道哪個服務呼叫哪個服務，哪裡慢了
4. **流量控制難**：想做金絲雀部署、A/B 測試，要改程式碼
5. **安全性**：服務間通訊沒加密，任何 Pod 都能呼叫內部 API

Istio Service Mesh 能解決這些問題，而且**不用改程式碼**！

> 使用版本：Istio 1.0.6（2019 年初的穩定版）

## Service Mesh 是什麼

Service Mesh 是微服務架構中的基礎設施層，負責處理服務間通訊。

傳統做法：
```
Service A ----直接呼叫----> Service B
```

Service Mesh：
```
Service A --> Sidecar Proxy --> Sidecar Proxy --> Service B
```

每個服務旁邊注入一個 Sidecar Proxy（Envoy），所有流量都經過 Proxy。Proxy 負責：
- 流量管理（路由、重試、超時）
- 安全（mTLS、認證授權）
- 可觀測性（追蹤、監控）

應用程式完全不知道 Proxy 的存在，這些功能都由基礎設施層提供。

## Istio 架構

Istio 分為兩層：

### Data Plane

- **Envoy Proxy**：作為 Sidecar 注入到每個 Pod，處理流量

### Control Plane

- **Pilot**：管理流量規則，配置 Envoy
- **Citadel**：管理證書，提供 mTLS
- **Galley**：驗證配置
- **Mixer**（已棄用）：遙測和策略檢查

## 安裝 Istio

### 下載 Istio

```bash
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.0.6 sh -

cd istio-1.0.6

# 加入 PATH
export PATH=$PWD/bin:$PATH

# 驗證
istioctl version
# Version: 1.0.6
```

### 安裝到 Kubernetes

使用 Helm 安裝：

```bash
# 建立 namespace
kubectl create namespace istio-system

# 安裝 Istio CRDs
helm install install/kubernetes/helm/istio-init \
  --name istio-init \
  --namespace istio-system

# 等待 CRDs 建立完成
kubectl get crds | grep 'istio.io' | wc -l
# 應該是 53

# 安裝 Istio
helm install install/kubernetes/helm/istio \
  --name istio \
  --namespace istio-system \
  --set global.mtls.enabled=false \
  --set tracing.enabled=true \
  --set kiali.enabled=true \
  --set grafana.enabled=true
```

參數說明：
- `global.mtls.enabled=false`：先不啟用 mTLS，簡化學習
- `tracing.enabled=true`：啟用分散式追蹤（Jaeger）
- `kiali.enabled=true`：啟用服務拓撲視覺化
- `grafana.enabled=true`：啟用監控儀表板

等待所有 Pod 啟動：
```bash
kubectl get pods -n istio-system

# NAME                                      READY   STATUS    RESTARTS   AGE
# grafana-...                               1/1     Running   0          2m
# istio-citadel-...                         1/1     Running   0          2m
# istio-egressgateway-...                   1/1     Running   0          2m
# istio-galley-...                          1/1     Running   0          2m
# istio-ingressgateway-...                  1/1     Running   0          2m
# istio-pilot-...                           2/2     Running   0          2m
# istio-policy-...                          2/2     Running   0          2m
# istio-sidecar-injector-...                1/1     Running   0          2m
# istio-telemetry-...                       2/2     Running   0          2m
# istio-tracing-...                         1/1     Running   0          2m
# kiali-...                                 1/1     Running   0          2m
# prometheus-...                            1/1     Running   0          2m
```

## 自動注入 Sidecar

讓 Istio 自動注入 Envoy Sidecar 到 Pod。

### 啟用 namespace 自動注入

```bash
kubectl label namespace default istio-injection=enabled

# 驗證
kubectl get namespace -L istio-injection
# NAME              STATUS   AGE   ISTIO-INJECTION
# default           Active   10d   enabled
# istio-system      Active   5m
# kube-system       Active   10d
```

### 部署應用

使用 Istio 範例應用 BookInfo：

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# service/details created
# deployment.apps/details-v1 created
# service/ratings created
# deployment.apps/ratings-v1 created
# service/reviews created
# deployment.apps/reviews-v1 created
# deployment.apps/reviews-v2 created
# deployment.apps/reviews-v3 created
# service/productpage created
# deployment.apps/productpage-v1 created
```

檢查 Pod：
```bash
kubectl get pods

# NAME                              READY   STATUS    RESTARTS   AGE
# details-v1-...                    2/2     Running   0          1m
# productpage-v1-...                2/2     Running   0          1m
# ratings-v1-...                    2/2     Running   0          1m
# reviews-v1-...                    2/2     Running   0          1m
# reviews-v2-...                    2/2     Running   0          1m
# reviews-v3-...                    2/2     Running   0          1m
```

注意 `READY` 是 `2/2`，表示有 2 個容器：應用 + Envoy Sidecar！

查看詳細資訊：
```bash
kubectl describe pod productpage-v1-... | grep -A 5 "Containers:"

# Containers:
#   productpage:
#     Image:         istio/examples-bookinfo-productpage-v1:1.10.1
#   istio-proxy:
#     Image:         docker.io/istio/proxyv2:1.0.6
```

Sidecar 自動注入了！

### 設定 Ingress Gateway

建立 Gateway 和 VirtualService 讓外部能訪問：

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# gateway.networking.istio.io/bookinfo-gateway created
# virtualservice.networking.istio.io/bookinfo created
```

取得 Ingress Gateway 的外部 IP：
```bash
kubectl get svc istio-ingressgateway -n istio-system

# NAME                   TYPE           EXTERNAL-IP     PORT(S)
# istio-ingressgateway   LoadBalancer   35.200.123.45   80:31380/TCP,443:31390/TCP,...
```

訪問應用：
```bash
export GATEWAY_URL=35.200.123.45

curl http://$GATEWAY_URL/productpage
# 看到 HTML 內容
```

或在瀏覽器開啟 `http://35.200.123.45/productpage`，可以看到書籍資訊頁面。

多重新整理幾次，會看到 reviews 區塊有三種版本：
- v1：沒有星星
- v2：黑色星星
- v3：紅色星星

這是因為預設流量均勻分配到三個版本。

## 流量管理

### 設定預設路由

讓所有流量都去 v1：

`destination-rule-all.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
```

`virtual-service-all-v1.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
```

套用：
```bash
kubectl apply -f destination-rule-all.yaml
kubectl apply -f virtual-service-all-v1.yaml
```

現在重新整理頁面，永遠都是 v1（沒有星星）。

### 基於使用者的路由

讓特定使用者看到 v2：

`virtual-service-reviews-test-v2.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

套用：
```bash
kubectl apply -f virtual-service-reviews-test-v2.yaml
```

測試：
1. 開啟頁面，點選右上角 "Sign in"
2. 使用者名稱輸入 `jason`（密碼隨意）
3. 看到黑色星星（v2）
4. 登出，看到沒有星星（v1）

其他使用者都看 v1，只有 jason 看 v2。這就是 A/B 測試！

### 金絲雀部署

90% 流量去 v1，10% 去 v3：

`virtual-service-reviews-90-10.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v3
      weight: 10
```

套用：
```bash
kubectl apply -f virtual-service-reviews-90-10.yaml
```

重新整理頁面多次，大約 10 次有 1 次看到紅色星星（v3）。

確認沒問題後，逐步增加 v3 的流量：50/50 → 10/90 → 0/100。

### 故障注入

測試應用的容錯能力。

#### 注入延遲

讓 ratings 服務對 jason 延遲 7 秒：

`virtual-service-ratings-delay.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

套用：
```bash
kubectl apply -f virtual-service-ratings-delay.yaml
```

登入為 jason，頁面載入超慢（7 秒），reviews 因為 timeout 顯示錯誤。

發現問題：reviews 服務的 timeout 設定太短！

#### 注入錯誤

讓 ratings 服務對 jason 回傳 HTTP 500：

`virtual-service-ratings-abort.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      abort:
        percentage:
          value: 100.0
        httpStatus: 500
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

登入為 jason，reviews 顯示 "Ratings service is currently unavailable"。

測試容錯機制是否正常！

## 可觀測性

### Kiali - 服務拓撲

Kiali 視覺化服務間的呼叫關係。

開啟 Kiali：
```bash
kubectl port-forward -n istio-system \
  $(kubectl get pod -n istio-system -l app=kiali -o jsonpath='{.items[0].metadata.name}') \
  20001:20001
```

瀏覽器開啟 `http://localhost:20001`，預設帳密 `admin/admin`。

在 Kiali 可以看到：
- 服務拓撲圖
- 請求流量（QPS）
- 錯誤率
- 回應時間

點選 Graph，選擇 namespace `default`，產生一些流量：
```bash
for i in {1..100}; do
  curl http://$GATEWAY_URL/productpage > /dev/null
  sleep 0.5
done
```

Kiali 會顯示：
```
productpage --> reviews (v1, v2, v3) --> ratings
            \-> details
```

非常直觀！

### Jaeger - 分散式追蹤

Jaeger 追蹤單一請求跨越多個服務的路徑。

開啟 Jaeger：
```bash
kubectl port-forward -n istio-system \
  $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') \
  16686:16686
```

瀏覽器開啟 `http://localhost:16686`。

選擇 Service `productpage.default`，點 "Find Traces"，可以看到每個請求的：
- 總耗時
- 經過哪些服務
- 每個服務的耗時

點進去看詳細的 Trace：
```
productpage.default (50ms)
├── details.default (10ms)
└── reviews.default (30ms)
    └── ratings.default (8ms)
```

一目了然！找效能瓶頸超方便。

### Grafana - 監控儀表板

Grafana 顯示 Istio 的監控指標。

開啟 Grafana：
```bash
kubectl port-forward -n istio-system \
  $(kubectl get pod -n istio-system -l app=grafana -o jsonpath='{.items[0].metadata.name}') \
  3000:3000
```

瀏覽器開啟 `http://localhost:3000`。

Istio 提供多個預製儀表板：
- **Istio Mesh Dashboard**：整體健康狀況
- **Istio Service Dashboard**：個別服務的指標
- **Istio Workload Dashboard**：個別 Pod 的指標

可以看到：
- 請求量（QPS）
- 成功率
- P50/P90/P99 延遲
- TCP 連線數

## 安全

### 啟用 mTLS

讓服務間通訊自動加密。

啟用全域 mTLS：
```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "MeshPolicy"
metadata:
  name: "default"
spec:
  peers:
  - mtls: {}
```

設定 DestinationRule 使用 mTLS：
```yaml
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

套用：
```bash
kubectl apply -f mtls.yaml
```

現在所有服務間通訊都自動加密了，應用程式完全不知道！

Istio 自動：
- 產生證書
- 分發到每個 Sidecar
- 定期輪換證書

### 授權策略

只允許特定服務呼叫 ratings：

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "ratings-viewer"
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
    to:
    - operation:
        methods: ["GET"]
```

只有 reviews 服務能 GET ratings，其他服務呼叫會被拒絕。

## 實際應用經驗

我們把公司的微服務架構導入 Istio，效果顯著：

### 問題追蹤變簡單

以前有使用者反應頁面慢，我們要：
1. 看 log 猜是哪個服務慢
2. 加 log 重新部署
3. 重現問題
4. 分析 log

現在直接開 Jaeger，找到那個請求的 Trace，立刻知道瓶頸在哪。

### 金絲雀部署零風險

以前做金絲雀部署：
1. 手動改 Kubernetes Service 的 selector
2. 手動計算要幾個 Pod
3. 觀察監控
4. 手動調整

現在：
```yaml
http:
- route:
  - destination:
      host: myapp
      subset: v1
    weight: 95
  - destination:
      host: myapp
      subset: v2
    weight: 5
```

改一下 weight，完成！

### 服務間通訊加密

以前要服務間 mTLS：
1. 每個服務都要改程式碼
2. 管理證書
3. 處理證書輪換

現在 Istio 全自動，零程式碼修改。

## 遇到的問題

### 問題一：Sidecar 注入失敗

Pod 只有 1 個容器，沒有 Sidecar。

原因：namespace 沒設定 `istio-injection=enabled`。

解決：
```bash
kubectl label namespace default istio-injection=enabled

# 重新部署 Pod
kubectl delete pods --all
```

### 問題二：mTLS 導致連線失敗

啟用 mTLS 後，外部服務無法連線。

原因：外部服務沒有 Sidecar，無法進行 mTLS 握手。

解決：
```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "external-service"
spec:
  targets:
  - name: external-api
  peers:
  - mtls:
      mode: PERMISSIVE  # 允許 plaintext 和 mTLS
```

### 問題三：效能開銷

加了 Sidecar 後，延遲增加 5-10ms。

這是正常的，因為多一層 Proxy。但換來的功能值得：
- 自動重試
- 負載平衡
- mTLS
- 可觀測性

## 心得

Istio 真的是微服務架構的遊戲規則改變者。以前要實現的功能（重試、熔斷、追蹤、mTLS）都要自己寫程式碼，而且要在每個服務重複實現。

現在這些都由基礎設施層提供，應用程式只要專注業務邏輯。

特別喜歡 Kiali 和 Jaeger，視覺化讓複雜的微服務架構變得清晰可見。以前要花一小時追蹤的問題，現在幾分鐘就找到。

不過 Istio 的學習曲線確實陡峭，概念很多（VirtualService、DestinationRule、Gateway...），需要時間消化。

下週要研究部署策略（金絲雀、藍綠、滾動），搭配 Istio 實現零停機部署。
