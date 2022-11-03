---
layout: post
title: "企業級安全：SSO 整合實務"
date: 2022-11-04 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, SSO, Dex, OAuth, RBAC, Security]
---

進入十一月,我們要開始處理企業級場景的核心需求:**權限管理**。

在前面的實作中,所有人都用 `admin` 帳號登入 ArgoCD,這在生產環境是絕對不可接受的。今天要整合 **SSO (Single Sign-On)**,讓團隊成員用 GitHub 帳號登入,並透過 RBAC 控制「誰能操作哪些 Application」。

這是 ArgoCD 從「玩具」變成「企業級工具」的關鍵一步。

---

## 本週目標

整合 GitHub OAuth,配置 RBAC,實現細粒度的權限管理。

---

## 技術實作重點

### 1. 為什麼需要 SSO?

**問題:**

- ❌ 共用 `admin` 帳號,無法追蹤「誰做了什麼」
- ❌ 離職員工的帳號無法及時移除
- ❌ 密碼管理混亂 (寫在文件裡,外洩風險高)

**SSO 的優勢:**

- ✅ 統一身份管理 (GitHub/GitLab/LDAP)
- ✅ 自動同步組織成員變更
- ✅ 審計日誌包含真實使用者名稱
- ✅ 支援 MFA (Multi-Factor Authentication)

### 2. ArgoCD 的 SSO 架構

ArgoCD 使用 **Dex** 作為 SSO Connector:

```
User → GitHub OAuth → Dex → ArgoCD → Kubernetes RBAC
```

**Dex 的角色:**

- 對接外部身份提供者 (GitHub, Google, LDAP 等)
- 統一轉換成 OIDC Token
- 傳遞給 ArgoCD 做權限判斷

### 3. 在 GitHub 建立 OAuth App

**Step 1: 進入 GitHub Settings**

```
https://github.com/settings/developers
→ OAuth Apps → New OAuth App
```

**Step 2: 填寫資訊**

- **Application name:** ArgoCD Production
- **Homepage URL:** `https://argocd.example.com`
- **Authorization callback URL:** `https://argocd.example.com/api/dex/callback`

**Step 3: 記錄 Credentials**

- **Client ID:** `abc123def456`
- **Client Secret:** `xyz789uvw012` (點擊 Generate 產生)

### 4. 配置 ArgoCD 的 Dex

修改 ArgoCD ConfigMap:

```bash
kubectl edit configmap argocd-cm -n argocd
```

**新增 Dex 配置:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Dex 配置
  dex.config: |
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: abc123def456
        clientSecret: $dex.github.clientSecret  # 引用 Secret
        orgs:
        - name: my-org  # 限制只有這個組織的成員能登入
          teams:
          - platform-team  # 可選:只允許特定 Team
```

**儲存 Client Secret 到 Secret:**

```bash
kubectl patch secret argocd-secret -n argocd \
  --patch='{"stringData": {"dex.github.clientSecret": "xyz789uvw012"}}'
```

### 5. 配置 RBAC

ArgoCD 的 RBAC 分兩層:

1. **Application-Level RBAC:** 哪些使用者能操作哪些 Application
2. **Cluster-Level RBAC:** 哪些使用者能管理 ArgoCD 本身

**編輯 RBAC ConfigMap:**

```bash
kubectl edit configmap argocd-rbac-cm -n argocd
```

**範例配置:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  # 預設策略:拒絕所有
  policy.default: role:readonly
  
  # 策略定義
  policy.csv: |
    # ──────────────────────────────────────
    # 角色:Admin (完全控制)
    # ──────────────────────────────────────
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow
    p, role:admin, repositories, *, *, allow
    p, role:admin, projects, *, *, allow
    
    # ──────────────────────────────────────
    # 角色:Developer (只能操作 Dev 環境)
    # ──────────────────────────────────────
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, *-dev/*, allow
    p, role:developer, applications, override, *-dev/*, allow
    p, role:developer, applications, delete, *-dev/*, deny
    
    # ──────────────────────────────────────
    # 角色:Operator (可以操作 Staging/Prod,但不能刪除)
    # ──────────────────────────────────────
    p, role:operator, applications, get, */*, allow
    p, role:operator, applications, sync, *-staging/*, allow
    p, role:operator, applications, sync, *-prod/*, allow
    p, role:operator, applications, delete, */*, deny
    
    # ──────────────────────────────────────
    # 角色:ReadOnly (只能查看)
    # ──────────────────────────────────────
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, repositories, get, *, allow
    
    # ──────────────────────────────────────
    # 綁定:GitHub 組織成員 → 角色
    # ──────────────────────────────────────
    # Tech Lead 擁有 Admin 權限
    g, my-org:platform-team, role:admin
    
    # 開發者只能操作 Dev 環境
    g, my-org:dev-team, role:developer
    
    # Ops 團隊可以操作 Staging 與 Production
    g, my-org:ops-team, role:operator
    
    # 其他成員只能查看
    g, my-org:*, role:readonly
  
  # 允許匿名查看 (可選)
  policy.default: role:readonly
```

