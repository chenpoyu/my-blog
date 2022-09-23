---
layout: post
title: "ArgoCD App 建立：初試 Sync Policy"
date: 2022-09-23 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Sync Policy, Prune, Self-Heal, Declarative]
---

上週我們打通了 CI 與 CD 的橋樑,這週要深入 ArgoCD 最核心的概念:**Sync Policy (同步策略)**。

很多人以為 ArgoCD 只是「自動執行 kubectl apply」,但它的價值遠不止於此。透過 Sync Policy,ArgoCD 能**主動維護** Kubernetes 的狀態,而不是被動地接受指令。這週我們要實驗三個關鍵功能:`Automated Sync`、`Prune`、`Self-Heal`,並理解它們如何確保「Git 就是唯一的真相來源」。

---

## 本週目標

理解並配置 ArgoCD Sync Policy,驗證 Prune 與 Self-Heal 的實際行為。

---

## 技術實作重點

### 1. Sync Policy 的三種模式

ArgoCD Application 的同步有三種觸發方式:

| 模式 | 觸發方式 | 使用時機 |
|------|----------|----------|
| **Manual Sync** | 手動執行 `argocd app sync` 或點擊 UI | 生產環境初期,需要人工確認 |
| **Automated Sync** | Git 有變更時自動同步 | 開發/測試環境,追求快速迭代 |
| **Sync Waves** | 按順序分批同步 (進階) | 複雜的應用啟動順序 |

**我們的策略:**

- **Dev 環境:** Automated Sync (快速反饋)
- **Staging 環境:** Automated Sync (接近生產的環境)
- **Production 環境:** Manual Sync (人工把關)

### 2. Automated Sync 配置

#### Application YAML 配置

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: dev/app-a
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  
  syncPolicy:
    automated:
      # 🔹 核心三要素
      prune: true       # 自動刪除 Git 中不存在的資源
      selfHeal: true    # 自動修復手動變更
      allowEmpty: false # 防止誤刪所有資源 (預設 false)
    
    # 🔹 同步選項
    syncOptions:
    - CreateNamespace=true    # 自動建立目標 Namespace
    - PruneLast=true          # 最後才刪除資源 (避免服務中斷)
    - ApplyOutOfSyncOnly=true # 只同步有變更的資源 (提升效能)
    
    # 🔹 重試策略
    retry:
      limit: 5        # 最多重試 5 次
      backoff:
        duration: 5s  # 初始等待 5 秒
        factor: 2     # 每次翻倍 (5s → 10s → 20s → 40s → 80s)
        maxDuration: 3m
```

#### 透過 CLI 建立

```bash
argocd app create app-a-dev \
  --repo https://github.com/my-org/k8s-manifests.git \
  --path dev/app-a \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true \
  --sync-option PruneLast=true
```

### 3. Prune:自動清理多餘資源

#### 什麼是 Prune?

當你從 Git 中刪除某個 Kubernetes 資源的 YAML 後,ArgoCD 會自動從集群中刪除對應的資源。

**實驗步驟:**

**Step 1: 建立初始 Manifest**

```yaml
# dev/app-a/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-a
  template:
    metadata:
      labels:
        app: app-a
    spec:
      containers:
      - name: app
        image: nginx:1.21

---
# dev/app-a/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-a-svc
  namespace: dev
spec:
  selector:
    app: app-a
  ports:
  - port: 80
    targetPort: 80

---
# dev/app-a/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-a-config
  namespace: dev
data:
  LOG_LEVEL: "debug"
```

Commit 並推送到 Git,等待 ArgoCD 同步。

**Step 2: 驗證資源創建**

```bash
kubectl get all,cm -n dev

# 輸出:
# NAME                         READY   STATUS    RESTARTS   AGE
# pod/app-a-xxx                1/1     Running   0          1m
# 
# NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# service/app-a-svc    ClusterIP   10.0.123.45    <none>        80/TCP    1m
# 
# NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/app-a   1/1     1            1           1m
# 
# NAME                       DATA   AGE
# configmap/app-a-config     1      1m
```

**Step 3: 從 Git 中刪除 ConfigMap**

```bash
cd k8s-manifests
rm dev/app-a/configmap.yaml
git add .
git commit -m "feat: remove unused configmap"
git push
```

**Step 4: 觀察 ArgoCD 自動 Prune**

```bash
# 等待 30 秒 (Webhook 觸發)
argocd app get app-a-dev

# 輸出:
# ...
# LAST SYNC: Succeeded
# ...
# Resource: ConfigMap/app-a-config  ← 已被刪除

# 驗證 K8s
kubectl get cm app-a-config -n dev
# Error from server (NotFound): configmaps "app-a-config" not found
```

**✅ Prune 成功!**

**為什麼重要?**

如果沒有 Prune,當你從 Git 刪除資源時,Kubernetes 中的資源會殘留,導致:

- 無用的資源浪費集群資源
- 難以追蹤「哪些資源是活躍的」
- 安全風險 (舊的 Secret 沒有被清除)

### 4. Self-Heal:自動修復手動變更

#### 什麼是 Self-Heal?

當有人手動修改 Kubernetes 資源時 (例如 `kubectl edit`),ArgoCD 會自動將其還原成 Git 中的版本。

**實驗步驟:**

**Step 1: 手動修改 Deployment**

```bash
kubectl scale deployment app-a -n dev --replicas=5
```

**Step 2: 觀察 ArgoCD UI**

幾秒後,ArgoCD UI 會顯示:

```
⚠️ OutOfSync
- Deployment/app-a: replicas mismatch (Git: 1, Live: 5)
```

**Step 3: Self-Heal 自動觸發**

等待約 10 秒,ArgoCD 會自動執行 Sync:

```bash
kubectl get deployment app-a -n dev

