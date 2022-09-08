---
layout: post
title: "Manifest Repository 的結構設計"
date: 2022-09-09 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [GitOps, Repository Structure, Separation of Concerns, Monorepo]
---

上週完成了 ArgoCD 的安裝,這週要面對一個更需要架構思維的問題:**如何設計 Manifest Repository 的結構?**

這不是一個有標準答案的問題,不同的團隊規模、應用特性、組織文化,都會影響最佳實踐。但有一個核心原則始終不變:**程式碼與環境配置必須分離 (Separation of Concerns)**。

今天我們要探討 Monorepo vs. Multi-repo 的取捨,以及如何設計一個既易於維護、又符合 GitOps 原則的 Repository 結構。

---

## 本週目標

設計並建立團隊的 Manifest Repository 結構,為後續的 CI/CD 整合奠定基礎。

---

## 技術實作重點

### 1. 核心原則:程式碼與配置分離

**為什麼要分離?**

| 分離前 (Monolithic Repo) | 分離後 (GitOps Best Practice) |
|--------------------------|-------------------------------|
| 應用程式代碼與 K8s Manifest 在同一個 Repo | 應用程式代碼與 K8s Manifest 在不同 Repo |
| 開發者修改代碼時,可能意外改到生產環境的配置 | 配置變更需要經過獨立的 PR 審核 |
| CI Pipeline 需要同時處理 Build 與 Deploy | CI 只負責 Build,CD 由 ArgoCD 負責 |
| 無法獨立回滾配置 (必須連代碼一起回滾) | 配置可以獨立回滾 (Git Revert) |

**案例:**

假設開發者在 `application-repo` 中不小心把生產環境的 `replicas: 3` 改成 `replicas: 0`,然後推送了。如果配置與代碼在同一個 Repo,這個變更會立刻觸發部署,導致服務中斷。

但如果配置在獨立的 `manifest-repo`,這個變更需要:

1. 開 PR 到 `manifest-repo`
2. Code Review (有人會發現 replicas 被改成 0)
3. 通過審核才能 Merge

**多一層保護。**

### 2. Repository 結構設計

#### 方案一:Monorepo (單一 Repository)

```
k8s-manifests/
├── apps/
│   ├── app-a/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   └── kustomization.yaml
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       └── production/
│   │           └── kustomization.yaml
│   └── app-b/
│       └── ...
├── infra/
│   ├── ingress-nginx/
│   ├── cert-manager/
│   └── monitoring/
└── README.md
```

**優點:**

- 單一來源,容易搜尋與管理
- 適合小型團隊 (10 個以下的微服務)
- 跨應用的配置變更可以在一個 PR 中完成

**缺點:**

- 隨著應用增加,Repository 會變得龐大
- 權限管理困難 (無法針對單一應用設定不同的存取權限)
- 所有人都能看到所有配置 (可能包含敏感資訊)

#### 方案二:Multi-repo (每個應用獨立 Repository)

```
Repositories:
├── app-a-manifests/
│   ├── dev/
│   ├── staging/
│   └── production/
├── app-b-manifests/
│   └── ...
└── infra-manifests/
    ├── ingress-nginx/
    └── cert-manager/
```

**優點:**

- 權限隔離性好 (團隊 A 只能改 app-a,團隊 B 只能改 app-b)
- 適合大型組織 (50+ 微服務)
- Repository 小,Git 操作更快

**缺點:**

- 管理複雜度高 (需要維護幾十個 Repository)
- 跨應用的配置變更需要多個 PR
- 需要更完善的自動化工具

#### 我們的選擇:Monorepo + 目錄隔離

基於團隊現況 (50 個微服務,3 個環境),我們選擇 **Monorepo 但按環境與應用分目錄**:

```
k8s-manifests/
├── dev/
│   ├── app-a/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── app-b/
│   │   └── ...
│   └── kustomization.yaml  # 整個 dev 環境的入口
├── staging/
│   ├── app-a/
│   │   └── ...
│   ├── app-b/
│   │   └── ...
│   └── kustomization.yaml
├── production/
│   ├── app-a/
│   │   └── ...
│   ├── app-b/
│   │   └── ...
│   └── kustomization.yaml
└── base/  # 共用的模板
    ├── deployment-template.yaml
    └── service-template.yaml
```

**為什麼這樣設計?**

1. **環境優先:** 每個環境的配置獨立,避免誤操作
2. **易於審核:** 生產環境的變更會有 CODEOWNERS 保護
3. **支援 Kustomize:** 用 `base` + `overlays` 模式重用配置

### 3. 實際建立 Repository

#### Step 1: 建立 Git Repository

```bash
mkdir k8s-manifests
cd k8s-manifests
git init
```

#### Step 2: 建立基本結構

