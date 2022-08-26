---
layout: post
title: "初探 ArgoCD：架構與元件分析"
date: 2022-08-26 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Architecture, Application Controller, Kubernetes]
---

前三週我們完成了理論建立與 Jenkins 重構,這週要開始深入 ArgoCD 本身。在安裝與配置之前,先理解它的內部架構是非常重要的——這決定了你能否在遇到問題時快速定位根因。

今天我們要拆解 ArgoCD 的核心元件:**Application Controller、API Server、Repo Server**,以及它們如何協同工作來實現 GitOps 的自動化部署。

> **工程哲學:** 不理解原理就使用工具,遲早會在生產環境踩坑。

---

## 本週目標

建立對 ArgoCD 內部架構的完整認知,為下週的實際安裝與配置奠定基礎。

---

## 技術實作重點

### 1. ArgoCD 的整體架構

ArgoCD 是一個運行在 Kubernetes 集群內的控制器,它的核心架構如下:

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │              ArgoCD Namespace                    │   │
│  │                                                   │   │
│  │  ┌──────────────┐      ┌──────────────┐        │   │
│  │  │   API Server │◄────►│ Redis Cache  │        │   │
│  │  └──────┬───────┘      └──────────────┘        │   │
│  │         │                                        │   │
│  │         ▼                                        │   │
│  │  ┌──────────────┐      ┌──────────────┐        │   │
│  │  │ Application  │◄────►│ Repo Server  │        │   │
│  │  │  Controller  │      └──────┬───────┘        │   │
│  │  └──────┬───────┘             │                 │   │
│  │         │                     │                 │   │
│  └─────────┼─────────────────────┼─────────────────┘   │
│            │                     │                     │
│            ▼                     ▼                     │
│      ┌──────────────┐      ┌──────────────┐          │
│      │  Kubernetes  │      │ Git Repository│          │
│      │   Resources  │      │  (External)   │          │
│      └──────────────┘      └──────────────┘          │
└─────────────────────────────────────────────────────────┘
```

### 2. 核心元件深度解析

#### (1) API Server:對外的門面

**職責:**
- 提供 gRPC/REST API,讓 CLI、UI、Webhook 與 ArgoCD 互動
- 管理 Application 的 CRUD 操作
- 處理使用者認證與授權 (RBAC)

**實作細節:**
- 使用 Kubernetes Custom Resource Definition (CRD) 儲存 Application 配置
- 支援 SSO 整合 (透過 Dex)
- 暴露 Prometheus Metrics 供監控

**範例:Application CRD**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: staging/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**當你執行 `argocd app create` 時,實際上是在 Kubernetes 中創建了這個 CRD 物件。**

#### (2) Application Controller:大腦

**職責:**
- 監控 Git Repository 的變更
- 比對 Git 中的期望狀態 (Desired State) 與 Kubernetes 中的實際狀態 (Live State)
- 當檢測到差異時,執行同步 (Sync) 操作
- 實現 Self-Healing:當有人手動改了 K8s 資源,自動還原

**工作流程:**

```
1. 輪詢 Git Repository (預設每 3 分鐘)
2. 如果檢測到 commit 變更:
   a. 呼叫 Repo Server 取得 Manifest
   b. 與 Kubernetes API 取得實際狀態
   c. 計算 Diff
   d. 如果有差異且啟用 Auto-Sync:執行 kubectl apply
3. 如果啟用 Self-Heal:
   a. 持續監控 K8s 資源
   b. 如果有人手動修改,立刻還原成 Git 中的版本
