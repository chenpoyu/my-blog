---
layout: post
title: "第一條 CI 與 CD 的橋樑"
date: 2022-09-16 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [Jenkins, ArgoCD, CI/CD, Git Integration, Automation]
---

經過前兩週的 ArgoCD 安裝與 Manifest Repository 設計,今天要迎來最激動人心的時刻:**打通 CI 與 CD 的完整流程**。

從這週開始,當開發者推送程式碼後,整個流程會自動運轉:Jenkins 建置 → Docker Image 推送 → 更新 Git Manifest → ArgoCD 自動同步 → Pod 更新。這就是 GitOps 的魔力——**Git 成為唯一的真相來源,所有變更都有紀錄,所有部署都可追蹤**。

---

## 本週目標

實現端到端的 GitOps 流程:從代碼提交到應用部署,全程自動化且可追蹤。

---

## 技術實作重點

### 1. 整體流程設計

```
┌──────────────┐
│  Developer   │
│  git push    │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────┐
│  GitHub (Application Repository)     │
│  https://github.com/my-org/app-a     │
└──────┬───────────────────────────────┘
       │ (Webhook)
       ▼
┌──────────────────────────────────────┐
│  Jenkins CI                          │
│  1. mvn clean package                │
│  2. docker build & push              │
│  3. Update k8s-manifests repo        │ ← 關鍵步驟
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  GitHub (Manifest Repository)        │
│  https://github.com/my-org/k8s-man.. │
└──────┬───────────────────────────────┘
       │ (Webhook / Polling)
       ▼
┌──────────────────────────────────────┐
│  ArgoCD                              │
│  1. Detect Git change                │
│  2. Compare with K8s state           │
│  3. Auto Sync (kubectl apply)        │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Kubernetes Cluster                  │
│  Pod updated to new version          │
└──────────────────────────────────────┘
```

**核心原則:**

- Jenkins **不再直接操作 Kubernetes**
- 所有部署邏輯由 ArgoCD 控制
- Git Commit 成為「部署指令」

### 2. Jenkins Pipeline 實作

#### Jenkinsfile (Application Repository)

```groovy
@Library('shared-pipeline-library') _

pipeline {
    agent any
    
    environment {
        APP_NAME = 'app-a'
        REGISTRY = 'myregistry.azurecr.io'
        MANIFEST_REPO = 'git@github.com:my-org/k8s-manifests.git'
        MANIFEST_REPO_BRANCH = 'main'
        TARGET_ENV = 'staging'  // 可以從參數傳入
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
                sh 'mvn test'
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                script {
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    def imageTag = "${env.BUILD_NUMBER}-${gitCommit}"
                    
                    sh """
                        docker build -t ${REGISTRY}/${APP_NAME}:${imageTag} .
                        docker push ${REGISTRY}/${APP_NAME}:${imageTag}
                    """
                    
                    env.IMAGE_TAG = imageTag
                }
            }
        }
        
        stage('Update Manifest') {
            steps {
                script {
                    updateGitManifest(
                        manifestRepo: env.MANIFEST_REPO,
                        manifestBranch: env.MANIFEST_REPO_BRANCH,
                        appName: env.APP_NAME,
                        environment: env.TARGET_ENV,
                        imageTag: env.IMAGE_TAG
                    )
                }
            }
        }
        
        stage('Notify') {
            steps {
                echo "✅ Image ${env.IMAGE_TAG} pushed and manifest updated."
                echo "🔄 ArgoCD will sync automatically in ~30 seconds."
            }
        }
    }
    
    post {
        failure {
            echo "❌ Pipeline failed. Manifest not updated."
        }
    }
}
```

### 3. Shared Library 實作 (關鍵邏輯)

**`vars/updateGitManifest.groovy`:**

```groovy
def call(Map config) {
    def manifestRepo = config.manifestRepo
    def manifestBranch = config.manifestBranch
    def appName = config.appName
    def environment = config.environment
    def imageTag = config.imageTag
    
    // 使用 SSH Key (預先配置在 Jenkins Credentials)
    sshagent(['github-ssh-key']) {
        sh """
            set -e
            
            # Clone manifest repository
            rm -rf manifest-repo
            git clone -b ${manifestBranch} ${manifestRepo} manifest-repo
            cd manifest-repo
            
            # 定位到正確的環境與應用目錄
            cd ${environment}/${appName}
            
            # 更新 image tag (使用 yq 或 sed)
            if command -v yq &> /dev/null; then
                # 使用 yq (更精準)
                yq eval '.spec.template.spec.containers[0].image = \"${REGISTRY}/${appName}:${imageTag}\"' -i deployment.yaml
            else
                # 使用 sed (備用方案)
                sed -i 's|image: .*/${appName}:.*|image: ${REGISTRY}/${appName}:${imageTag}|' deployment.yaml
            fi
            
            # 檢查是否有變更
            if git diff --exit-code deployment.yaml; then
                echo "ℹ️ No changes detected. Skipping commit."
                exit 0
            fi
            
            # Commit and Push
            git config user.name "Jenkins CI"
            git config user.email "jenkins@company.com"
            git add deployment.yaml
            git commit -m "chore(${environment}): update ${appName} to ${imageTag}
            
            Triggered by: ${env.BUILD_URL}
            Commit: ${env.GIT_COMMIT}"
            
            # Retry logic for push (避免衝突)
            for i in {1..3}; do
                if git push origin ${manifestBranch}; then
                    echo "✅ Manifest updated successfully."
                    exit 0
                else
                    echo "⚠️ Push failed (attempt \$i/3). Retrying..."
                    git pull --rebase origin ${manifestBranch}
                    sleep 5
                fi
            done
            
            echo "❌ Failed to push after 3 attempts."
            exit 1
        """
    }
}
```

