---
layout: post
title: "Image 更新自動化"
date: 2022-11-25 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, Image Updater, Automation, CI/CD, Tagging Strategy]
---

這週要解決一個實務痛點:**如何讓 CI 推送 Image 後,ArgoCD 自動更新 Manifest?**

目前的流程是:Jenkins 建置完成 → 手動更新 Git Manifest → ArgoCD 同步。今天要引入 **ArgoCD Image Updater**,實現「CI 推送 Image,CD 自動更新」的完整閉環。

---

## 本週目標

安裝 ArgoCD Image Updater,建立自動化的 Image 更新機制。

---

## 技術實作重點

### 1. 為什麼需要 Image Updater?

**現狀:**

```
Jenkins CI → Build Image → Push to Registry → 
手動更新 k8s-manifests/deployment.yaml → Git Push →
ArgoCD 偵測到變更 → 部署
```

**痛點:**

- ❌ 需要人工介入 (或寫複雜的 Jenkins Pipeline)
- ❌ Git Manifest Repository 被 CI 頻繁污染
- ❌ Image Tag 與 Git Commit 難以對應

**理想狀態:**

```
Jenkins CI → Build Image (tag: sha-a1b2c3d) → Push to Registry →
Image Updater 偵測到新 Image → 自動更新 Manifest → 
ArgoCD 部署
```

### 2. ArgoCD Image Updater 原理

```
1. 定期掃描 Container Registry (Docker Hub, ACR, ECR...)
2. 根據策略 (latest, semver, digest) 判斷是否有新 Image
3. 如果有新 Image:
   a. 更新 Manifest Repository (Git Commit)
   b. 或直接更新 Application 的 Image Override
4. ArgoCD 偵測到變更,觸發 Sync
```

**兩種更新模式:**

| 模式 | 原理 | 優點 | 缺點 |
|------|------|------|------|
| **Git Write-back** | 直接 Commit 到 Git | 符合 GitOps 精神 | 產生大量 Commit |
| **ArgoCD Parameter Override** | 只更新 Application CRD | 不污染 Git | 實際 Image 與 Git 不一致 |

**我們選擇:Git Write-back (更符合 GitOps)**

### 3. 安裝 Image Updater

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

**驗證安裝:**

```bash
kubectl get pods -n argocd | grep image-updater

# 輸出:
# argocd-image-updater-xxx   1/1     Running   0   30s
```

### 4. 配置 Registry 認證

Image Updater 需要能連到 Container Registry 拉取 Image Tags。

#### 為 Azure Container Registry (ACR) 配置

```bash
# 建立 Secret
kubectl create secret docker-registry acr-creds \
  --docker-server=myregistry.azurecr.io \
  --docker-username=<service-principal-id> \
  --docker-password=<service-principal-password> \
  --namespace argocd
```

#### 配置 Image Updater

```bash
kubectl edit configmap argocd-image-updater-config -n argocd
```

```yaml
data:
  registries.conf: |
    registries:
    - name: ACR
      prefix: myregistry.azurecr.io
      api_url: https://myregistry.azurecr.io
      credentials: secret:argocd/acr-creds
      default: true
```

### 5. 為 Application 啟用 Image Updater

**在 Application Annotation 中加入配置:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-dev
  namespace: argocd
  annotations:
    # 🔹 啟用 Image Updater
    argocd-image-updater.argoproj.io/image-list: app=myregistry.azurecr.io/app-a
    
    # 🔹 更新策略
    argocd-image-updater.argoproj.io/app.update-strategy: latest
    
    # 🔹 允許的 Tag 模式 (避免拉到 test 或 debug 標籤)
    argocd-image-updater.argoproj.io/app.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    
    # 🔹 Git Write-back 配置
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ... (省略其他配置)
```

### 6. Image Tagging 策略

**策略一:latest (不推薦生產環境)**

```yaml
argocd-image-updater.argoproj.io/app.update-strategy: latest
```

**問題:**

- 無法回溯「這個版本對應哪個代碼」
- 無法回滾到特定版本

**適用:**  開發環境,追求「永遠是最新的」。

---

**策略二:Semantic Versioning (推薦)**

```yaml
argocd-image-updater.argoproj.io/app.update-strategy: semver
argocd-image-updater.argoproj.io/app.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
```

**範例:**

```
Registry 中的 Tags:
- v1.2.3
- v1.2.4
- v1.3.0
- debug-test

Image Updater 會選擇:v1.3.0 (最新的符合 semver 的版本)
```

**適用:**  生產環境,有明確的版本管理。

---

**策略三:Digest (SHA256) (最安全)**

```yaml
argocd-image-updater.argoproj.io/app.update-strategy: digest
```

**範例:**

```
Image: myregistry.azurecr.io/app-a:v1.2.3
Digest: sha256:a1b2c3d4e5f6...

