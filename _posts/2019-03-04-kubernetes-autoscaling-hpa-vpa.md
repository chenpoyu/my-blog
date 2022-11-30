---
layout: post
title: "Kubernetes 自動擴展 - HPA 和 VPA"
date: 2019-03-04 10:00:00 +0800
categories: [DevOps, Kubernetes]
tags: [Kubernetes, HPA, VPA, Autoscaling, Cloud Native]
---

上週學了 Ansible 自動化配置（參考 [Ansible 自動化配置管理](/posts/2019/02/25/ansible-automation-configuration/)），這週回到 Kubernetes，研究自動擴展功能。

我們的問題：

每到活動期間（例如雙十一、週年慶），流量暴增，應用開始變慢、甚至當機。然後我們手忙腳亂地：
1. `kubectl scale deployment myapp --replicas=10`
2. 等 Pod 啟動
3. 活動結束後忘記縮回去
4. 月底收到帳單嚇一跳

需要自動化！流量多時自動擴展，流量少時自動縮減。

Kubernetes 提供兩種自動擴展：
- **HPA (Horizontal Pod Autoscaler)**：水平擴展，增減 Pod 數量
- **VPA (Vertical Pod Autoscaler)**：垂直擴展，調整單個 Pod 的資源

這週先研究 HPA。

> 使用版本：Kubernetes 1.13（2018 年 12 月發布），HPA v2beta2

## HPA 基本原理

HPA 持續監控指標（CPU、記憶體、自訂指標），自動調整 Deployment 的 replicas。

流程：
1. HPA Controller 每 15 秒檢查一次指標
2. 計算所需的 Pod 數量
3. 調整 Deployment 的 replicas
4. Deployment Controller 建立或刪除 Pod

公式：
```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / targetMetricValue)]
```

例如：
- 目前 3 個 Pod
- 平均 CPU 使用率 90%
- 目標 CPU 使用率 50%
- 計算：`ceil[3 * (90 / 50)] = ceil[5.4] = 6`
- 擴展到 6 個 Pod

## 前置準備：Metrics Server

HPA 需要 Metrics Server 提供資源指標。

### 安裝 Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.1/metrics-server-0.3.1.yaml
```

等待 Pod 啟動：
```bash
kubectl get pods -n kube-system | grep metrics-server
# metrics-server-v0.3.1-5c6fbf777-abcde   1/1     Running   0          1m
```

驗證：
```bash
kubectl top nodes
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1     250m         12%    1500Mi          40%
# node-2     180m         9%     1200Mi          32%

kubectl top pods
# NAME                     CPU(cores)   MEMORY(bytes)
# myapp-5b9c7d6f4c-abcde   50m          128Mi
# myapp-5b9c7d6f4c-fghij   45m          120Mi
```

## 基於 CPU 的 HPA

### 部署應用

`deployment.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m        # 重要！HPA 需要這個
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

**重點**：必須設定 `resources.requests`，HPA 才能計算使用率。

部署：
```bash
kubectl apply -f deployment.yaml
```

### 建立 HPA

方法一：使用 `kubectl autoscale`
```bash
kubectl autoscale deployment myapp --cpu-percent=50 --min=2 --max=10

# horizontalpodautoscaler.autoscaling/myapp autoscaled
```

方法二：使用 YAML

`hpa.yaml`：
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

解釋：
- `minReplicas: 2`：最少 2 個 Pod
- `maxReplicas: 10`：最多 10 個 Pod
- `averageUtilization: 50`：平均 CPU 使用率目標 50%

部署：
```bash
kubectl apply -f hpa.yaml
```

### 檢查 HPA 狀態

```bash
kubectl get hpa
# NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# myapp   Deployment/myapp   15%/50%   2         10        2          1m
```

說明：
- `TARGETS`: 15%/50% 表示目前 CPU 使用率 15%，目標 50%
- `REPLICAS`: 目前 2 個 Pod

