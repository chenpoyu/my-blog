---
layout: post
title: "Jenkins 瘦身計畫：回歸 CI 本質"
date: 2022-08-19 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [Jenkins, CI, Shared Library, Pipeline, Refactoring]
---

前兩週我們討論了「為什麼要轉型」和「為什麼選 ArgoCD」,這週開始動手改造現有的 CI/CD Pipeline。第一步:**讓 Jenkins 回歸 CI 的本質**。

在傳統的 Jenkins Pipeline 中,我們把 Build、Test、Deploy 全部塞在一起。但在 GitOps 架構下,Jenkins 應該只負責 **Continuous Integration**,也就是「持續集成」:建置程式碼、跑測試、打包 Image、推送 Registry。至於部署 (Continuous Deployment),則交給 ArgoCD。

這週的任務就是:**從 50 個 Jenkinsfile 中移除所有 `kubectl` 指令**。

---

## 本週目標

重構 Jenkins Pipeline,移除所有與 Kubernetes 部署相關的邏輯,並建立 Shared Library 來標準化 CI 流程。

---

## 技術實作重點

### 1. 傳統 Pipeline 的問題

先來看一個典型的 Jenkinsfile:

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t my-app:${BUILD_NUMBER} .'
                sh 'docker push my-registry/my-app:${BUILD_NUMBER}'
            }
        }
        stage('Deploy to K8s') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
                sh 'kubectl set image deployment/my-app my-app=my-registry/my-app:${BUILD_NUMBER}'
                sh 'kubectl rollout status deployment/my-app'
            }
        }
    }
}
```

**問題分析:**

1. **職責混亂:** CI (Build/Test) 和 CD (Deploy) 混在一起
2. **權限風險:** Jenkins 需要持有 Kubeconfig
3. **重複代碼:** 50 個專案都有類似的 `kubectl` 邏輯
4. **難以審計:** 無法追蹤「誰部署了什麼」(因為都是 Jenkins 的 Service Account 操作的)

### 2. GitOps 架構下的 CI Pipeline

在新的架構中,Jenkins 的職責變成:

```
[1] Build Code
[2] Run Unit Tests
[3] Build Docker Image
[4] Push to Registry
[5] Update Git Manifest  ← 這是新增的步驟
```

**關鍵改變:**  
不再直接操作 Kubernetes,而是**更新 Git 中的 Manifest**,剩下的交給 ArgoCD。

### 3. 重構後的 Jenkinsfile

```groovy
@Library('shared-pipeline-library') _

pipeline {
    agent any
    
    environment {
        REGISTRY = 'my-registry.azurecr.io'
        IMAGE_NAME = 'my-app'
        GIT_MANIFEST_REPO = 'https://github.com/my-org/k8s-manifests.git'
    }
    
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
                    def imageTag = "${env.BUILD_NUMBER}"
                    dockerBuildAndPush(
                        registry: env.REGISTRY,
                        imageName: env.IMAGE_NAME,
                        imageTag: imageTag
                    )
                }
            }
        }
        
        stage('Update Manifest') {
            steps {
                script {
                    updateK8sManifest(
                        gitRepo: env.GIT_MANIFEST_REPO,
                        appName: env.IMAGE_NAME,
                        imageTag: "${env.BUILD_NUMBER}",
                        environment: 'staging'
                    )
                }
            }
        }
    }
}
```

**重點變化:**

1. **移除了所有 `kubectl` 指令**
2. **新增 `updateK8sManifest` 步驟:** 自動更新 Git 中的 Deployment YAML
3. **使用 Shared Library:** 減少重複代碼

### 4. Shared Library 的實作

建立一個 Git Repository (`jenkins-shared-library`),結構如下:

```
jenkins-shared-library/
├── vars/
│   ├── dockerBuildAndPush.groovy
│   └── updateK8sManifest.groovy
└── resources/
    └── manifest-template.yaml
