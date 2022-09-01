---
layout: post
title: "K8s 控制面搭建：ArgoCD 安裝實務"
date: 2022-09-02 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Kubernetes, Helm, Installation, CLI]
---

經過八月的理論建立與架構分析,九月開始進入實戰階段。今天要完成的任務:**在 Kubernetes 集群中安裝 ArgoCD,並完成基本配置**。

這不只是執行幾個指令而已,我們要理解每個步驟背後的原理:為什麼用 Helm 安裝?Namespace 如何隔離?Initial Password 的安全機制是什麼?只有理解這些,才能在遇到問題時不慌亂。

---

## 本週目標

成功安裝 ArgoCD 到 Kubernetes 集群,完成 CLI 登入,並驗證核心元件的運行狀態。

---

## 技術實作重點

### 1. 環境準備

**前置需求:**

- Kubernetes 集群 (v1.20+)
- kubectl 已配置並能連接集群
- Helm 3 (建議 3.8+)

**我們的測試環境:**

- AKS (Azure Kubernetes Service)
- 3 個 Worker Nodes (Standard_D4s_v3)
- Kubernetes v1.24

### 2. 為什麼用 Helm 安裝?

ArgoCD 官方提供三種安裝方式:

| 方式 | 優點 | 缺點 |
|------|------|------|
| kubectl apply -f install.yaml | 最簡單 | 難以客製化,升級麻煩 |
| Helm Chart | 易於配置與升級 | 需要理解 Helm |
| Operator | 聲明式管理 | 較複雜,適合大規模 |

**我們選擇 Helm,理由:**

1. **可配置性:** 透過 `values.yaml` 集中管理配置
2. **可升級性:** `helm upgrade` 即可無縫升級
3. **社群標準:** 企業環境常用,團隊已有 Helm 基礎

### 3. 安裝步驟

#### Step 1: 建立專用 Namespace

```bash
kubectl create namespace argocd
```

**為什麼要獨立 Namespace?**

- **隔離性:** ArgoCD 的元件與應用程式不混在一起
- **安全性:** 可以對 argocd namespace 設定嚴格的 RBAC
- **可維護性:** 未來要移除或升級,直接刪除 namespace 即可

#### Step 2: 新增 ArgoCD Helm Repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

#### Step 3: 準備 values.yaml

不要直接用預設配置,建立一個自定義的 `values.yaml`:

```yaml
# argocd-values.yaml

global:
  image:
    tag: v2.4.11  # 指定版本,避免意外升級

server:
  # 啟用 Ingress (透過 HTTPS 訪問 UI)
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - argocd.example.com
    tls:
      - secretName: argocd-tls
        hosts:
          - argocd.example.com
  
  # 設定資源限制
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi

controller:
  # Application Controller 的資源配置
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

repoServer:
  # Repo Server 的資源配置
  replicas: 2  # 提高可用性
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi

redis:
  # Redis 快取配置
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi

# 配置 Metrics (供 Prometheus 抓取)
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

**關鍵配置說明:**

1. **指定版本:** 避免自動升級導致不相容
2. **資源限制:** 防止 ArgoCD 吃掉過多集群資源
3. **Repo Server replicas: 2:** 提高處理 Git 請求的效能
4. **Ingress 配置:** 讓團隊能透過 HTTPS 訪問 UI

#### Step 4: 執行安裝

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --values argocd-values.yaml \
  --version 5.5.0  # 指定 Helm Chart 版本
```

**安裝過程會創建以下資源:**

```
NAME                                    READY   STATUS    RESTARTS   AGE
argocd-application-controller-0         1/1     Running   0          2m
argocd-dex-server-xxx                   1/1     Running   0          2m
argocd-redis-xxx                        1/1     Running   0          2m
argocd-repo-server-xxx                  1/1     Running   0          2m
argocd-repo-server-yyy                  1/1     Running   0          2m
argocd-server-xxx                       1/1     Running   0          2m
```

#### Step 5: 驗證安裝

```bash
# 檢查所有 Pod 是否 Running
kubectl get pods -n argocd

# 檢查 Service
kubectl get svc -n argocd

# 查看日誌 (確認沒有錯誤)
kubectl logs -n argocd deployment/argocd-server
```

### 4. 取得 Initial Admin Password

ArgoCD 預設會產生一個隨機的 admin 密碼,存放在 Secret 中:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