```bash
mkdir -p {dev,staging,production}/{app-a,app-b}
mkdir base
```

#### Step 3: 撰寫 Base Template

**`base/deployment-template.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: PLACEHOLDER_APP_NAME
spec:
  replicas: PLACEHOLDER_REPLICAS
  selector:
    matchLabels:
      app: PLACEHOLDER_APP_NAME
  template:
    metadata:
      labels:
        app: PLACEHOLDER_APP_NAME
    spec:
      containers:
      - name: app
        image: PLACEHOLDER_IMAGE
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

#### Step 4: 建立環境特定配置

**`dev/app-a/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: dev
spec:
  replicas: 1  # Dev 環境只需要 1 個 Pod
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
        image: my-registry.azurecr.io/app-a:latest
        ports:
        - containerPort: 8080
        env:
        - name: ENVIRONMENT
          value: "dev"
        resources:
          requests:
            cpu: 50m  # Dev 資源更少
            memory: 64Mi
```

**`production/app-a/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: production
spec:
  replicas: 3  # 生產環境需要 HA
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
        image: my-registry.azurecr.io/app-a:v1.2.3  # 生產環境固定版本
        ports:
        - containerPort: 8080
        env:
        - name: ENVIRONMENT
          value: "production"
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

#### Step 5: 設定 CODEOWNERS (保護生產環境)

**`.github/CODEOWNERS`:**

```
# 生產環境的變更需要 Tech Lead 審核
/production/** @tech-lead @senior-engineer

# Staging 可以由團隊成員審核
/staging/** @team-members

# Dev 環境自由變更
```

#### Step 6: 提交到 Git

```bash
git add .
git commit -m "feat: initial manifest repository structure"
git remote add origin https://github.com/my-org/k8s-manifests.git
git push -u origin main
```

---

## 遇到的挑戰與對策

### 挑戰一:如何避免配置重複?

50 個微服務,如果每個都寫一份完整的 Deployment YAML,會有大量重複。

**對策:**

使用 **Kustomize** 或 **Helm**:

- **Kustomize:** 適合配置差異小的場景 (我們選這個)
- **Helm:** 適合配置差異大且需要模板化的場景

**Kustomize 範例:**

**`base/kustomization.yaml`:**

```yaml
resources:
- deployment-template.yaml
- service-template.yaml
```

**`production/app-a/kustomization.yaml`:**

```yaml
bases:
- ../../base

nameSuffix: -prod

replicas:
- name: app-a
  count: 3

images:
- name: PLACEHOLDER_IMAGE
  newName: my-registry.azurecr.io/app-a
  newTag: v1.2.3
```

### 挑戰二:CI 如何更新 Manifest?

Jenkins CI 完成 Build 後,需要更新 Manifest Repository 中的 Image Tag。

**對策:**

在 Jenkins Shared Library 中加入邏輯:

```groovy
// vars/updateManifest.groovy
def call(String appName, String imageTag, String environment) {
    sh """
        git clone https://github.com/my-org/k8s-manifests.git
        cd k8s-manifests/${environment}/${appName}
        
        # 更新 image tag
        kustomize edit set image my-registry.azurecr.io/${appName}:${imageTag}
        
        git add .
        git commit -m "chore: update ${appName} to ${imageTag} in ${environment}"
        git push
    """
}
```

---

## 給團隊的觀念分享

### 好的結構勝過複雜的工具

這週最大的收穫是:**不要為了用工具而用工具**。

Kustomize、Helm、Jsonnet 都是好工具,但如果團隊還不熟悉,不如先用最簡單的 Plain YAML。等到真的遇到「重複太多難以維護」的問題時,再引入工具。

**過早優化是萬惡之源。**

### Repository 結構就是團隊的工作流程

當我們決定用 Monorepo + 環境隔離時,也決定了:

- 開發者可以自由改 `dev/` (快速迭代)
- Staging 需要 Code Review (確保品質)
- Production 需要 Tech Lead 審核 (嚴格控管)

**Repository 結構不只是技術問題,更是組織管理問題。**

---

## 下週預告

Manifest Repository 建立完成後,下週要打通 **CI 與 CD 的橋樑**。

我們要實現:

1. Jenkins CI 完成後,自動更新 Manifest Repository
2. ArgoCD 偵測到變更後,自動同步到 Kubernetes
3. 整個流程端到端驗證

這是 GitOps 的核心閉環,也是最激動人心的一刻。

---

**作者的話:**  
這週花了很多時間在「思考」而非「寫代碼」,但這種思考是值得的。一個好的 Repository 結構,會讓接下來半年的工作都變得順暢。記住:慢即是快。

**Tags:** #GitOps #Repository-Structure #Monorepo #Kustomize #Separation-of-Concerns
