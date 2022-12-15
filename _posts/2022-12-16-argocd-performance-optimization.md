---
layout: post
title: "效能優化:調校 ArgoCD Controller"
date: 2022-12-16 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Performance, Optimization, Kubernetes, Controller]
---

隨著應用數量增加,我們發現 **ArgoCD 的效能開始下降**:同步變慢、UI 卡頓、Controller CPU 飆高。

這週要深入探討 **ArgoCD 效能優化**,包括 Controller 資源配置、Repository 大小控制、Sync 策略優化。讓 ArgoCD 能夠優雅地處理大規模部署。

---

## 本週目標

優化 ArgoCD 效能,確保能穩定管理 100+ 應用。

---

## 技術實作重點

### 1. 效能瓶頸分析

#### 當前狀態

- **管理的應用數量:** 120 個
- **Manifest Repository 大小:** 450 MB
- **Controller CPU 使用率:** 80% (2 Core)
- **Controller Memory 使用率:** 60% (4 GB)
- **平均 Sync 時間:** 45 秒 (原本只要 10 秒)
- **UI 載入時間:** 8 秒 (原本 2 秒)

#### 問題定位

**使用 Prometheus + Grafana 監控 ArgoCD:**

```bash
# 1. 檢查 ArgoCD Controller 的關鍵指標
kubectl port-forward -n argocd svc/argocd-metrics 8082:8082

# 2. 查看 Prometheus Metrics
curl http://localhost:8082/metrics | grep argocd_app
```

**關鍵指標:**

```
# 應用同步延遲
argocd_app_reconcile_duration_seconds_bucket

# Git 拉取時間
argocd_git_request_duration_seconds_bucket

# Controller 協調次數
argocd_app_reconcile_count
```

**發現問題:**

1. **Git Clone 時間過長** (Repository 太大)
2. **App Reconcile 頻率過高** (預設 3 分鐘一次,120 個應用 = 每秒 0.67 次)
3. **Controller 資源不足** (2 Core 不夠用)

---

### 2. 優化策略一:調整 Controller 資源

#### 增加 Controller 的 CPU 和 Memory

```yaml
# argocd/values.yaml (Helm)
controller:
  replicas: 1
  resources:
    requests:
      cpu: 4000m      # 從 2 Core → 4 Core
      memory: 8Gi     # 從 4 GB → 8 GB
    limits:
      cpu: 8000m      # 允許 Burst 到 8 Core
      memory: 12Gi
  
  # 調整 Reconcile 參數
  env:
    - name: ARGOCD_RECONCILIATION_TIMEOUT
      value: "180s"   # 單次 Reconcile 超時時間 (預設 30s)
    - name: ARGOCD_REPO_SERVER_TIMEOUT_SECONDS
      value: "120"    # Git 請求超時時間 (預設 60s)
```

**效果:**

- ✅ Controller CPU 使用率降到 40%
- ✅ Sync 時間從 45 秒降到 25 秒

---

### 3. 優化策略二:調整 Reconcile 頻率

#### 減少不必要的 Git Polling

**預設行為:**

```
ArgoCD 每 3 分鐘檢查一次所有 Application 的 Git Repository 是否有變更
120 個應用 × 每 3 分鐘 1 次 = 每秒 0.67 次 Git 請求
```

**優化:**

```yaml
# Application 層級:關閉 Auto-Sync 的頻繁檢查
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-prod
spec:
  source:
    repoURL: https://github.com/myorg/manifests.git
    targetRevision: main
    path: apps/app-a/overlays/prod
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  
  # 調整 Reconcile 頻率 (新增)
  info:
    - name: refresh
      value: "10m"  # 從 3 分鐘改成 10 分鐘
```

**或全域設定:**

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # 預設 Reconcile 間隔
  timeout.reconciliation: 10m
  
  # Git 請求 Cache 時間
  timeout.reconciliation.jitter: 1m
