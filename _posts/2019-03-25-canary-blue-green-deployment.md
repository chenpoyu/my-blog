---
layout: post
title: "金絲雀部署和藍綠部署策略"
date: 2019-03-25 10:00:00 +0800
categories: [DevOps, Deployment]
tags: [Deployment Strategy, Canary, Blue-Green, Rolling Update, Kubernetes]
---

上週學了 Istio Service Mesh（參考 [Service Mesh 入門](/posts/2019/03/18/service-mesh-istio-intro/)），這週深入研究各種部署策略，目標是實現零停機、低風險的持續部署。

我們的痛點：

去年有次上線，新版本有個嚴重 Bug，但已經部署到所有 Pod。結果：
1. 使用者大量反應問題
2. 緊急 rollback
3. 但 rollback 也要時間，期間服務持續出錯
4. 老闆很生氣，客戶很不滿

需要更安全的部署方式！

> 使用版本：Kubernetes 1.13, Istio 1.0.6

## 部署策略概覽

常見的部署策略：

| 策略 | 說明 | 優點 | 缺點 |
|------|------|------|------|
| **Recreate** | 停掉舊版，啟動新版 | 簡單 | 有停機時間 |
| **Rolling Update** | 逐步替換 Pod | 無停機 | 回滾慢 |
| **Blue-Green** | 兩套環境切換 | 快速切換、回滾 | 資源翻倍 |
| **Canary** | 小流量測試，逐步放大 | 風險低 | 複雜 |
| **A/B Testing** | 依條件分流 | 可測試多版本 | 需要額外基礎設施 |

## Recreate 部署

最簡單但有停機時間。

`deployment-recreate.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:v1
        ports:
        - containerPort: 8080
```

更新時：
```bash
kubectl set image deployment/myapp myapp=myapp:v2
```

Kubernetes 會：
1. 刪除所有 v1 Pod
2. 等待全部終止
3. 建立 v2 Pod

期間服務完全停止。**不適合正式環境**。

## Rolling Update 部署

Kubernetes 預設策略，逐步替換 Pod。

`deployment-rolling.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # 最多多 2 個 Pod
      maxUnavailable: 1    # 最多有 1 個 Pod 不可用
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:v1
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

參數說明：
- `maxSurge: 2`：更新時最多可以有 12 個 Pod（10 + 2）
- `maxUnavailable: 1`：更新時最少要有 9 個 Pod 可用（10 - 1）

更新流程：
```bash
kubectl set image deployment/myapp myapp=myapp:v2
```

Kubernetes 會：
1. 建立 2 個 v2 Pod（maxSurge）
2. 等待 v2 Pod ready
3. 終止 2 個 v1 Pod
4. 繼續建立 v2、終止 v1
5. 直到全部替換完成

觀察過程：
```bash
kubectl rollout status deployment/myapp

# Waiting for deployment "myapp" rollout to finish: 2 out of 10 new replicas have been updated...
# Waiting for deployment "myapp" rollout to finish: 2 out of 10 new replicas have been updated...
# Waiting for deployment "myapp" rollout to finish: 5 out of 10 new replicas have been updated...
# Waiting for deployment "myapp" rollout to finish: 8 out of 10 new replicas have been updated...
# Waiting for deployment "myapp" rollout to finish: 9 out of 10 new replicas have been updated...
# deployment "myapp" successfully rolled out
```

期間服務持續可用，流量同時打到 v1 和 v2。

### 暫停和恢復

```bash
# 暫停部署（觀察狀況）
kubectl rollout pause deployment/myapp

# 確認沒問題，繼續
kubectl rollout resume deployment/myapp
```

### 回滾

發現問題，立刻回滾：
```bash
kubectl rollout undo deployment/myapp

# 回滾到特定版本
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=2
```

**優點**：
- 無停機
- 簡單

**缺點**：
- 回滾需要時間
- v1 和 v2 同時存在期間，可能有相容性問題
- 不能控制流量比例

## Blue-Green 部署

維護兩套完整環境（Blue 和 Green），切換時瞬間完成。

### 架構

```
             Service (selector: version=blue)
                      ↓
  Blue (v1):  Pod Pod Pod
  Green (v2): Pod Pod Pod  (待命)
```

切換時改變 Service 的 selector。

### 實作

`deployment-blue.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:v1
        ports:
        - containerPort: 8080
