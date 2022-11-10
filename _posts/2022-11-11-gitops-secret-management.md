---
layout: post
title: "GitOps 下的 Secret 管理"
date: 2022-11-11 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [Sealed Secrets, Secret Management, GitOps, Security, Vault]
---

這週要解決 GitOps 最大的挑戰:**如何安全地管理 Secret?**

GitOps 的核心是「所有配置都在 Git 中」,但 Secret (密碼、API Key、證書) 顯然不能明文提交。今天要評估三種主流方案:**Sealed Secrets、External Secrets Operator、HashiCorp Vault**,並選擇最適合我們團隊的方案。

---

## 本週目標

建立安全的 Secret 管理機制,讓敏感資訊也能納入 GitOps 流程。

---

## 技術實作重點

### 1. 為什麼 Secret 管理這麼困難?

**GitOps 的矛盾:**

- ✅ 配置要在 Git 中 (可追蹤、可審計)
- ❌ Secret 不能在 Git 中 (安全風險)

**錯誤做法範例:**

```yaml
# ❌ 千萬不要這樣做!
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
stringData:
  password: "MySecretPassword123"  # 明文密碼
```

即使刪除 Commit,Git History 中仍然保留,任何人都能看到。

### 2. 方案評估

| 方案 | 原理 | 優點 | 缺點 | 適用場景 |
|------|------|------|------|----------|
| **Sealed Secrets** | 用公鑰加密,只有集群能解密 | 簡單,無外部依賴 | 密鑰輪替困難 | 小型團隊,Secret 不常變更 |
| **External Secrets Operator** | 從外部系統 (Vault/AWS Secrets Manager) 拉取 | 集中管理,支援輪替 | 需要維護外部系統 | 企業級,已有 Vault/雲端 Secret 服務 |
| **SOPS** | 用 PGP/KMS 加密 YAML 檔案 | 靈活,支援多種加密後端 | 手動操作多 | 需要精細控制加密範圍 |

**我們的選擇:Sealed Secrets (先求簡單)**

理由:

- 團隊還沒有 Vault
- Secret 變更頻率低 (每季一次)
- 不需要額外維護外部系統

### 3. Sealed Secrets 原理

```
1. 管理員用公鑰加密 Secret
   plaintext → SealedSecret (可以放 Git)

2. 提交到 Git

3. ArgoCD 同步到 K8s

4. Sealed Secrets Controller 用私鑰解密
   SealedSecret → Secret (只存在 K8s 中)

5. Pod 使用解密後的 Secret
```

**關鍵:私鑰只存在 Kubernetes 集群中,永不離開。**

### 4. 安裝 Sealed Secrets

#### 在集群中安裝 Controller

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml
```

**驗證安裝:**

```bash
kubectl get pods -n kube-system | grep sealed-secrets

# 輸出:
# sealed-secrets-controller-xxx   1/1     Running   0   1m
```

#### 安裝 CLI 工具 (kubeseal)

```bash
# macOS
brew install kubeseal

# 或手動下載
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/kubeseal-0.18.0-darwin-amd64.tar.gz
tar -xvzf kubeseal-0.18.0-darwin-amd64.tar.gz
sudo mv kubeseal /usr/local/bin/
```

### 5. 建立 Sealed Secret

#### Step 1: 建立普通的 Secret YAML (不提交到 Git!)

```yaml
# database-secret.yaml (本地檔案,不加入 Git)
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
  namespace: production
stringData:
  DB_PASSWORD: "MySecretPassword123"
  DB_USERNAME: "admin"
```

#### Step 2: 用 kubeseal 加密

```bash
kubeseal \
  --format yaml \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  < database-secret.yaml \
  > database-sealed-secret.yaml
```

**產生的檔案:**

```yaml
# database-sealed-secret.yaml (可以提交到 Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: database-secret
  namespace: production
spec:
  encryptedData:
    DB_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...  # 加密後的內容
    DB_USERNAME: AgAR5JExOlW0xYz7M5J9xEhG8QwNPm6v...
  template:
    metadata:
      name: database-secret
      namespace: production
```

#### Step 3: 提交到 Git

```bash
git add database-sealed-secret.yaml
git commit -m "feat: add sealed database secret"
git push
```

#### Step 4: ArgoCD 自動同步

ArgoCD 部署 SealedSecret 到 K8s 後,Controller 會自動解密並創建 Secret:

```bash
kubectl get secret database-secret -n production

# 輸出:
# NAME               TYPE     DATA   AGE
# database-secret    Opaque   2      10s
```

**驗證內容:**

```bash
kubectl get secret database-secret -n production -o jsonpath="{.data.DB_PASSWORD}" | base64 -d

# 輸出:
# MySecretPassword123
```

### 6. 在 Deployment 中使用

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: app
        image: myregistry.azurecr.io/app-a:v1.2.3
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_PASSWORD
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_USERNAME
```

### 7. 更新 Secret 的流程

**問題:如何更新已經加密的 Secret?**

**流程:**