```

**效果:**

- ✅ Git 請求從每秒 0.67 次降到 0.2 次
- ✅ Repository Server CPU 使用率降低 50%

---

### 4. 優化策略三:Repository Cache 優化

#### 問題:每次 Sync 都要 Clone 整個 Repository

**預設行為:**

```bash
# 每次 Sync 都執行:
git clone https://github.com/myorg/manifests.git /tmp/xxx
```

**優化:啟用 Git Cache**

```yaml
# argocd/values.yaml
repoServer:
  replicas: 2
  resources:
    requests:
      cpu: 2000m
      memory: 4Gi
    limits:
      cpu: 4000m
      memory: 8Gi
  
  # 啟用 Cache
  volumeMounts:
    - name: git-cache
      mountPath: /git-cache
  volumes:
    - name: git-cache
      emptyDir: {}
  
  env:
    - name: ARGOCD_GIT_MODULES_ENABLED
      value: "false"  # 禁用 Git Submodules (加速)
    - name: XDG_CACHE_HOME
      value: /git-cache
```

**啟用 Repository Server 的 Cache 後:**

```
首次 Clone:3 分鐘
後續 Pull:10 秒 (只拉差異)
```

**效果:**

- ✅ Git Clone 時間從 3 分鐘降到 10 秒
- ✅ Repository Server 負載降低 70%

---

### 5. 優化策略四:拆分 Manifest Repository

#### 問題:單一 Repository 過大

**當前結構:**

```
manifests/ (450 MB)
├── apps/
│   ├── app-a/
│   ├── app-b/
│   ├── ...
│   └── app-z/ (120 個應用)
├── helm-charts/
└── base/
```

**優化:按團隊拆分 Repository**

```
manifests-team-a/ (80 MB)
├── apps/
│   ├── app-a/
│   ├── app-b/
│   └── app-c/

manifests-team-b/ (90 MB)
├── apps/
│   ├── app-x/
│   ├── app-y/
│   └── app-z/

manifests-infra/ (60 MB)
├── ingress-nginx/
├── cert-manager/
└── monitoring/
```

**優點:**

- ✅ 每個 Repository 都更小,Clone 更快
- ✅ 團隊可以獨立管理自己的 Repository
- ✅ 減少 Merge 衝突

**缺點:**

- ❌ 管理複雜度增加
- ❌ 跨團隊的依賴關係需要額外處理

---

### 6. 優化策略五:Manifest 結構優化

#### 問題:過多的 YAML 檔案

**當前狀態:**

```
apps/app-a/overlays/prod/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── configmap.yaml
├── secret.yaml
├── hpa.yaml
├── pdb.yaml
├── servicemonitor.yaml
└── networkpolicy.yaml  # 9 個檔案
```

**優化:使用 Helm Chart (單一 values.yaml)**

```
helm-charts/app-a/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
└── values-prod.yaml  # 只需要維護這一個檔案
```

**或使用 Kustomize Components:**

```
apps/app-a/base/
└── all-in-one.yaml  # 所有資源在一個檔案 (用 --- 分隔)

apps/app-a/overlays/prod/
└── kustomization.yaml
    resources:
      - ../../base
    patches:
      - target:
          kind: Deployment
        patch: |-
          - op: replace
            path: /spec/replicas
            value: 5
```

**效果:**

- ✅ 檔案數量減少 90%
- ✅ Git 操作更快
- ✅ Kustomize Build 時間減少 50%

---

### 7. 優化策略六:Application Set 優化

#### 問題:過多的 Application CR

**當前狀態:**

```bash
kubectl get applications -n argocd | wc -l
# 120 個 Application
```

**優化:使用 ApplicationSet 合併相似應用**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-a-apps
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - app: app-a
            env: prod
          - app: app-b
            env: prod
          - app: app-c
            env: prod
  template:
    metadata:
      name: '{{app}}-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/manifests.git
        targetRevision: main
        path: 'apps/{{app}}/overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{app}}-{{env}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**效果:**

- ✅ 從 120 個 Application CR 減少到 10 個 ApplicationSet
- ✅ Controller 需要監聽的 CR 減少,效能提升
- ✅ 管理更簡單

---

### 8. 效能測試與監控

#### 壓力測試

```bash
# 1. 模擬大規模 Sync
for i in {1..120}; do
  argocd app sync app-$i-prod --async
done