```

**關鍵洞察:**  
Application Controller 實現了 Kubernetes 的 **Reconciliation Loop (調和迴圈)** 模式,這也是為什麼 GitOps 能保證「狀態一致性」。

#### (3) Repo Server:Git 與 Manifest 的橋樑

**職責:**
- Clone Git Repository
- 處理 Helm、Kustomize、Jsonnet 等工具
- 產生最終的 Kubernetes Manifest (Plain YAML)
- 快取 Git 內容,減少重複 Clone

**支援的 Manifest 格式:**

| 格式 | 說明 | 使用時機 |
|------|------|----------|
| Plain YAML | 最直接的 K8s 配置 | 簡單應用 |
| Helm Chart | 模板化配置 | 需要跨環境重用 |
| Kustomize | 基於覆蓋 (Overlay) 的配置 | 多環境差異小 |
| Jsonnet | 程式化生成配置 | 複雜的動態配置 |

**實作細節:**

當 Application Controller 需要 Manifest 時:

```
Controller: "給我 https://github.com/my-org/k8s-manifests.git 的 staging/my-app"
Repo Server: "我先檢查快取... 沒有,開始 Clone"
Repo Server: "偵測到是 Helm Chart,執行 helm template..."
Repo Server: "產生的 Manifest 如下:..."
Controller: "收到,開始比對與同步"
```

#### (4) Redis:快取與狀態暫存

**職責:**
- 快取 Git Repository 的 commit hash
- 暫存 Application 的狀態
- 減少對 Kubernetes API 的查詢壓力

**為什麼需要 Redis?**

假設你有 100 個 Application,每個都要每 3 分鐘去 Git 檢查更新:

- 沒有快取:每次都要 Clone Git → 100 * 20 = 300 次 Git 操作/分鐘
- 有快取:只需要檢查 commit hash → 100 次輕量級查詢/分鐘

**效能提升數十倍。**

---

### 3. 從架構理解 GitOps 的核心優勢

#### 優勢一:Pull-based 的安全性

```
[外部] → [API Server] → [Application Controller] → [K8s API]
```

注意:**外部系統 (如 Jenkins) 只能透過 API Server 更新 Application CRD,但不能直接操作 K8s 資源。** 所有的 `kubectl apply` 都是由 Application Controller 在集群內部執行。

#### 優勢二:可觀測性

因為 Application Controller 持續監控狀態,所以 ArgoCD 能提供:

- **實時視覺化:** Resource Tree 顯示每個資源的健康狀態
- **Diff 檢視:** 清楚看到 Git 與 K8s 的差異
- **事件日誌:** 記錄每次同步的歷史

這些都是傳統 CI/CD 工具做不到的。

#### 優勢三:聲明式管理

所有配置都儲存在 Application CRD 中,意味著:

- 你可以用 `kubectl` 查看 Application: `kubectl get applications -n argocd`
- 你可以用 GitOps 管理 ArgoCD 本身 (App-of-Apps 模式)
- 你可以備份與還原 Application 配置

---

## 遇到的挑戰與對策

### 挑戰一:如何理解 Application Controller 的 Reconciliation?

有同事問:「Controller 怎麼知道 K8s 資源被手動改了?」

**對策:**  
我畫了一個時序圖說明:

```
Time 0: Git (replicas: 3) = K8s (replicas: 3) ✅ Synced
Time 1: 工程師手動執行 kubectl scale --replicas=5
Time 2: Git (replicas: 3) ≠ K8s (replicas: 5) ⚠️ OutOfSync
Time 3: Controller 偵測到差異,執行 Sync
Time 4: K8s (replicas: 3) ✅ Synced (Self-Heal)
```

**關鍵:** Controller 透過 Kubernetes Watch API 持續監聽資源變更,不是靠輪詢。

### 挑戰二:Repo Server 會不會成為瓶頸?

當有大量 Application 時,Repo Server 需要處理很多 Git Clone 與 Helm Template 操作。

**對策:**  
ArgoCD 支援 Horizontal Scaling:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  replicas: 3  # 增加副本數
```

而且 Redis 快取機制能大幅降低重複運算。

---

## 給團隊的觀念分享

### 架構決定了系統的行為邊界

理解 ArgoCD 的架構後,我們就能回答很多問題:

- **「ArgoCD 能不能部署到多個集群?」**  
  → 能,因為 Application 的 `destination` 可以指向不同的 K8s API Server。

- **「如果 Git Repository 很大,會不會很慢?」**  
  → 會,但可以用 Shallow Clone 或調整快取策略。

- **「能不能只同步某個 Namespace?」**  
  → 能,在 Application CRD 的 `destination.namespace` 指定即可。

**架構不是抽象概念,而是決定了你能做什麼、不能做什麼。**

### 閱讀原始碼是最好的文件

這週我花了不少時間讀 ArgoCD 的原始碼 (Go 語言),特別是 Application Controller 的實作。雖然剛開始有點吃力,但收穫很大:

- 官方文件可能過時或不完整,但程式碼不會說謊
- 理解實作細節後,遇到問題時能更快定位
- 也能更有信心地向團隊解釋「為什麼這樣設計」

**給年輕工程師的建議:** 不要只停留在「會用」的層面,嘗試去理解「為什麼這樣設計」。

---

## 下週預告

架構分析完成後,下週要正式動手了:**K8s 控制面搭建:ArgoCD 安裝實務**。

我們會:

1. 用 Helm Chart 安裝 ArgoCD
2. 配置 Initial Admin Password
3. 透過 CLI 登入與基本操作
4. 理解 Namespace 隔離與 RBAC 配置

準備好你的 Kubernetes 集群,下週開始實戰!

---

**作者的話:**  
這週沒有寫代碼,但花了大量時間在理解架構。這是值得的——理解了原理,後續的安裝、配置、排錯都會變得順暢。技術的深度,往往比廣度更重要。

**Tags:** #ArgoCD #Architecture #Application-Controller #Repo-Server #Kubernetes #原理分析