```bash
# 1. 修改原始 Secret (本地檔案)
vim database-secret.yaml

# 2. 重新加密
kubeseal < database-secret.yaml > database-sealed-secret.yaml

# 3. 提交更新
git add database-sealed-secret.yaml
git commit -m "chore: rotate database password"
git push

# 4. ArgoCD 自動同步,Controller 解密
```

**注意:需要重啟 Pod 才能讀取新的 Secret。**

### 8. Sealed Secrets 的進階配置

#### Scope 控制 (控制 Secret 可以在哪些 Namespace 使用)

```bash
# Strict (預設):只能在特定 Namespace 使用
kubeseal --scope strict < secret.yaml > sealed-secret.yaml

# Namespace-wide:可以在同 Namespace 的不同名稱使用
kubeseal --scope namespace-wide < secret.yaml > sealed-secret.yaml

# Cluster-wide:可以在任何 Namespace 使用 (不安全,不建議)
kubeseal --scope cluster-wide < secret.yaml > sealed-secret.yaml
```

#### 備份私鑰 (災難恢復用)

```bash
# 導出私鑰
kubectl get secret -n kube-system sealed-secrets-key \
  -o jsonpath="{.data['tls\.crt']}" | base64 -d > sealed-secrets.crt

kubectl get secret -n kube-system sealed-secrets-key \
  -o jsonpath="{.data['tls\.key']}" | base64 -d > sealed-secrets.key

# ⚠️ 妥善保管這兩個檔案!存在安全的地方 (如 Password Manager)
```

**災難恢復時:**

```bash
kubectl create secret tls sealed-secrets-key \
  --cert=sealed-secrets.crt \
  --key=sealed-secrets.key \
  --namespace=kube-system
```

---

## 遇到的挑戰與對策

### 挑戰一:如何避免原始 Secret 被提交到 Git?

**對策:使用 .gitignore**

```bash
# .gitignore
*-secret.yaml
!*-sealed-secret.yaml
```

### 挑戰二:團隊成員如何加密 Secret?

每個人都需要 `kubeseal`,但不需要私鑰 (因為只需要公鑰加密)。

**對策:分享公鑰**

```bash
# 導出公鑰
kubeseal --fetch-cert > sealed-secrets-pub.pem

# 團隊成員使用公鑰加密 (不需要連接集群)
kubeseal --cert sealed-secrets-pub.pem < secret.yaml > sealed-secret.yaml
```

### 挑戰三:Secret 輪替流程繁瑣

**對策:寫腳本自動化**

```bash
#!/bin/bash
# rotate-secret.sh

SECRET_NAME=$1
NAMESPACE=$2

echo "Rotating $SECRET_NAME in $NAMESPACE..."

# 1. 產生新密碼
NEW_PASSWORD=$(openssl rand -base64 32)

# 2. 建立 Secret YAML
cat <<EOF > /tmp/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: $SECRET_NAME
  namespace: $NAMESPACE
stringData:
  password: "$NEW_PASSWORD"
EOF

# 3. 加密
kubeseal < /tmp/secret.yaml > manifests/$NAMESPACE/$SECRET_NAME-sealed.yaml

# 4. 提交
git add manifests/$NAMESPACE/$SECRET_NAME-sealed.yaml
git commit -m "chore: rotate $SECRET_NAME"
git push

# 5. 清理
rm /tmp/secret.yaml

echo "✅ Secret rotated. New password: $NEW_PASSWORD"
```

---

## 給團隊的觀念分享

### 完美的安全不存在,重點是「夠安全」

Sealed Secrets 不是最完美的方案 (例如私鑰輪替困難),但對我們的規模來說「夠安全」:

- ✅ Secret 不在 Git 中明文存在
- ✅ 即使 Git Repository 洩漏,攻擊者也無法解密
- ✅ 審計追蹤:誰在什麼時候更新了 Secret

**過度設計的安全,往往導致團隊無法執行。**

### Secret 管理是持續演進的

我們選擇 Sealed Secrets 作為起點,但未來可能會遷移到 Vault:

```
Phase 1 (現在): Sealed Secrets (簡單,快速)
Phase 2 (6個月後): 評估 Vault (集中管理,自動輪替)
Phase 3 (1年後): 整合雲端 Secret 服務 (Azure Key Vault)
```

**不要一開始就追求最完美的方案,先解決當下的問題。**

---

## 下週預告

Secret 管理建立後,下週要探索 **多集群管理規劃**。

主題內容:

- 如何用一個 ArgoCD 管理多個 Kubernetes 集群
- 如何派發 Manifest 到遠端集群
- 如何實現「中央控制,分散執行」

這是大型企業的必備能力,也是 ArgoCD 的強項。

---

**作者的話:**  
這週建立 Sealed Secrets 時,團隊終於能把「完整的配置」都放進 Git 了。那種「終於沒有例外」的感覺很棒。GitOps 的最後一塊拼圖,完成!

**Tags:** #Sealed-Secrets #Secret-Management #GitOps #Security #Encryption