```

`deployment-green.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v2
        ports:
        - containerPort: 8080
```

`service.yaml`：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue    # 目前流量打到 Blue
  ports:
  - port: 80
    targetPort: 8080
```

部署流程：

1. **部署 Blue（v1）**
```bash
kubectl apply -f deployment-blue.yaml
kubectl apply -f service.yaml
```

2. **部署 Green（v2）**
```bash
kubectl apply -f deployment-green.yaml
```

Green 已經啟動，但沒有流量。

3. **測試 Green**
```bash
# 建立測試 Service 直接連 Green
kubectl expose deployment myapp-green --name=myapp-green-test --port=8080

kubectl run test --image=busybox --rm -it -- sh
wget -q -O- myapp-green-test:8080
```

確認 Green 正常。

4. **切換流量到 Green**
```bash
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'
```

**瞬間**切換，所有流量立刻打到 Green！

5. **驗證**
```bash
curl http://myapp-service/version
# v2
```

6. **出問題？立刻切回 Blue**
```bash
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

一秒回滾！

7. **確認沒問題，刪除 Blue**
```bash
kubectl delete deployment myapp-blue
```

下次更新，Green 變 Blue，新版本變 Green，循環使用。

**優點**：
- 瞬間切換
- 瞬間回滾
- 新版本可以充分測試

**缺點**：
- 需要雙倍資源
- 資料庫遷移複雜（兩版本可能 schema 不同）

## Canary 部署

小流量測試新版本，逐步增加流量，風險最低。

### 使用 Kubernetes

利用 Deployment 的 replicas 控制流量比例。

`deployment-v1.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:v1
        ports:
        - containerPort: 8080
```

`deployment-v2.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: myapp:v2
        ports:
        - containerPort: 8080
```

`service.yaml`：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp    # 沒有指定 version，會打到 v1 和 v2
  ports:
  - port: 80
    targetPort: 8080
```

部署：
```bash
kubectl apply -f deployment-v1.yaml
kubectl apply -f deployment-v2.yaml
kubectl apply -f service.yaml
```

流量分配：
- v1: 9 個 Pod，90% 流量
- v2: 1 個 Pod，10% 流量

觀察 v2 的指標（錯誤率、延遲），沒問題就增加流量：

```bash
# 增加 v2 到 50%
kubectl scale deployment myapp-v1 --replicas=5
kubectl scale deployment myapp-v2 --replicas=5

# 再增加到 90%
kubectl scale deployment myapp-v1 --replicas=1
kubectl scale deployment myapp-v2 --replicas=9

# 全部切到 v2
kubectl scale deployment myapp-v1 --replicas=0
kubectl scale deployment myapp-v2 --replicas=10
```

**問題**：流量分配不精確，依賴 Pod 數量。

### 使用 Istio（推薦）

Istio 可以精確控制流量比例。

`destination-rule.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

`virtual-service-canary-10.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
```

套用：
```bash
kubectl apply -f destination-rule.yaml
kubectl apply -f virtual-service-canary-10.yaml
```

精確的 90/10 分流，不管 Pod 數量！

逐步增加 v2 流量：

`virtual-service-canary-50.yaml`：
```yaml
http:
- route:
  - destination:
      host: myapp
      subset: v1
    weight: 50
  - destination:
      host: myapp
      subset: v2
    weight: 50
```

```bash
kubectl apply -f virtual-service-canary-50.yaml
```

最後全部切到 v2：

`virtual-service-canary-100.yaml`：
```yaml
http:
- route:
  - destination:
      host: myapp
      subset: v2
    weight: 100
```

### 自動化 Canary

使用 Flagger 自動化 Canary 流程：

安裝 Flagger：
```bash
kubectl apply -k github.com/weaveworks/flagger//kustomize/istio
```

定義 Canary：

`canary.yaml`：
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
    targetPort: 8080
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
```

Flagger 會：
1. 部署新版本（金絲雀）
2. 給 10% 流量
3. 檢查指標（成功率、延遲）
4. 如果正常，每分鐘增加 10% 流量
5. 最多到 50%
6. 最終全部切換
7. 如果指標異常，自動回滾

完全自動化！

## A/B Testing

根據使用者特徵分流，測試不同版本的效果。

