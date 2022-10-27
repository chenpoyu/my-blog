---
layout: post
title: "日誌監控：從 ArgoCD UI 排除故障"
date: 2022-10-28 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Troubleshooting, Resource Tree, Sync Fail, Debugging]
---

前面幾週我們建立了完整的 GitOps 流程,但真正的考驗是:**當部署失敗時,如何快速定位問題?**

今天要深入 ArgoCD 的 UI 功能,學習如何利用 Resource Tree、Diff 檢視、Event Log 來進行故障排除。這些技能將決定你的 MTTR (Mean Time To Recovery) 是 30 分鐘還是 5 分鐘。

---

## 本週目標

掌握 ArgoCD UI 的排錯工具,建立系統化的故障排除流程。

---

## 技術實作重點

### 1. Resource Tree:視覺化的資源關係圖

**功能:**

ArgoCD UI 最強大的功能就是 **Resource Tree**,它以樹狀圖顯示應用的所有資源及其健康狀態。

**範例:**

```
Application: app-a-prod
├── Deployment (app-a)                    ✅ Healthy
│   └── ReplicaSet (app-a-7d8f9c)        ✅ Healthy
│       ├── Pod (app-a-7d8f9c-a1b2c)     ✅ Running
│       ├── Pod (app-a-7d8f9c-d3e4f)     ✅ Running
│       └── Pod (app-a-7d8f9c-g5h6i)     ❌ CrashLoopBackOff
├── Service (app-a-svc)                   ✅ Healthy
├── Ingress (app-a-ingress)               ⚠️ Progressing
└── ConfigMap (app-a-config)              ✅ Healthy
```

**健康狀態圖例:**

- ✅ **Healthy:** 資源運行正常
- ⚠️ **Progressing:** 資源正在部署中
- ❌ **Degraded:** 資源有問題
- ⏸️ **Suspended:** 資源被暫停
- ❓ **Unknown:** 無法判斷狀態

### 2. 常見故障場景與排除流程

#### 場景一:Pod CrashLoopBackOff

**症狀:**

```
Pod (app-a-xxx) ❌ CrashLoopBackOff
```

**排查步驟:**

**Step 1: 點擊 Pod → 查看 Events**

```
Events:
  Warning  BackOff  5m  kubelet  Back-off restarting failed container
  Warning  Failed   5m  kubelet  Error: ErrImagePull
```

**Step 2: 查看 Pod Logs**

在 ArgoCD UI 中點擊 「LOGS」按鈕:

```
Error: connect ECONNREFUSED 10.0.0.5:5432
    at TCPConnectWrap.afterConnect [as oncomplete]
```

**根因:** 無法連接到資料庫。

**Step 3: 檢查 ConfigMap**

```yaml
# ConfigMap: app-a-config
DATABASE_URL: postgres://10.0.0.5:5432/myapp  # ← 錯誤的 IP
```

**解決方案:**

```bash
cd k8s-manifests/apps/app-a/overlays/production
# 修正 ConfigMap
git commit -m "fix: correct database URL"
git push
```

**驗證:** 等待 ArgoCD 自動同步,Pod 變成 Running。

---

#### 場景二:Sync Failed

**症狀:**

```
Application: app-a-prod
Status: OutOfSync
Sync: Failed
Message: The Kubernetes API could not find ...
```

**排查步驟:**

**Step 1: 查看 Sync Status 詳情**

```
Failed to sync:
  Resource: ConfigMap/app-a-config
  Error: namespaces "production" not found
```

**根因:** Namespace 不存在。

**解決方案:**

在 Application 中啟用 `CreateNamespace`:

```yaml
syncPolicy:
  syncOptions:
  - CreateNamespace=true
```

或手動建立:

```bash
kubectl create namespace production
```

---

#### 場景三:Image Pull Error

**症狀:**

```
Pod (app-a-xxx) ❌ ErrImagePull
Events:
  Warning  Failed  1m  kubelet  Failed to pull image "myregistry.azurecr.io/app-a:v1.2.3": rpc error: code = Unknown desc = Error response from daemon: Get https://myregistry.azurecr.io/v2/: unauthorized
```

**根因:** 沒有配置 Image Pull Secret。

**解決方案:**

**Step 1: 建立 Secret**

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=myregistry.azurecr.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --namespace=production
```

**Step 2: 在 Deployment 中引用**

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
      - name: acr-secret
      containers:
      - name: app
        image: myregistry.azurecr.io/app-a:v1.2.3
```

---

#### 場景四:Ingress 無法訪問

**症狀:**

```
Ingress (app-a-ingress) ⚠️ Progressing
```

長時間卡在 Progressing 狀態。

**排查步驟:**

**Step 1: 檢查 Ingress Events**

```
Warning  Sync  1m  ingress-nginx-controller  
Error: service "app-a-svc" not found
```

**根因:** Service 名稱不匹配。

**Step 2: 檢查 Service**

```bash
kubectl get svc -n production

# 輸出:
# NAME           TYPE        CLUSTER-IP     ...
# app-a-service  ClusterIP   10.0.123.45    ...
```

發現 Service 名稱是 `app-a-service`,但 Ingress 引用的是 `app-a-svc`。

