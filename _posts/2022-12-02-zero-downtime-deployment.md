---
layout: post
title: "首個微服務遷移與零停機部署"
date: 2022-12-02 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Blue-Green Deployment, Zero Downtime, Migration]
---

進入十二月,是時候進行「實戰檢驗」了。今天要選擇一個真實的微服務,從傳統 Jenkins 部署完整遷移到 GitOps,並實現**零停機部署 (Zero Downtime Deployment)**。

這不只是技術驗證,更是向團隊展示「GitOps 能解決實際問題」的最好機會。

---

## 本週目標

完成第一個微服務的 GitOps 遷移,驗證零停機部署可行性。

---

## 技術實作重點

### 1. 選擇遷移目標:User Service

**為什麼選它?**

- ✅ 業務重要性中等 (不是核心支付服務,但也不是邊緣功能)
- ✅ 有完善的測試覆蓋 (可以驗證部署是否成功)
- ✅ 流量穩定 (不會突然暴增,便於觀察)
- ✅ 技術棧成熟 (Spring Boot + MySQL,沒有特殊依賴)

**服務規格:**

```
Language: Java 17 + Spring Boot 2.7
Database: MySQL 8.0
Current Deployment: Jenkins Pipeline (imperative kubectl apply)
Traffic: ~1000 req/min
SLA: 99.9%
```

### 2. 零停機部署的挑戰

**問題場景:**

```
1. 部署新版本 (v1.2.4)
2. 舊 Pod (v1.2.3) 立刻被終止
3. 新 Pod 尚未完全啟動 (需要 30 秒)
4. 這 30 秒內,流量無處可去 → 服務中斷
```

**零停機的關鍵要素:**

1. **Readiness Probe:** 確保 Pod 完全啟動後才接收流量
2. **Rolling Update:** 逐步替換 Pod,而非一次全部
3. **Pre-Stop Hook:** 優雅關閉,確保正在處理的請求完成
4. **Connection Draining:** Load Balancer 停止轉發新請求給舊 Pod

### 3. 準備 Helm Chart

#### values-prod.yaml

```yaml
replicaCount: 3  # 生產環境至少 3 個 Pod

image:
  repository: myregistry.azurecr.io/user-service
  tag: v1.2.3
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

# 🔹 健康檢查配置 (關鍵!)
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

# 🔹 Rolling Update 策略
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # 最多多 1 個 Pod (總共 4 個)
    maxUnavailable: 0  # 確保至少 3 個 Pod 隨時可用

# 🔹 優雅關閉
lifecycle:
  preStop:
    exec:
      command:
      - sh
      - -c
      - sleep 15  # 等待 Load Balancer 停止轉發流量
terminationGracePeriodSeconds: 30

# 環境變數
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "production"
  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: host
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: password
```

#### templates/deployment.yaml

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: {{ .Values.strategy.type }}
    rollingUpdate:
      maxSurge: {{ .Values.strategy.rollingUpdate.maxSurge }}
      maxUnavailable: {{ .Values.strategy.rollingUpdate.maxUnavailable }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
        version: {{ .Values.image.tag }}
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        
        # 健康檢查
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 10 }}
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 10 }}
        
        # 生命週期
        lifecycle:
          {{- toYaml .Values.lifecycle | nindent 10 }}
        
        # 資源限制
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        
        # 環境變數
        env:
          {{- toYaml .Values.env | nindent 10 }}
      
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
```
{% endraw %}

### 4. 遷移步驟

#### Phase 1: 建立 ArgoCD Application (不啟用自動同步)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-service-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/helm-charts.git
    targetRevision: main
    path: user-service
    helm:
      valueFiles:
      - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    # ⚠️ 先不啟用自動同步,手動控制
    syncOptions:
    - CreateNamespace=true
```

#### Phase 2: 驗證 Diff

```bash
argocd app diff user-service-prod

# 確認 ArgoCD 產生的 Manifest 與現有的一致
```

#### Phase 3: 手動 Sync (週五晚上,低流量時段)

```bash
# 先 Dry-run
argocd app sync user-service-prod --dry-run

# 確認無誤後,正式 Sync
argocd app sync user-service-prod
```

#### Phase 4: 監控部署過程

**Terminal 1: 監控 Pod 狀態**

```bash
watch -n 1 'kubectl get pods -n production -l app=user-service'

# 輸出:
# NAME                            READY   STATUS    AGE
# user-service-7d8f9c-xxx         1/1     Running   10m  ← 舊版本
# user-service-7d8f9c-yyy         1/1     Running   10m
# user-service-7d8f9c-zzz         1/1     Running   10m
# user-service-9a1b2c-aaa         0/1     Running   5s   ← 新版本啟動中
```

**Terminal 2: 監控流量**

```bash
# 查看 Service Endpoints
kubectl get endpoints user-service-svc -n production -w

# 查看請求成功率
curl -s http://user-service-prod/actuator/metrics/http.server.requests | jq
```

**Terminal 3: ArgoCD UI**

觀察 Resource Tree,確保所有資源 Healthy。

### 5. Rolling Update 過程解析