# 輸出:
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# app-a   1/1     1            1           5m
```

**✅ 自動還原成 replicas: 1**

**實際案例:**

某天凌晨,值班工程師因為線上流量突增,手動執行了:

```bash
kubectl scale deployment my-app --replicas=10
```

隔天早上,團隊發現集群 CPU 使用率異常高,檢查後發現有 10 個 Pod 在跑。但沒有人記得是誰改的,也不知道為什麼。

**如果有 Self-Heal:**

- 工程師手動調整後,ArgoCD 立刻還原 (或是記錄 OutOfSync 事件)
- 團隊能從 ArgoCD UI 看到「誰在什麼時候嘗試手動變更」
- 如果真的需要擴容,應該**修改 Git 中的 YAML**,而不是手動操作

**Self-Heal 的哲學:Git 是唯一的真相來源,任何繞過 Git 的變更都是「異常」。**

### 5. Sync Options 進階配置

#### `PruneLast=true`:最後才刪除

預設情況下,ArgoCD 會先刪除資源,再創建新資源。但這可能導致短暫的服務中斷。

**範例:**

你修改了 Service 的 Selector,ArgoCD 會:

1. 刪除舊的 Service
2. 創建新的 Service

**問題:** 在步驟 1 和 2 之間,流量無法進入。

**解決方案:**

```yaml
syncOptions:
- PruneLast=true  # 先創建新資源,最後才刪除舊資源
```

#### `ApplyOutOfSyncOnly=true`:只同步有變更的資源

預設情況下,ArgoCD 會對所有資源執行 `kubectl apply`。啟用此選項後,只會同步有差異的資源。

**效能提升:**

- 50 個資源的 Application,如果只改了 1 個 Deployment
- 沒啟用:執行 50 次 `kubectl apply`
- 啟用後:只執行 1 次 `kubectl apply`

#### `RespectIgnoreDifferences=true`:忽略特定欄位差異

某些欄位 (如 `status`) 會被 Kubernetes 自動修改,導致 ArgoCD 誤判為 OutOfSync。

```yaml
ignoreDifferences:
- group: apps
  kind: Deployment
  jsonPointers:
  - /spec/replicas  # 忽略 replicas 差異 (如果你用 HPA)
```

---

## 遇到的挑戰與對策

### 挑戰一:Self-Heal 太「激進」?

有同事反應:「我想臨時改個環境變數測試,但 ArgoCD 一直改回去!」

**對策:**

針對不同環境,採用不同策略:

- **Dev 環境:** 關閉 Self-Heal,讓開發者自由實驗
- **Staging/Production:** 啟用 Self-Heal,確保一致性

```bash
# Dev 環境不啟用 Self-Heal
argocd app create app-a-dev \
  --sync-policy automated \
  --auto-prune
  # ← 注意:沒有 --self-heal
```

### 挑戰二:如何避免誤刪所有資源?

某次測試時,我們不小心推送了一個空的 `kustomization.yaml`,結果 ArgoCD 以為「所有資源都該被刪除」,觸發了 Prune。

**對策:**

啟用 `allowEmpty: false`:

```yaml
syncPolicy:
  automated:
    allowEmpty: false  # 如果 Git 沒有任何資源,拒絕同步
```

ArgoCD 會報錯:

```
❌ Sync failed: source contains no resources
```

---

## 給團隊的觀念分享

### Sync Policy 就是「控制理論」的實踐

Self-Heal 的概念來自**控制理論 (Control Theory)**:

```
期望狀態 (Git) - 實際狀態 (K8s) = 誤差 (Drift)
ArgoCD 持續調整,讓誤差趨近於零
```

這跟**溫控器**的原理一樣:

- 你設定室溫 25°C (期望狀態)
- 溫控器檢測到 22°C (實際狀態)
- 自動開啟暖氣,直到達到 25°C (Self-Heal)

**GitOps 就是把這套理論應用在基礎設施管理上。**

### 不要害怕自動化

剛開始使用 Automated Sync 時,團隊會有點害怕:「萬一自動部署錯了怎麼辦?」

但我們發現:

- **手動部署一樣會出錯** (而且更難追蹤)
- **自動化+監控+快速回滾** 比「小心翼翼的手動操作」更可靠

**關鍵是建立信心:透過 Dev/Staging 環境驗證,再推到 Production。**

---

## 下週預告

ArgoCD 的基本功能已經掌握,下週要開始結合 **Helm**。

主題:**Helm 與 ArgoCD 的完美結合**。

我們要探討:

- 如何用 Helm Chart 部署應用
- 如何管理不同環境的 `values.yaml`
- ArgoCD 如何處理 Helm Hooks

十月開始,會進入更進階的主題:App-of-Apps 模式、多環境管理策略等。

---

**作者的話:**  
這週最大的收穫是「看到 Self-Heal 自動還原被手動改掉的配置」。那種感覺就像看到系統擁有了「自我修復能力」。GitOps 不只是工具,更是一種哲學:讓系統自己維護自己,而不是依賴人工介入。

**Tags:** #ArgoCD #Sync-Policy #Prune #Self-Heal #Declarative #GitOps