詳細資訊：
```bash
kubectl describe hpa myapp

# Name:                                                  myapp
# Namespace:                                             default
# Reference:                                             Deployment/myapp
# Metrics:                                               ( current / target )
#   resource cpu on pods  (as a percentage of request):  15% (15m) / 50%
# Min replicas:                                          2
# Max replicas:                                          10
# Deployment pods:                                       2 current / 2 desired
```

## 測試自動擴展

### 產生負載

使用 `kubectl run` 啟動一個 Pod 產生流量：

```bash
kubectl run -it --rm load-generator --image=busybox /bin/sh

# 在 Pod 內執行
while true; do wget -q -O- http://myapp; done
```

### 觀察擴展

開另一個終端監控：
```bash
kubectl get hpa myapp --watch

# NAME    REFERENCE          TARGETS    MINPODS   MAXPODS   REPLICAS
# myapp   Deployment/myapp   15%/50%    2         10        2
# myapp   Deployment/myapp   65%/50%    2         10        2          # CPU 上升
# myapp   Deployment/myapp   65%/50%    2         10        3          # 擴展到 3
# myapp   Deployment/myapp   58%/50%    2         10        3
# myapp   Deployment/myapp   58%/50%    2         10        4          # 擴展到 4
# myapp   Deployment/myapp   52%/50%    2         10        4
# myapp   Deployment/myapp   48%/50%    2         10        4          # 穩定了
```

檢查 Pod：
```bash
kubectl get pods -l app=myapp

# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-5b9c7d6f4c-abcde   1/1     Running   0          5m
# myapp-5b9c7d6f4c-fghij   1/1     Running   0          5m
# myapp-5b9c7d6f4c-klmno   1/1     Running   0          1m
# myapp-5b9c7d6f4c-pqrst   1/1     Running   0          30s
```

### 停止負載

停止 load-generator（Ctrl+C），觀察縮減：

```bash
kubectl get hpa myapp --watch

# myapp   Deployment/myapp   48%/50%    2         10        4
# myapp   Deployment/myapp   12%/50%    2         10        4          # CPU 下降
# myapp   Deployment/myapp   12%/50%    2         10        4          # 等待 5 分鐘
# myapp   Deployment/myapp   12%/50%    2         10        3          # 縮減到 3
# myapp   Deployment/myapp   10%/50%    2         10        3
# myapp   Deployment/myapp   10%/50%    2         10        2          # 縮減到 2
```

**注意**：縮減比擴展慢，預設等待 5 分鐘才縮減，避免頻繁抖動。

## 基於記憶體的 HPA

`hpa-memory.yaml`：
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-memory
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

## 多指標 HPA

同時根據 CPU 和記憶體：

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-multi
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

HPA 會計算每個指標需要的 replicas，取**最大值**。

例如：
- CPU 需要 5 個 Pod
- 記憶體需要 3 個 Pod
- 實際擴展到 5 個 Pod

## 自訂指標

基於應用的自訂指標，例如 HTTP 請求數、佇列長度。

需要：
1. 應用暴露指標（Prometheus format）
2. Prometheus 收集指標
3. Prometheus Adapter 提供 Custom Metrics API

### 安裝 Prometheus Adapter

```bash
helm install prometheus-adapter stable/prometheus-adapter \
  --set prometheus.url=http://prometheus-server.monitoring.svc \
  --set prometheus.port=80
```

### 設定自訂指標

`custom-metrics-config.yaml`：
```yaml
rules:
- seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    matches: "^(.*)_total$"
    as: "${1}_per_second"
  metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)'
```

### 使用自訂指標

`hpa-custom.yaml`：
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

意思：每個 Pod 處理 1000 requests/sec，超過就擴展。

## HPA 行為調整