`virtual-service-ab.yaml`：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  # iOS 使用者去 v2
  - match:
    - headers:
        user-agent:
          regex: ".*iPhone.*"
    route:
    - destination:
        host: myapp
        subset: v2
  # Chrome 使用者去 v2
  - match:
    - headers:
        user-agent:
          regex: ".*Chrome.*"
    route:
    - destination:
        host: myapp
        subset: v2
  # 其他去 v1
  - route:
    - destination:
        host: myapp
        subset: v1
```

或根據 Header：

```yaml
http:
- match:
  - headers:
      x-user-group:
        exact: beta-testers
  route:
  - destination:
      host: myapp
      subset: v2
- route:
  - destination:
      host: myapp
      subset: v1
```

## 實際案例

### 案例一：電商促銷活動

去年雙十一前一天，我們要上線新的購物車功能。

決策：使用 Blue-Green 部署。

原因：
- 高流量期間，不能出錯
- 需要瞬間回滾能力
- 有足夠資源支撐雙倍 Pod

流程：
1. 提前 2 天部署 Green（新版本）
2. 測試團隊測試 Green
3. 活動前 1 小時切換到 Green
4. 監控所有指標
5. 活動順利，沒出問題！

### 案例二：推薦演算法升級

要上線新的推薦演算法，但不確定效果。

決策：使用 A/B Testing。

原因：
- 需要比較兩個版本的業務指標（點擊率、轉換率）
- 不是所有使用者都適合新演算法

流程：
1. 隨機 20% 使用者看新演算法
2. 80% 使用者看舊演算法
3. 追蹤業務指標 2 週
4. 分析結果：新演算法點擊率提升 15%！
5. 逐步增加到 100%

### 案例三：API 重構

後端 API 從 v1 重構到 v2，結構完全不同。

決策：使用 Canary 部署。

原因：
- 大幅度改動，風險高
- 需要逐步驗證
- 可能有未預見的問題

流程：
1. 部署 v2，給 5% 流量
2. 監控錯誤率、延遲
3. 發現 v2 有個邊緣案例會出錯
4. 修復，重新部署 v2
5. 再給 10% 流量
6. 逐步增加到 100%
7. 整個過程 1 週，安全上線

## 選擇部署策略

| 情境 | 推薦策略 |
|------|----------|
| 小改動（bug fix） | Rolling Update |
| 中等改動 | Canary（Istio） |
| 大改動、重構 | Canary（慢慢來） |
| 高流量期間 | Blue-Green（快速回滾） |
| 測試業務指標 | A/B Testing |
| 內部工具 | Recreate 或 Rolling |

## 遇到的問題

### 問題一：資料庫 Schema 變更

Blue-Green 部署時，v1 和 v2 的資料庫 schema 不同。

解決方法：
1. **向後相容的遷移**：先加欄位，不刪舊欄位
2. **分階段遷移**：
   - 階段一：加新欄位，兩邊都寫
   - 階段二：切換到新版本
   - 階段三：刪除舊欄位

### 問題二：Session 問題

Rolling Update 時，使用者的 Session 可能在舊 Pod，但下個請求打到新 Pod。

解決方法：
1. **外部 Session Store**：用 Redis 存 Session
2. **Sticky Session**：Istio 設定 consistent hash
3. **無狀態設計**：用 JWT，不依賴 Session

### 問題三：Canary 時如何追蹤指標

需要區分 v1 和 v2 的指標。

解決：
1. 應用加 version label
2. Prometheus 區分不同版本的指標
3. Grafana 儀表板顯示兩個版本的對比

## 心得

以前我們只用 Rolling Update，每次上線都提心吊膽。有次線上出問題，回滾花了 10 分鐘，那 10 分鐘感覺像 10 小時。

現在有了這些部署策略，上線輕鬆多了：

- **日常更新**：Canary，慢慢來，安全第一
- **緊急修復**：Blue-Green，快速上線，快速回滾
- **新功能測試**：A/B Testing，數據說話

搭配 Istio，部署策略完全不用改程式碼，改個 YAML 就好。這才是雲原生的威力！

下個月要研究更進階的主題：API Gateway、分散式追蹤、配置中心等等。DevOps 的路還很長，但每週都在進步，很有成就感。

下週計畫研究 API Gateway，統一管理所有微服務的入口。