**輸出範例:**
```
kR3zP9vT2xQ7mN5w
```

**安全建議:**

1. **立刻修改密碼:** 第一次登入後,執行 `argocd account update-password`
2. **刪除 Secret:** 改完密碼後,刪除 `argocd-initial-admin-secret`
3. **啟用 SSO:** 生產環境應整合 GitHub/GitLab OAuth

### 5. CLI 安裝與登入

#### 安裝 ArgoCD CLI (macOS)

```bash
brew install argocd
```

或手動下載:

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.11/argocd-darwin-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

#### CLI 登入

**方式一:透過 Port-forward (測試環境)**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 另一個 Terminal
argocd login localhost:8080 \
  --username admin \
  --password <initial-password> \
  --insecure  # 因為是 self-signed cert
```

**方式二:透過 Ingress (生產環境)**

```bash
argocd login argocd.example.com \
  --username admin \
  --password <initial-password>
```

#### 驗證登入

```bash
argocd version

# 輸出:
# argocd: v2.4.11+sha
# argocd-server: v2.4.11+sha
```

---

## 遇到的挑戰與對策

### 挑戰一:Ingress 配置失敗

安裝後發現 Ingress 無法訪問,錯誤:`503 Service Temporarily Unavailable`。

**根因分析:**

檢查 Ingress 日誌:
```bash
kubectl logs -n ingress-nginx <nginx-controller-pod>
```

發現錯誤:`TLS Secret "argocd-tls" not found`。

**對策:**

我們使用 cert-manager 自動產生 TLS 憑證,但需要等待憑證產生:

```bash
kubectl get certificate -n argocd

NAME          READY   SECRET        AGE
argocd-tls    True    argocd-tls    5m
```

等待 `READY` 變成 `True` 後,Ingress 就正常了。

**教訓:** 分層驗證——先確認 Service 正常,再檢查 Ingress,最後才看 TLS。

### 挑戰二:Application Controller 持續 CrashLoopBackOff

**錯誤日誌:**
```
panic: unable to connect to redis: dial tcp: lookup argocd-redis: no such host
```

**根因:**

Helm Chart 的 Redis Service Name 與我們的配置不一致。

**對策:**

檢查 Service:
```bash
kubectl get svc -n argocd | grep redis
```

發現 Service 名稱是 `argocd-redis-ha-haproxy`(因為我們啟用了 Redis HA),但 Controller 預期的是 `argocd-redis`。

**修正 values.yaml:**

```yaml
redis-ha:
  enabled: false  # 測試環境先關閉 HA

redis:
  enabled: true
```

重新安裝:
```bash
helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  --values argocd-values.yaml
```

**教訓:** 不要盲目啟用所有「高可用」功能,先確保基本功能正常。

---

## 給團隊的觀念分享

### 安裝不是終點,而是起點

很多人以為「安裝成功」就代表「可以用了」,但其實:

- 安裝只是讓系統跑起來
- 配置才是決定系統行為的關鍵
- 監控與日誌才能確保系統穩定

**接下來我們要做的:**

1. 配置 RBAC (誰能操作哪些 Application)
2. 整合 Prometheus 監控
3. 設定告警 (當 Sync 失敗時通知團隊)

### 基礎設施即代碼的實踐

注意到我們把 `argocd-values.yaml` 納入 Git 版控了嗎?這就是 **Infrastructure as Code** 的實踐:

- 所有配置都在 Git 中
- 任何人都能重現這個安裝過程
- 未來要升級或遷移,直接用這個 YAML 即可

**這也是 GitOps 的一部分——連 ArgoCD 本身也用 GitOps 管理。**

---

## 下週預告

ArgoCD 安裝完成後,下週要設計 **Manifest Repository 的結構**。

這是一個關鍵決策:

- Monorepo 還是 Multi-repo?
- 如何組織不同環境 (Dev/Staging/Prod)?
- 程式碼與配置如何分離?

好的 Repository 結構,會讓後續的管理事半功倍。

---

**作者的話:**  
這週的安裝過程比預期順利,但也踩了幾個小坑。最重要的收穫是:理解每個配置的意義,而不是照抄官方文件。下週開始,我們要進入 GitOps 的核心——Manifest 管理。

**Tags:** #ArgoCD #Installation #Helm #Kubernetes #CLI #基礎設施即代碼