# 2. 監控 Sync 時間
argocd app list --output json | jq -r '.[] | "\(.metadata.name) \(.status.operationState.finishedAt)"'
```

#### Grafana Dashboard

```yaml
# Prometheus Query:平均 Sync 時間
histogram_quantile(0.95, 
  rate(argocd_app_reconcile_duration_seconds_bucket[5m])
)

# Query:Git 請求成功率
rate(argocd_git_request_total{status_code="200"}[5m]) 
/ 
rate(argocd_git_request_total[5m])
```

---

## 效能優化總結

| 優化項目 | 優化前 | 優化後 | 提升 |
|---------|--------|--------|------|
| **Sync 時間** | 45 秒 | 12 秒 | 73% ⬇️ |
| **Controller CPU** | 80% | 30% | 62% ⬇️ |
| **UI 載入時間** | 8 秒 | 2 秒 | 75% ⬇️ |
| **Git Clone 時間** | 3 分鐘 | 10 秒 | 94% ⬇️ |
| **Application 數量** | 120 個 | 120 個 | - |

**關鍵優化:**

1. ✅ Controller 資源增加 (2C4G → 4C8G)
2. ✅ Reconcile 間隔調整 (3m → 10m)
3. ✅ Git Cache 啟用
4. ✅ Repository 拆分 (450MB → 3 個 80MB)
5. ✅ ApplicationSet 合併 (120 個 → 10 個)

---

## 遇到的挑戰與對策

### 挑戰一:優化後仍然偶爾卡頓

**問題:**

即使優化後,偶爾 UI 還是會卡住 10 秒。

**排查:**

```bash
# 檢查 API Server 日誌
kubectl logs -n argocd argocd-server-xxx -f

# 發現:某些大型 Application 的 Diff 計算很慢
GET /api/v1/applications/app-huge/diff - 15s
```

**原因:**

某個應用有 **5000 個 Kubernetes 資源**(因為用了 Helm Chart 產生大量 ServiceMonitor)。

**對策:**

```yaml
# 將這個應用拆分成多個小應用
app-huge-core/        # 100 個資源
app-huge-monitoring/  # 5000 個 ServiceMonitor (單獨管理)
```

### 挑戰二:Git Cache 導致「看不到最新變更」

**問題:**

啟用 Git Cache 後,有時推送到 Git 卻沒有觸發 Sync。

**原因:**

Cache 的 TTL 太長,需要手動刷新。

**對策:**

```bash
# 手動刷新 Repository Cache
argocd repo list
argocd repocreds list

# 或在 ArgoCD UI 中點擊 "REFRESH HARD"
```

**自動化:**

```yaml
# 使用 Webhook 觸發刷新
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  webhook.github.secret: "your-webhook-secret"
```

**GitHub Webhook 配置:**

```
URL: https://argocd.example.com/api/webhook
Content type: application/json
Secret: your-webhook-secret
Events: Just the push event
```

---

## 給團隊的觀念分享

### 不要過早優化

我們一開始只有 20 個應用時,完全不需要優化。但到了 100 個應用,優化變得必要。

**經驗法則:**

- **< 50 個應用:** 預設配置就夠用
- **50-200 個應用:** 需要調整 Controller 資源和 Reconcile 頻率
- **> 200 個應用:** 需要考慮拆分 Repository 或使用 ApplicationSet

### 監控先行

沒有監控,就不知道問題在哪。

**必備監控:**

- ✅ ArgoCD Metrics (Prometheus)
- ✅ Controller 資源使用率
- ✅ Git 請求延遲
- ✅ Application Sync 時間

**工具:**

- Prometheus + Grafana
- ArgoCD 官方 Dashboard: https://github.com/argoproj/argo-cd/blob/master/examples/dashboard.json

---

## 下週預告

經過五個月的旅程,下週將是 **年度回顧**。

主題內容:

- GitOps 轉型的關鍵成果
- Lead Time 和 Deployment Frequency 的變化
- 團隊反饋與教訓
- 2023 年的計劃

這是一個總結,也是新的開始。

---

**作者的話:**  
這週最大的挑戰是「找出真正的瓶頸」。很多時候我們以為是 X,結果發現是 Y。沒有監控,就像閉著眼睛開車。

**Tags:** #ArgoCD #Performance #Optimization #Kubernetes #Controller