Kubernetes 1.13 開始支援調整擴展行為（v2beta2）。

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-behavior
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0        # 立即擴展
      policies:
      - type: Percent
        value: 100                          # 每次最多增加 100%
        periodSeconds: 15
      - type: Pods
        value: 4                            # 或每次最多增加 4 個 Pod
        periodSeconds: 15
      selectPolicy: Max                     # 取最大值
    scaleDown:
      stabilizationWindowSeconds: 300      # 觀察 5 分鐘才縮減
      policies:
      - type: Percent
        value: 50                           # 每次最多減少 50%
        periodSeconds: 15
      - type: Pods
        value: 2                            # 或每次最多減少 2 個 Pod
        periodSeconds: 15
      selectPolicy: Min                     # 取最小值（保守縮減）
```

## VPA 簡介

Vertical Pod Autoscaler 調整 Pod 的 CPU/記憶體 requests 和 limits。

適用場景：
- 不確定應該設定多少 resources
- 負載模式會變化

安裝：
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

使用：
```yaml
apiVersion: autoscaling.k8s.io/v1beta2
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
```

VPA 會：
1. 監控 Pod 的實際使用量
2. 推薦合適的 requests/limits
3. 自動更新並重啟 Pod

**注意**：VPA 會重啟 Pod，適合無狀態應用。

## 實際應用經驗

### 案例一：電商促銷

雙十一活動，流量是平時的 10 倍。

設定：
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: shopping-cart
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shopping-cart
  minReplicas: 5           # 活動前先提高 min
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"
```

結果：
- 活動開始，流量湧入
- HPA 在 2 分鐘內擴展到 30 個 Pod
- 系統穩定，沒有當機
- 活動結束後自動縮減到 5 個 Pod

成功！

### 案例二：背景任務處理

處理訂單的背景 worker，根據 RabbitMQ 佇列長度擴展。

使用 KEDA (Kubernetes Event Driven Autoscaling)：

```yaml
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
spec:
  scaleTargetRef:
    deploymentName: order-processor
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: rabbitmq
    metadata:
      queueName: orders
      queueLength: "10"
      host: amqp://user:password@rabbitmq.default.svc.cluster.local:5672
```

意思：佇列有超過 10 個訊息，就增加 worker。

## 遇到的問題

### 問題一：HPA 顯示 `<unknown>`

```bash
kubectl get hpa
# NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS
# myapp   Deployment/myapp   <unknown>/50%   2         10        0
```

原因：
1. Metrics Server 沒裝或沒啟動
2. Deployment 沒設定 `resources.requests`

解決：檢查 Metrics Server，補上 resources.requests。

### 問題二：頻繁抖動

Pod 數量一直在變：2 → 3 → 2 → 3 → 2...

原因：指標在閾值附近波動。

解決：
1. 調整 `averageUtilization`，留些緩衝
2. 增加 `stabilizationWindowSeconds`
3. 設定合理的 `scaleUp` 和 `scaleDown` 策略

### 問題三：擴展不夠快

流量暴增，但 HPA 反應太慢，還是有請求失敗。

解決：
1. 降低 `stabilizationWindowSeconds`（擴展時）
2. 增加 `maxReplicas`
3. 提高 `minReplicas`（活動前預先擴展）
4. 使用更激進的擴展策略

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
```

## 心得

HPA 真的是雲原生應用的必備功能。以前每次活動都要提心吊膽，深怕系統撐不住。現在有了 HPA，系統能自動應對流量變化，我終於可以在活動期間安心睡覺了。

而且省錢！以前為了應對尖峰，平時也開著一堆 Pod，浪費資源。現在平時只開 2 個，需要時自動擴展，成本降低 60%。

不過也要注意，HPA 不是萬能的：
- 應用要設計成無狀態，才能隨意擴展
- 資料庫之類的有狀態服務不適合 HPA
- 擴展速度有限，極端情況還是要預先準備

下週要研究 Helm，Kubernetes 的套件管理工具，讓部署更簡單。