**解決方案:**

修正 Ingress:

```yaml
backend:
  service:
    name: app-a-service  # 正確的名稱
    port:
      number: 80
```

---

### 3. Diff 檢視:比對 Git vs. K8s

**功能:**

當 Application 顯示 `OutOfSync` 時,可以用 Diff 檢視看到具體差異。

**範例:**

```diff
# Git (Desired State)
spec:
  replicas: 3

# Kubernetes (Live State)
spec:
  replicas: 5  # 被手動改了
```

**使用方式:**

1. 在 ArgoCD UI 中點擊 「APP DIFF」
2. 選擇 「Compact Diff」或「Full Diff」
3. 紅色代表 Git 中的版本,綠色代表 K8s 中的版本

**實戰案例:**

發現 Production 的 CPU Request 被手動改過:

```diff
- requests:
-   cpu: 200m
+ requests:
+   cpu: 1000m  # 有人手動調整了
```

**行動:**

如果這是臨時調整,應該更新 Git:

```yaml
# overlays/production/resource-patch.yaml
resources:
  requests:
    cpu: 1000m
```

然後讓 Self-Heal 還原。

---

### 4. Event Log:追蹤變更歷史

**功能:**

ArgoCD 記錄了所有 Sync 操作的歷史,包括:

- 誰觸發的 (User or Automated)
- 什麼時候觸發的
- 哪些資源被修改了
- Sync 是否成功

**查看方式:**

CLI:

```bash
argocd app history app-a-prod

# 輸出:
# ID  DATE                           REVISION
# 10  2022-10-28 10:23:45 +0800      a1b2c3d (Update image to v1.2.4)
# 9   2022-10-28 09:15:32 +0800      d4e5f6g (Fix configmap)
# 8   2022-10-27 18:42:11 +0800      g7h8i9j (Initial deployment)
```

**回滾到特定版本:**

```bash
argocd app rollback app-a-prod 9
```

---

### 5. 系統化的排錯流程

```
1. 檢查 Application Status
   ├─ Synced? → 正常
   ├─ OutOfSync? → 查看 Diff
   └─ Sync Failed? → 查看錯誤訊息

2. 檢查 Resource Tree
   ├─ 哪個資源 Degraded?
   └─ 點擊該資源查看 Events

3. 查看 Pod Logs (如果是 Pod 問題)
   ├─ 應用程式日誌
   └─ Init Container 日誌

4. 檢查相關資源
   ├─ ConfigMap / Secret 是否正確?
   ├─ Service 是否存在?
   └─ Ingress 是否配置正確?

5. 查看 Git History
   ├─ 最近有哪些變更?
   └─ 誰改的?為什麼改?

6. 決策
   ├─ 是配置問題? → 修正 Git 並重新 Sync
   ├─ 是程式問題? → 回滾到上一個版本
   └─ 是環境問題? → 檢查 K8s 集群狀態
```

---

## 遇到的挑戰與對策

### 挑戰一:Resource Tree 太複雜

當應用有幾十個資源時,Tree 會很亂。

**對策:**

使用 **Filter** 功能:

- 只顯示 Degraded 的資源
- 只顯示特定類型 (如 Pod)
- 搜尋特定資源名稱

### 挑戰二:無法判斷是「代碼問題」還是「配置問題」

**對策:**

建立 **Checklist**:

```
✅ Image Tag 是否正確?
✅ ConfigMap / Secret 是否更新?
✅ 資源限制是否足夠?
✅ Dependency (資料庫/Redis) 是否可用?
✅ 最近有沒有改過 Base 配置?
```

---

## 給團隊的觀念分享

### 可觀測性 > 除錯技巧

很多人以為「會除錯」就是「技術好」,但其實:**可觀測性高的系統,根本不需要複雜的除錯技巧**。

ArgoCD 的 Resource Tree、Diff 檢視、Event Log,都是「可觀測性」的體現:

- 不需要 `kubectl describe` 十幾次
- 不需要翻遍 Git History
- 不需要問「誰改了配置」

**所有資訊都在 UI 中,一目了然。**

### 故障是學習的機會

這週我們刻意製造了幾個故障場景,讓團隊練習排錯。結果發現:

- 大部分問題都是「配置錯誤」,而非「程式 Bug」
- 有了 Resource Tree 後,定位問題的速度快了 10 倍
- Self-Heal 減少了 80% 的手動介入

**故障不可怕,可怕的是無法快速恢復。**

---

## 下週預告

十月的最後一週,我們要進入 **企業級功能:SSO 整合實務**。

主題內容:

- 如何整合 GitHub OAuth (透過 Dex)
- 如何配置 RBAC (角色權限管理)
- 如何實現「開發者只能操作 Dev 環境」

這是 ArgoCD 在企業落地的關鍵一環——沒有權限管理,GitOps 只是玩具。

---

**作者的話:**  
這週透過模擬故障場景,讓團隊建立了「系統化排錯」的習慣。最大的收穫是:好的工具能讓排錯變得簡單,而不是依賴「經驗豐富的資深工程師」。這才是可持續的團隊成長。

**Tags:** #ArgoCD #Troubleshooting #Resource-Tree #Debugging #MTTR
