---
layout: post
title: "多集群 (Multi-cluster) 管理規劃"
date: 2022-11-18 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Multi-cluster, Cluster Management, GitOps]
---

這週要探討企業級場景的重要能力:**用一個 ArgoCD 管理多個 Kubernetes 集群**。

隨著業務成長,我們可能會有:開發集群、測試集群、生產集群,甚至多個地理區域的集群。如果每個集群都裝一個 ArgoCD,管理成本會非常高。今天要實現「中央控制,分散執行」的架構。

---

## 本週目標

將遠端 Kubernetes 集群註冊到 ArgoCD,實現跨集群部署。

---

## 技術實作重點

### 1. 多集群架構設計

**模式一:每個集群一個 ArgoCD (不推薦)**

```
Dev Cluster → ArgoCD (Dev)
Staging Cluster → ArgoCD (Staging)  
Production Cluster → ArgoCD (Production)
```

**缺點:**

- 需要維護多個 ArgoCD
- 權限管理複雜
- 無法統一查看所有環境

**模式二:中央 ArgoCD + 多個目標集群 (推薦)**

```
┌─────────────────────────┐
│  Control Plane Cluster  │
│  (ArgoCD installed)     │
└───────────┬─────────────┘
            │
     ┌──────┼──────┐
     │      │      │
     ▼      ▼      ▼
┌─────┐ ┌─────┐ ┌─────┐
│ Dev │ │Stage│ │Prod │
└─────┘ └─────┘ └─────┘
```

**優點:**

- 單一控制點
- 統一權限管理
- 跨集群視圖

### 2. 註冊遠端集群

#### Step 1: 準備遠端集群的 Kubeconfig

假設我們有三個集群:

```bash
# 列出所有 Context
kubectl config get-contexts

# 輸出:
# CURRENT   NAME                 CLUSTER              AUTHINFO
# *         control-plane-ctx    control-plane-cluster ...
#           dev-ctx              dev-cluster          ...
#           staging-ctx          staging-cluster      ...
#           prod-ctx             prod-cluster         ...
```

#### Step 2: 用 ArgoCD CLI 註冊集群

```bash
# 註冊 Dev 集群
argocd cluster add dev-ctx --name dev-cluster

# 註冊 Staging 集群
argocd cluster add staging-ctx --name staging-cluster

# 註冊 Production 集群
argocd cluster add prod-ctx --name production-cluster
```

**背後發生了什麼?**

1. ArgoCD 在遠端集群中創建 ServiceAccount `argocd-manager`
2. 賦予該 ServiceAccount Cluster-Admin 權限 (可以後續調整)
3. 將憑證存儲在 ArgoCD 的 Secret 中

#### Step 3: 驗證註冊

```bash
argocd cluster list

# 輸出:
# SERVER                          NAME                 VERSION  STATUS
# https://kubernetes.default.svc  in-cluster           1.24     Successful
# https://dev-cluster.example     dev-cluster          1.24     Successful
# https://staging.example         staging-cluster      1.24     Successful
# https://prod.example            production-cluster   1.24     Successful
```

### 3. 跨集群部署 Application

#### Application YAML (部署到 Dev 集群)

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
    path: apps/app-a/overlays/dev
  destination:
    # 指向遠端集群
    server: https://dev-cluster.example
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

#### 用 CLI 建立

```bash
argocd app create app-a-dev \
  --repo https://github.com/my-org/k8s-manifests.git \
  --path apps/app-a/overlays/dev \
  --dest-server https://dev-cluster.example \
  --dest-namespace default \
  --sync-policy automated
```

### 4. 集群分層管理策略

**實務上,我們會按「重要性」分層:**

| 集群 | ArgoCD 安裝位置 | 用途 | 自動同步 |
|------|----------------|------|---------|
| **Control Plane** | ✅ ArgoCD 主節點 | 管理其他集群 | Manual |
| **Dev** | - | 開發測試 | Automated |
| **Staging** | - | 預生產驗證 | Automated |
| **Production** | ✅ 獨立 ArgoCD (HA) | 生產環境 | Manual |

**為什麼 Production 要獨立?**

- **高可用性:** Production ArgoCD 也做 HA
- **安全隔離:** 即使 Control Plane 被入侵,Production 不受影響
- **審計要求:** 某些企業要求生產環境完全隔離

### 5. 用 App-of-Apps 管理多集群

#### Repository 結構