**RBAC 語法說明:**

```
p, <subject>, <resource>, <action>, <object>, <effect>

p: Policy (策略)
g: Group (群組綁定)

<resource> 可以是:
  - applications
  - clusters
  - repositories
  - projects

<action> 可以是:
  - get (查看)
  - create (建立)
  - update (更新)
  - delete (刪除)
  - sync (同步)
  - override (覆蓋參數)

<object> 格式:
  - */\* (所有)
  - *-dev/* (所有 dev 環境的 Application)
  - default/app-a (特定 Project 的特定 Application)
```

### 6. 重啟 ArgoCD 套用配置

```bash
kubectl rollout restart deployment argocd-server -n argocd
kubectl rollout restart deployment argocd-dex-server -n argocd
```

### 7. 測試 SSO 登入

**Step 1: 訪問 ArgoCD UI**

```
https://argocd.example.com
```

**Step 2: 點擊「LOG IN VIA GITHUB」**

**Step 3: 授權 GitHub OAuth**

**Step 4: 驗證權限**

```bash
# 用 Developer 帳號登入後,嘗試 Sync Production Application
argocd app sync app-a-prod

# 預期輸出:
# FATA[0001] rpc error: code = PermissionDenied 
# desc = permission denied: applications, sync, default/app-a-prod
```

✅ RBAC 生效!

---

## 遇到的挑戰與對策

### 挑戰一:GitHub Organization 權限設定錯誤

配置完後,發現所有人都無法登入,錯誤訊息:

```
User is not a member of allowed organizations
```

**根因:**

GitHub Org 的 **Third-party application access policy** 設定為「Restrict」。

**對策:**

```
GitHub Org Settings → Third-party access → OAuth App policies
→ 找到 ArgoCD → Grant access
```

### 挑戰二:RBAC 策略互相衝突

設定了多條規則後,發現某些使用者的權限不符合預期。

**根因:**

RBAC 策略的評估順序是「從上到下,先匹配先生效」。

**對策:**

- 把「deny」規則放在最前面
- 用 `argocd admin settings rbac can` 驗證權限

```bash
# 測試 developer 能否 Sync Dev Application
argocd admin settings rbac can developer sync applications '*-dev/*'
# 輸出: yes

# 測試 developer 能否 Sync Prod Application
argocd admin settings rbac can developer sync applications '*-prod/*'
# 輸出: no
```

### 挑戰三:CLI 如何使用 SSO?

**對策:**

```bash
# CLI 登入時,會自動開啟瀏覽器完成 OAuth 流程
argocd login argocd.example.com --sso

# 或使用 Access Token
argocd login argocd.example.com --sso --token-only
```

---

## 給團隊的觀念分享

### 最小權限原則 (Principle of Least Privilege)

RBAC 的設計哲學是:**預設拒絕,明確允許**。

**錯誤做法:**

```
# 所有人預設都是 Admin,需要限制的才設定
policy.default: role:admin
```

**正確做法:**

```
# 所有人預設只能查看,需要更高權限的才設定
policy.default: role:readonly
```

**這樣做的好處:**

- 即使配置錯誤,也不會「意外給太多權限」
- 新成員加入時,不會「預設就能刪除 Production」
- 符合資安稽核要求

### SSO 不只是「方便」,更是「安全」

很多團隊以為 SSO 只是「省得記密碼」,但其實:

**案例:**

某天,一名員工離職。如果用本地帳號,需要:

1. 改 ArgoCD 的密碼
2. 改 Kubernetes 的 Kubeconfig
3. 改 Jenkins 的帳號
4. 改 Grafana 的帳號
5. ...

**如果用 SSO:**

1. 在 GitHub Org 中移除該員工
2. 所有系統自動失效

**一個動作,全面生效。**

---

## 下週預告

SSO 與 RBAC 建立後,下週要面對另一個安全問題:**GitOps 下的 Secret 管理**。

主題內容:

- 為什麼不能把 Secret 直接放在 Git 中?
- Sealed Secrets 的原理與實作
- 如何整合 HashiCorp Vault 或 Azure Key Vault

Secret 管理是 GitOps 最大的挑戰之一,也是企業最關心的問題。

---

**作者的話:**  
這週整合 SSO 後,團隊的安全意識明顯提升。當開發者發現「我無法刪除 Production Application」時,反而覺得放心——這代表系統有足夠的保護機制。安全不是阻礙,而是信心的來源。

**Tags:** #ArgoCD #SSO #Dex #OAuth #RBAC #Security #Least-Privilege
