---
layout: post
title: "App-of-Apps 模式 (資深工程師必學)"
date: 2022-10-14 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, App-of-Apps, Cluster Bootstrap, Declarative]
---

今天要介紹 ArgoCD 最優雅的設計模式:**App-of-Apps (應用中的應用)**。

這個模式讓你能用一個 Application 來管理其他 Application,實現「聲明式的應用管理」。更重要的是,它能解決一個實際痛點:**如何快速初始化一個全新的 Kubernetes 集群?**

想像這個場景:災難發生,你需要在 30 分鐘內重建整個生產環境。有了 App-of-Apps,你只需要執行一條指令,其他一切都會自動部署。這就是 Infrastructure as Code 的極致。

---

## 本週目標

理解並實作 App-of-Apps 模式,實現集群的聲明式初始化。

---

## 技術實作重點

### 1. 什麼是 App-of-Apps?

**傳統做法:**

```bash
# 手動建立 50 個 Application
argocd app create app-a-prod ...
argocd app create app-b-prod ...
argocd app create app-c-prod ...
...
```

**App-of-Apps 做法:**

```bash
# 只建立一個「根 Application」
argocd app create bootstrap-prod ...

# 它會自動創建其他 50 個 Application
```

**原理:**

ArgoCD Application 本身就是一個 Kubernetes CRD (Custom Resource),所以可以用 **YAML 來描述 Application**,然後讓一個 Application 去管理這些 YAML。

### 2. Repository 結構設計

```
k8s-manifests/
├── bootstrap/
│   └── prod/
│       ├── app-a.yaml          # Application CRD
│       ├── app-b.yaml
│       ├── ingress-nginx.yaml  # Infrastructure
│       └── cert-manager.yaml
├── apps/
│   ├── app-a/
│   │   └── ... (Helm Chart or YAML)
│   └── app-b/
│       └── ...
└── infra/
    ├── ingress-nginx/
    └── cert-manager/
```

**概念:**

- `bootstrap/prod/` 存放 Application CRD 定義
- `apps/` 和 `infra/` 存放實際的 Manifest

### 3. 建立 Application CRD

#### bootstrap/prod/app-a.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-prod
  namespace: argocd
  # 🔹 關鍵:設定 Finalizer,確保刪除時也會清理資源
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: apps/app-a
    helm:
      valueFiles:
      - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

#### bootstrap/prod/ingress-nginx.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx-prod
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/kubernetes/ingress-nginx.git
    targetRevision: helm-chart-4.3.0
    path: charts/ingress-nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 4. 建立「根 Application」

**方式一:透過 CLI**

```bash
argocd app create bootstrap-prod \
  --repo https://github.com/my-org/k8s-manifests.git \
  --path bootstrap/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

**方式二:透過 YAML**

```yaml
# bootstrap-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: bootstrap/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd  # 注意:部署到 argocd namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**部署:**

```bash
kubectl apply -f bootstrap-prod.yaml
```

### 5. 觀察神奇的自動化

執行完後,觀察 ArgoCD UI 或 CLI:

```bash
argocd app list

# 輸出:
# NAME                  ...  STATUS
# bootstrap-prod        ...  Synced
# app-a-prod            ...  Syncing  ← 自動創建!
# app-b-prod            ...  Syncing
# ingress-nginx-prod    ...  Syncing
# cert-manager-prod     ...  Syncing
```

**完整流程:**

```
1. 你創建 bootstrap-prod Application
2. ArgoCD 從 Git 讀取 bootstrap/prod/*.yaml
3. 發現裡面都是 Application CRD
4. 自動在 K8s 中創建這些 Application
5. 每個 Application 再去同步自己的 Manifest
```

### 6. 進階:應用依賴管理

某些應用有啟動順序要求,例如:

```
cert-manager (先啟動)
  ↓
ingress-nginx (需要 cert-manager 的 CRD)
  ↓
app-a (需要 Ingress)
```

**使用 Sync Waves 控制順序:**

```yaml
# bootstrap/prod/cert-manager.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager-prod
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # 第一批同步
spec:
  ...
```

```yaml
# bootstrap/prod/ingress-nginx.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx-prod
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # 第二批同步
spec:
  ...
```