```
k8s-manifests/
├── bootstrap/
│   ├── dev-cluster/
│   │   ├── app-a.yaml
│   │   └── app-b.yaml
│   ├── staging-cluster/
│   │   ├── app-a.yaml
│   │   └── app-b.yaml
│   └── prod-cluster/
│       ├── app-a.yaml
│       └── app-b.yaml
└── apps/
    ├── app-a/
    └── app-b/
```

#### Bootstrap Application (Dev 集群)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: bootstrap/dev-cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: {}
```

**一個指令,部署整個 Dev 集群:**

```bash
kubectl apply -f bootstrap-dev.yaml
```

### 6. 集群權限管理

**問題:註冊集群時,預設給了 Cluster-Admin 權限,太大了。**

**對策:建立受限的 ServiceAccount**

#### Step 1: 在遠端集群建立 Role

```yaml
# dev-cluster-role.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-manager
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager-role
rules:
# 允許讀取所有資源
- apiGroups:
  - "*"
  resources:
  - "*"
  verbs:
  - get
  - list
  - watch
# 允許在特定 Namespace 寫入
- apiGroups:
  - "*"
  resources:
  - "*"
  verbs:
  - create
  - update
  - patch
  - delete
  namespaces:
  - default
  - dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-manager-role
subjects:
- kind: ServiceAccount
  name: argocd-manager
  namespace: kube-system
```

#### Step 2: 取得 ServiceAccount Token

```bash
kubectl -n kube-system get secret \
  $(kubectl -n kube-system get sa argocd-manager -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{.data.token}' | base64 -d
```

#### Step 3: 手動註冊集群 (而非用 CLI 自動註冊)

```bash
argocd cluster add dev-ctx \
  --name dev-cluster \
  --service-account argocd-manager \
  --namespace kube-system
```

---

## 遇到的挑戰與對策

### 挑戰一:遠端集群網路不通

註冊集群時,錯誤:

```
FATA[0005] failed to connect to cluster: dial tcp: i/o timeout
```

**根因:**

- 遠端集群的 API Server 沒有公開 IP
- 或是 Firewall 阻擋了

**對策:**

**方式一:使用 VPN 或 Private Link**

確保 Control Plane 集群能連到遠端集群的 API Server。

**方式二:使用 ArgoCD Agent (Pull-based)**

ArgoCD 2.5+ 支援 Agent 模式:

```
Remote Cluster (ArgoCD Agent) --pull--> Control Plane (ArgoCD)
```

這樣遠端集群主動連回 Control Plane,不需要開防火牆。

### 挑戰二:集群憑證過期

某天發現 ArgoCD 無法連到遠端集群:

```
Unable to connect to cluster: x509: certificate has expired
```

**對策:**

定期更新集群憑證:

```bash
# 重新註冊集群
argocd cluster add dev-ctx --name dev-cluster --upsert
```

或使用 **Service Account Token** 而非證書 (Token 可以設定不過期)。

---

## 給團隊的觀念分享

### 中央化 vs. 去中心化

**中央化 (Single ArgoCD):**

- ✅ 統一管理
- ✅ 統一權限
- ❌ 單點故障風險

**去中心化 (每集群一個 ArgoCD):**

- ✅ 故障隔離
- ❌ 管理複雜
- ❌ 權限分散

**我們的折衷:**

- Dev/Staging:中央管理 (追求效率)
- Production:獨立 ArgoCD (追求穩定)

**沒有完美的架構,只有適合的架構。**

### 多集群是「能力」,不是「必須」

不要為了「炫技」而導入多集群。問自己:

- 真的需要多個集群嗎? (成本、維護負擔)
- 能否用 Namespace 隔離就好?
- 團隊有能力維護多集群嗎?

**技術要服務業務,而非為了技術而技術。**

---

## 下週預告

多集群管理建立後,下週要探索 **Image 更新自動化**。

主題內容:

- ArgoCD Image Updater 實驗
- Image Tagging 策略 (latest vs. SHA vs. Semantic Version)
- 如何實現「CI 推送 Image,CD 自動更新」

這是 GitOps 完整閉環的最後一塊拼圖。

---

**作者的話:**  
這週配置多集群時,感受到「架構的力量」。當你有 3 個集群、50 個微服務時,單一 ArgoCD 的價值就顯現了——一個 UI 看到所有環境的狀態,這種掌控感無價。

**Tags:** #ArgoCD #Multi-cluster #Cluster-Management #Central-Control #GitOps