```

**`vars/updateK8sManifest.groovy`:**

```groovy
def call(Map config) {
    def gitRepo = config.gitRepo
    def appName = config.appName
    def imageTag = config.imageTag
    def environment = config.environment
    
    sh """
        # Clone manifest repository
        git clone ${gitRepo} manifest-repo
        cd manifest-repo
        
        # Update image tag in deployment.yaml
        sed -i 's|image: .*/${appName}:.*|image: ${REGISTRY}/${appName}:${imageTag}|' \
            ${environment}/${appName}/deployment.yaml
        
        # Commit and push
        git config user.name "Jenkins CI"
        git config user.email "jenkins@company.com"
        git add .
        git commit -m "chore: update ${appName} to ${imageTag} in ${environment}"
        git push origin main
    """
    
    echo "✅ Manifest updated. ArgoCD will sync automatically."
}
```

**為什麼要用 Shared Library?**

1. **統一標準:** 所有專案使用相同的更新邏輯
2. **容易維護:** 改一次,所有 Pipeline 都生效
3. **降低門檻:** 開發者不需要理解 Git 操作細節

---

## 遇到的挑戰與對策

### 挑戰一:如何確保 Git Manifest 更新成功?

有同事問:「如果 Git Push 失敗怎麼辦?Jenkins Pipeline 會中斷嗎?」

**對策:**  
在 Shared Library 中加入錯誤處理:

```groovy
def pushRetry(int maxRetries = 3) {
    for (int i = 0; i < maxRetries; i++) {
        try {
            sh 'git push origin main'
            return true
        } catch (Exception e) {
            if (i == maxRetries - 1) {
                throw e  // 最後一次失敗就拋出異常
            }
            echo "Push failed, retrying... (${i + 1}/${maxRetries})"
            sleep(5)
        }
    }
}
```

### 挑戰二:如何避免 Git Conflict?

當多個 Pipeline 同時更新 Manifest Repository 時,可能會產生衝突。

**對策:**  
採用 **Monorepo 但分目錄管理** 的策略:

```
k8s-manifests/
├── staging/
│   ├── app-a/
│   │   └── deployment.yaml
│   └── app-b/
│       └── deployment.yaml
└── production/
    ├── app-a/
    │   └── deployment.yaml
    └── app-b/
        └── deployment.yaml
```

每個應用的 Manifest 放在獨立目錄,減少衝突機會。如果真的衝突,加入 `git pull --rebase` 邏輯:

```bash
git pull --rebase origin main
git push origin main
```

### 挑戰三:開發者會不會覺得「部署變慢了」?

以前 Jenkins 執行完 `kubectl apply` 就算完成,現在要等 Git Push → ArgoCD Sync → Pod Rollout。

**對策:**  
實測結果:

- **舊流程:** Jenkins Pipeline 執行 → 5 分鐘
- **新流程:** Jenkins CI → 3 分鐘 + ArgoCD Sync → 1 分鐘 = **4 分鐘**

反而**更快**了!因為 Jenkins 不再等待 `kubectl rollout status`。

而且更重要的是:**Jenkins 不會因為 K8s 問題而失敗**。以前常遇到 K8s API Server 不穩定導致 Pipeline 失敗,現在 Jenkins 只負責推送 Git,成功率大幅提升。

---

## 給團隊的觀念分享

### CI 和 CD 的職責邊界

很多團隊混淆了 CI/CD 的概念,以為「自動化部署」就等於 CI/CD。但其實:

- **CI (Continuous Integration):** 持續整合代碼,確保每次提交都能通過建置與測試
- **CD (Continuous Delivery):** 持續交付,讓軟體隨時處於可部署狀態
- **CD (Continuous Deployment):** 持續部署,自動將軟體部署到生產環境

在 GitOps 架構下:

- **Jenkins 負責 CI:** 確保代碼品質
- **ArgoCD 負責 CD:** 確保部署一致性

**兩者不應該混在一起。**

### 標準化的價值

這次重構,我們用 Shared Library 統一了 50 個專案的 CI 流程。這帶來的好處不只是「少寫點代碼」,更重要的是:

1. **可預測性:** 所有專案的部署行為一致,降低認知負擔
2. **可維護性:** 未來要改流程,只需要修改 Shared Library
3. **可觀測性:** 統一的流程便於建立監控指標

**這就是基礎設施即代碼 (IaC) 的精神:把人工經驗轉化為可複用的自動化。**

---

## 下週預告

Jenkins 瘦身完成後,下週要開始接觸 ArgoCD 本身了。主題:**初探 ArgoCD:架構與元件分析**。

我們會深入研究 ArgoCD 的內部架構:

- Application Controller 如何監控 Git 變更?
- API Server 的角色是什麼?
- Repo Server 怎麼處理 Helm 和 Kustomize?

只有理解了這些,才能在後續的安裝與配置中做出正確的決策。

---

**作者的話:**  
這週的重構工作量不小,團隊一起改了 50 個 Jenkinsfile。但當看到所有 Pipeline 都變得簡潔且一致時,那種成就感是值得的。下週開始,我們要真正進入 ArgoCD 的世界了。

**Tags:** #Jenkins #CI #Shared-Library #Refactoring #GitOps #職責分離