**關鍵設計:**

1. **使用 SSH Key:** 避免在 Pipeline 中暴露 Token
2. **Commit Message 包含 Trigger 資訊:** 方便追蹤「這次部署是誰觸發的」
3. **Retry Logic:** 處理多個 Pipeline 同時更新的衝突情況
4. **檢查 Diff:** 避免產生空 Commit (當 Image Tag 沒變時)

### 4. ArgoCD Application 配置

在 ArgoCD 中建立 Application,監控 Manifest Repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-staging
  namespace: argocd
spec:
  project: default
  
  # Git Repository 配置
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: staging/app-a
  
  # 部署目標
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  
  # 自動同步策略
  syncPolicy:
    automated:
      prune: true       # 自動刪除 Git 中不存在的資源
      selfHeal: true    # 自動修復手動變更
      allowEmpty: false # 防止誤刪所有資源
    syncOptions:
    - CreateNamespace=true  # 自動建立 Namespace
    
  # 重試策略
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

**透過 CLI 建立:**

```bash
argocd app create app-a-staging \
  --repo https://github.com/my-org/k8s-manifests.git \
  --path staging/app-a \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace staging \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 5. 配置 Webhook (即時同步)

預設 ArgoCD 每 3 分鐘輪詢 Git,太慢了。配置 Webhook 讓 Git 主動通知:

**在 GitHub Repository 設定:**

1. 進入 `k8s-manifests` Repository → Settings → Webhooks
2. 新增 Webhook:
   - **Payload URL:** `https://argocd.example.com/api/webhook`
   - **Content type:** `application/json`
   - **Secret:** (從 ArgoCD 取得)
   - **Events:** Just the push event

**取得 ArgoCD Webhook Secret:**

```bash
kubectl get secret argocd-secret -n argocd -o jsonpath="{.data.webhook\.github\.secret}" | base64 -d
```

**驗證 Webhook:**

推送一個測試 Commit 到 `k8s-manifests`,檢查 GitHub 的 Webhook 頁面是否顯示 `200 OK`。

---

## 遇到的挑戰與對策

### 挑戰一:Git Push 衝突頻繁

當多個 Pipeline 同時更新 Manifest Repository 時,經常出現 `[rejected] main -> main (fetch first)` 錯誤。

**對策:**

在 Shared Library 中加入 **Pull-Rebase-Push** 邏輯:

```bash
for i in {1..3}; do
    if git push origin main; then
        break
    else
        git pull --rebase origin main
        sleep $((RANDOM % 5 + 1))  # 隨機等待 1-5 秒
    fi
done
```

### 挑戰二:如何確認 ArgoCD 同步成功?

Jenkins 更新完 Git 後就結束了,但無法確認 ArgoCD 是否真的部署成功。

**對策:**

在 Pipeline 最後加入「等待並驗證」邏輯:

```groovy
stage('Wait for ArgoCD Sync') {
    steps {
        script {
            timeout(time: 5, unit: 'MINUTES') {
                waitUntil {
                    def syncStatus = sh(
                        script: "argocd app get app-a-staging -o json | jq -r '.status.sync.status'",
                        returnStdout: true
                    ).trim()
                    
                    def healthStatus = sh(
                        script: "argocd app get app-a-staging -o json | jq -r '.status.health.status'",
                        returnStdout: true
                    ).trim()
                    
                    return syncStatus == 'Synced' && healthStatus == 'Healthy'
                }
            }
        }
    }
}
```

但這樣做有爭議:**Jenkins 不應該等待 K8s 部署完成**。我們最終決定讓 Pipeline 結束,由 Slack 通知來告知同步結果。

---

## 給團隊的觀念分享

### CI 與 CD 的邊界要清晰

這週最大的收穫是:**不要讓 CI 工具去監控 CD 的結果**。

Jenkins 的職責是:

- ✅ Build Code
- ✅ Run Tests
- ✅ Push Image
- ✅ Update Git Manifest

至於「部署是否成功」,應該由 ArgoCD + Monitoring (Prometheus/Grafana) 來負責。

**不要把所有邏輯塞進 Pipeline,這樣會變得難以維護。**

### Git Commit Message 是文件

注意到我們在 Commit Message 中加入了 `Triggered by` 和 Build URL 嗎?這不是多餘的,而是**可追溯性 (Traceability)** 的體現:

```
chore(staging): update app-a to 123-a1b2c3d

Triggered by: https://jenkins.example.com/job/app-a/123/
Commit: a1b2c3d4e5f6
```

當線上出問題時,你可以立刻回答:

- 這個部署是誰觸發的? → 從 Jenkins Build URL 追蹤
- 對應的應用程式代碼是哪個版本? → 從 Git Commit Hash 追蹤
- 為什麼部署? → 從原始 Application Repo 的 Commit Message 看

**這就是 GitOps 的價值:所有資訊都在 Git 中。**

---

## 下週預告

第一條 CI/CD 橋樑打通後,下週要開始探索 ArgoCD 的進階功能:**初試 Sync Policy**。

我們要實驗:

- `prune`: 自動刪除多餘的資源
- `selfHeal`: 自動修復手動變更
- `syncOptions`: 控制同步行為

這些功能將決定 ArgoCD 如何「主動維護」Kubernetes 的狀態,而不是被動地執行指令。

---

**作者的話:**  
這週最激動人心的時刻,是看到 Jenkins 完成 Build 後,ArgoCD UI 中的 Application 自動變成 `Syncing` 狀態,然後幾秒後 Pod 就更新了。這種「自動化的美感」,只有親身體驗過才能理解。GitOps,真香!

**Tags:** #Jenkins #ArgoCD #GitOps #CI-CD #Automation #Git-Integration