```yaml
# bootstrap/prod/app-a.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-prod
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"  # 第三批同步
spec:
  ...
```

**ArgoCD 會按 wave 順序部署,等前一個 wave 全部 Synced 才繼續。**

### 7. 實戰案例:30 分鐘重建集群

假設災難發生,我們需要重建整個 Production 環境:

```bash
# 步驟 1: 建立新的 K8s 集群 (Azure AKS)
az aks create --resource-group prod-rg --name prod-cluster ...

# 步驟 2: 安裝 ArgoCD
helm install argocd argo/argo-cd -n argocd --create-namespace

# 步驟 3: 部署 Bootstrap Application
kubectl apply -f bootstrap-prod.yaml

# 步驟 4: 等待所有應用同步完成
argocd app wait bootstrap-prod --sync --health --timeout 600

# ✅ 完成!所有應用自動部署完畢
```

**從空集群到完整環境,只需要 4 條指令 + 15 分鐘。**

---

## 遇到的挑戰與對策

### 挑戰一:循環依賴問題

如果 `bootstrap-prod` 管理的 Application 中,又包含了 `bootstrap-prod` 本身,會產生循環依賴。

**對策:**

確保 `bootstrap/prod/` 目錄中**不包含** `bootstrap-prod.yaml` 本身。

### 挑戰二:如何管理 Bootstrap Application?

Bootstrap Application 本身是手動創建的,如何把它也納入 GitOps?

**對策:Bootstrap of Bootstrap**

```yaml
# bootstrap-of-bootstrap.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap-prod
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    path: bootstrap-apps/prod  # 新目錄
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: {}
```

**但這引入了「雞生蛋」問題:誰來創建 bootstrap-of-bootstrap?**

**實務上:**

- Bootstrap Application 可以手動創建 (災難時才需要)
- 其他 Application 全部由 Bootstrap 管理
- 定期備份 Bootstrap Application 的 YAML

### 挑戰三:Application 太多導致 UI 混亂

當有 100 個 Application 時,ArgoCD UI 會很亂。

**對策:使用 Project 分類**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: microservices
  namespace: argocd
spec:
  description: All microservice applications
  sourceRepos:
  - https://github.com/my-org/k8s-manifests.git
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
```

然後在 Application 中指定:

```yaml
spec:
  project: microservices  # 而非 default
```

---

## 給團隊的觀念分享

### GitOps 的終極形態

App-of-Apps 讓我們實現了「完全聲明式的基礎設施」:

```
Git Repository (Single Source of Truth)
  ↓
Bootstrap Application (Entry Point)
  ↓
All Applications (Automated)
  ↓
Entire Cluster State (Reconciled)
```

**這意味著:**

- 集群的所有配置都在 Git 中
- 任何人都能重現這個環境
- 災難恢復只需要幾條指令

**這就是 Infrastructure as Code 的極致。**

### 不要過度設計

雖然 App-of-Apps 很強大,但不是所有團隊都需要。

**適合的場景:**

- ✅ 需要管理多個集群 (Dev/Staging/Prod)
- ✅ 應用數量超過 20 個
- ✅ 需要快速初始化集群

**不適合的場景:**

- ❌ 只有 5 個應用
- ❌ 團隊不熟悉 ArgoCD
- ❌ 應用之間沒有依賴關係

**記住:簡單的架構,往往更可靠。**

---

## 下週預告

App-of-Apps 解決了「如何管理多個應用」,但下週要面對另一個問題:**如何管理多環境的差異?**

主題:**多環境管理策略 (Dev/Staging/Production)**。

我們要探討:

- 用 Kustomize Overlays 管理環境差異
- 如何避免配置重複
- 如何確保 Production 配置不會被誤改

這是 GitOps 在企業落地的關鍵一環。

---

**作者的話:**  
這週實作 App-of-Apps 時,有種「打通任督二脈」的感覺。當看到一條指令就能部署整個集群時,那種成就感無法言喻。這不只是技術,更是一種思維方式:把複雜的系統,用聲明式的方式管理。

**Tags:** #ArgoCD #App-of-Apps #Cluster-Bootstrap #Declarative #IaC