```
時間軸:

T+0s:  ArgoCD 執行 Sync
       3 個舊 Pod (v1.2.3) Running
       開始創建 1 個新 Pod (v1.2.4)

T+30s: 新 Pod 通過 Readiness Probe
       Service Endpoints 加入新 Pod
       現在有 4 個 Pod 接收流量 (3舊1新)

T+35s: 開始終止 1 個舊 Pod
       執行 preStop Hook (sleep 15s)
       Service Endpoints 移除該 Pod

T+50s: 舊 Pod 優雅關閉完成
       現在有 3 個 Pod (2舊1新)

T+80s: 第 2 個新 Pod 啟動完成
       終止第 2 個舊 Pod

T+130s: 第 3 個新 Pod 啟動完成
        終止最後 1 個舊 Pod

T+160s: 部署完成
        3 個新 Pod (v1.2.4) Running
        舊 Pod 全部優雅關閉
```

**整個過程中,至少有 3 個 Pod 隨時可用 → 零停機!**

### 6. 驗證零停機

**測試腳本:**

```bash
#!/bin/bash
# load-test.sh

START_TIME=$(date +%s)
SUCCESS_COUNT=0
FAIL_COUNT=0

echo "Starting load test... (Press Ctrl+C to stop)"

while true; do
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://user-service-prod/api/users/1)
    
    if [ "$RESPONSE" -eq 200 ]; then
        ((SUCCESS_COUNT++))
    else
        ((FAIL_COUNT++))
        echo "❌ Failed request at $(date): HTTP $RESPONSE"
    fi
    
    ELAPSED=$(($(date +%s) - START_TIME))
    TOTAL=$((SUCCESS_COUNT + FAIL_COUNT))
    SUCCESS_RATE=$(echo "scale=2; $SUCCESS_COUNT / $TOTAL * 100" | bc)
    
    echo -ne "\r⏱️  Elapsed: ${ELAPSED}s | ✅ Success: $SUCCESS_COUNT | ❌ Failed: $FAIL_COUNT | 📊 Rate: ${SUCCESS_RATE}%"
    
    sleep 0.1
done
```

**執行測試:**

```bash
# 在部署開始前,啟動測試
./load-test.sh

# 同時在另一個 Terminal 執行部署
argocd app sync user-service-prod
```

**預期結果:**

```
⏱️  Elapsed: 180s | ✅ Success: 18000 | ❌ Failed: 0 | 📊 Rate: 100.00%
```

**✅ 零停機部署成功!**

---

## 遇到的挑戰與對策

### 挑戰一:Pod 啟動慢,導致短暫503

**現象:**

新 Pod 狀態顯示 `Running`,但請求仍然失敗。

**根因:**

Readiness Probe 配置太早,Pod 內的 Spring Boot 應用還沒完全啟動。

**對策:**

調整 Readiness Probe:

```yaml
readinessProbe:
  initialDelaySeconds: 60  # 從 30s 改為 60s
  periodSeconds: 5
```

### 挑戰二:資料庫連線池耗盡

**現象:**

部署過程中,新 Pod 無法連接到 MySQL。

**根因:**

舊 Pod 和新 Pod 同時存在,總連線數超過 MySQL 的 `max_connections`。

**對策:**

**方式一:調整 MySQL 配置**

```sql
SET GLOBAL max_connections = 200;  -- 從 100 提高到 200
```

**方式二:調整 Spring Boot 連線池**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10  # 每個 Pod 最多 10 個連線
```

### 挑戰三:如何回滾?

部署完成後,發現新版本有 Bug。

**對策:**

```bash
# 方式一:透過 ArgoCD 回滾
argocd app rollback user-service-prod <history-id>

# 方式二:透過 Git Revert
cd helm-charts/user-service
git revert HEAD
git push

# ArgoCD 自動同步回舊版本
```

---

## 給團隊的觀念分享

### 零停機不是「炫技」,是「責任」

在傳統部署中,我們經常說「晚上維護窗口部署」。但這意味著:

- 用戶在這段時間無法使用服務
- 工程師要熬夜
- 如果出問題,回滾時間更長

**零停機部署讓我們能:**

- 白天部署 (問題能立刻發現)
- 隨時回滾 (不影響用戶)
- 提高部署頻率 (從每週一次到每天多次)

**這不只是技術提升,更是服務品質的提升。**

### 健康檢查是「生命線」

很多團隊忽略 Readiness Probe,導致:

- Pod 狀態是 Running,但應用還沒啟動
- Load Balancer 轉發流量到「殭屍 Pod」
- 用戶看到 502/503

**正確的健康檢查,是零停機部署的基礎。**

---

## 下週預告

首個微服務遷移成功後,下週要制定 **團隊規範:GitOps Workflow**。

我們要建立:

- Merge Request 審核機制
- 部署流程標準化 (Dev → Staging → Prod)
- 變更管理規範 (Change Management)

讓 GitOps 從「個人實驗」變成「團隊標準」。

---

**作者的話:**  
這週最激動人心的時刻,是看到 Load Test 腳本顯示「18000 個請求,0 個失敗」。那種「理論成為現實」的感覺,無法言喻。零停機部署,不再是遙不可及的夢想。

**Tags:** #Zero-Downtime #Blue-Green #Rolling-Update #Migration #GitOps