即使 Tag 被覆蓋 (v1.2.3 指向新的 Image),Digest 不變
```

**適用:**  對安全要求極高的場景,確保 Image 不被篡改。

### 7. CI Pipeline 整合

**Jenkins 不再需要更新 Git Manifest:**

```groovy
pipeline {
    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
                sh 'mvn test'
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                script {
                    def version = sh(returnStdout: true, script: 'cat VERSION').trim()
                    def imageTag = "v${version}"
                    
                    sh """
                        docker build -t myregistry.azurecr.io/app-a:${imageTag} .
                        docker push myregistry.azurecr.io/app-a:${imageTag}
                    """
                    
                    echo "✅ Image pushed: ${imageTag}"
                    echo "🤖 ArgoCD Image Updater will detect and deploy automatically"
                }
            }
        }
        
        // ✅ 不再需要 "Update Manifest" Stage!
    }
}
```

### 8. 監控 Image Updater 運作

#### 查看日誌

```bash
kubectl logs -n argocd deployment/argocd-image-updater -f
```

**正常運作的日誌:**

```
time="2022-11-25T10:30:00Z" level=info msg="Checking application app-a-dev for updated images"
time="2022-11-25T10:30:02Z" level=info msg="Found new image: myregistry.azurecr.io/app-a:v1.2.4"
time="2022-11-25T10:30:03Z" level=info msg="Updating deployment.yaml in Git"
time="2022-11-25T10:30:05Z" level=info msg="Successfully pushed commit to main branch"
```

#### 查看 Git History

```bash
cd k8s-manifests
git log --oneline

# 輸出:
# a1b2c3d build: automatic update of app-a (myregistry.azurecr.io/app-a:v1.2.4)
# d4e5f6g build: automatic update of app-a (myregistry.azurecr.io/app-a:v1.2.3)
```

---

## 遇到的挑戰與對策

### 挑戰一:Git Commit 噪音過多

每次 Image 更新都產生一個 Commit,導致 Git History 混亂。

**對策:**

**方式一:使用獨立分支**

```yaml
argocd-image-updater.argoproj.io/git-branch: image-updates
```

定期 Merge 到 `main`,而非直接 Commit 到 `main`。

**方式二:使用 Squash Commit**

定期執行:

```bash
git rebase -i HEAD~100  # 合併最近 100 個 Commit
```

### 挑戰二:如何避免「無限循環」?

Image Updater 更新 Git → ArgoCD Sync → Pod 重啟 → 但 Image 沒變 → Image Updater 又更新一次...

**對策:**

確保 `allow-tags` 設定正確,避免匹配到相同的 Tag:

```yaml
argocd-image-updater.argoproj.io/app.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
```

而不是:

```yaml
argocd-image-updater.argoproj.io/app.allow-tags: regexp:.*  # ❌ 太寬鬆
```

### 挑戰三:Registry 掃描頻率太高

預設每 2 分鐘掃描一次,對 Registry API 有壓力。

**對策:**

調整掃描間隔:

```yaml
# argocd-image-updater-config
data:
  interval: 10m  # 改為 10 分鐘一次
```

---

## 給團隊的觀念分享

### 自動化要適度

Image Updater 很強大,但不是所有環境都適合:

- ✅ **Dev 環境:** 自動更新,快速反饋
- ⚠️ **Staging 環境:** 自動更新,但需要嚴格的測試
- ❌ **Production 環境:** 不建議自動更新,改用手動審核

**生產環境的變更,應該由人決定,而非機器。**

### Tag 策略決定了可維護性

**錯誤的 Tag 策略:**

```
latest
test
debug
```

**正確的 Tag 策略:**

```
v1.2.3 (Semantic Version)
sha-a1b2c3d (Git Commit Hash)
20221125-103045 (Build Timestamp)
```

**好的 Tag 策略,讓你能回答:**

- 這個 Image 對應哪個代碼版本?
- 如何回滾到上一個穩定版本?
- 哪些 Image 是測試用的,哪些是生產用的?

---

## 下週預告

十一月結束,進入十二月的最後衝刺。下週主題:**首個微服務遷移與零停機部署**。

我們要:

- 選擇一個真實的微服務進行完整遷移
- 實現 Blue-Green Deployment
- 驗證零停機部署的可行性

這是理論走向實踐的關鍵一步。

---

**作者的話:**  
這週配置 Image Updater 後,終於實現了「完全自動化」的部署流程。從代碼提交到生產部署,中間不需要任何人工介入 (當然,生產環境我們還是保留了手動審核)。這就是 GitOps 的終極目標。

**Tags:** #ArgoCD #Image-Updater #Automation #Tagging-Strategy #CI-CD
