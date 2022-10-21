---
layout: post
title: "多環境管理策略 (Dev/Staging/Production)"
date: 2022-10-21 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [Kustomize, Overlays, Multi-Environment, GitOps]
---

上週的 App-of-Apps 解決了「如何管理多個應用」,這週要解決另一個實際問題:**如何優雅地管理多環境的配置差異?**

我們有 Dev、Staging、Production 三個環境,每個環境的差異不大,但又不能完全相同。如果用 Helm,需要維護多個 values 檔案;如果用 Plain YAML,會有大量重複。這週要介紹 **Kustomize Overlays**,它是專門為「基於覆蓋的配置管理」設計的工具。

---

## 本週目標

使用 Kustomize Overlays 管理多環境配置,實現配置重用與環境隔離。

---

## 技術實作重點

### 1. Kustomize vs. Helm:如何選擇?

| 特性 | Kustomize | Helm |
|------|-----------|------|
| **學習曲線** | 低 (YAML + Patch) | 高 (Go Template) |
| **適用場景** | 配置差異小 | 配置差異大,需要模板化 |
| **社群生態** | K8s 原生 | 龐大的 Chart 生態 |
| **複雜度** | 簡單直觀 | 模板語法複雜 |

**我們的策略:**

- **自己開發的應用:** Kustomize (差異小,易維護)
- **第三方套件 (如 Prometheus):** Helm (利用社群 Chart)

### 2. Kustomize 的核心概念

```
Base (基礎配置,所有環境共用)
  ↓
Overlay (覆蓋配置,針對特定環境)
  ↓
Final Manifest (最終產生的 YAML)
```

**範例:**

```
Base: replicas = 2
Dev Overlay: replicas = 1
Final Dev Manifest: replicas = 1
```

### 3. Repository 結構

```
k8s-manifests/
└── apps/
    └── app-a/
        ├── base/
        │   ├── kustomization.yaml
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── configmap.yaml
        └── overlays/
            ├── dev/
            │   ├── kustomization.yaml
            │   └── replica-patch.yaml
            ├── staging/
            │   └── kustomization.yaml
            └── production/
                ├── kustomization.yaml
                ├── resource-patch.yaml
                └── ingress.yaml  # Production 才需要的資源
```

### 4. Base 配置

#### base/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
spec:
  replicas: 2  # 預設值
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
        image: myregistry.azurecr.io/app-a:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        envFrom:
        - configMapRef:
            name: app-a-config
```

#### base/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-a-svc
spec:
  selector:
    app: app-a
  ports:
  - port: 80
    targetPort: 8080
```

#### base/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-a-config
data:
  LOG_LEVEL: "info"
  DATABASE_URL: "placeholder"
```

#### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

# 共同的 Labels (所有環境)
commonLabels:
  app: app-a
  managed-by: argocd
```

### 5. Overlay 配置

#### overlays/dev/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 基於 base
bases:
- ../../base

# Dev 環境特定的標籤
commonLabels:
  environment: dev

# Namespace
namespace: dev

# 修改 replicas
replicas:
- name: app-a
  count: 1  # Dev 只需要 1 個 Pod

# 修改 Image
images:
- name: myregistry.azurecr.io/app-a
  newTag: latest  # Dev 用 latest

# ConfigMap 覆蓋
configMapGenerator:
- name: app-a-config
  behavior: merge  # 合併而非替換
  literals:
  - LOG_LEVEL=debug  # Dev 提高日誌等級
  - DATABASE_URL=postgres://dev-db:5432/myapp
```

#### overlays/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

commonLabels:
  environment: production

namespace: production

replicas:
- name: app-a
  count: 3  # Production 需要 HA

images:
- name: myregistry.azurecr.io/app-a
  newTag: v1.2.3  # Production 用固定版本

# 新增資源 (只有 Production 需要 Ingress)
resources:
- ingress.yaml

# Patch 資源限制
patchesStrategicMerge:
- resource-patch.yaml

configMapGenerator:
- name: app-a-config
  behavior: merge
  literals:
  - LOG_LEVEL=warn
  - DATABASE_URL=postgres://prod-db:5432/myapp
```

#### overlays/production/resource-patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

#### overlays/production/ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-a-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
  - host: app-a.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-a-svc
            port:
              number: 80
  tls:
  - hosts:
    - app-a.example.com
    secretName: app-a-tls
```

### 6. 本地驗證

```bash
# 查看 Dev 環境的最終 YAML
kustomize build overlays/dev

# 查看 Production 環境的最終 YAML
kustomize build overlays/production

# 比對差異
diff <(kustomize build overlays/dev) <(kustomize build overlays/production)
```

### 7. ArgoCD 整合

#### Application CRD (Dev)

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
    path: apps/app-a/overlays/dev  # 指向 overlay
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

#### Application CRD (Production)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: apps/app-a/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 8. CI Pipeline 更新 Image Tag

Jenkins 更新 Production Image Tag 時:

```bash
cd k8s-manifests/apps/app-a/overlays/production

# 使用 kustomize edit 更新 image
kustomize edit set image myregistry.azurecr.io/app-a:v1.2.4

git add kustomization.yaml
git commit -m "chore: update app-a to v1.2.4 in production"
git push
```

---

## 遇到的挑戰與對策

### 挑戰一:Patch 語法複雜

Kustomize 的 Strategic Merge Patch 有時難以理解。

**對策:**

使用 **JSON6902 Patch** (更精確):

```yaml
# overlays/production/kustomization.yaml
patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: app-a
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
    - op: replace
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: 200m
```

### 挑戰二:如何避免 Production 配置被誤改?

**對策:使用 CODEOWNERS**

```
# .github/CODEOWNERS
/apps/*/overlays/production/** @tech-lead @ops-team
/apps/*/overlays/staging/** @team-members
```

Production 的變更必須經過 Tech Lead 審核。

### 挑戰三:Base 改了會影響所有環境

當修改 `base/deployment.yaml` 時,所有環境都會受影響,風險高。

**對策:**

1. **充分測試:** 先在 Dev 驗證,再推到 Staging,最後才 Production
2. **使用 Git Branch:** Production 用 `main` 分支,Dev 用 `dev` 分支
3. **監控變更:** 設定 Slack 通知,當 Base 有變更時提醒團隊

---

## 給團隊的觀念分享

### DRY 原則的實踐

Kustomize 完美實現了 **Don't Repeat Yourself**:

- Base 定義共同邏輯 (避免重複)
- Overlay 只描述差異 (最小化變更)

**對比 Plain YAML:**

- 需要維護 3 份幾乎相同的 `deployment.yaml`
- 改一個欄位,要改 3 個檔案

**Kustomize:**

- 只維護 1 份 Base + 3 份小的 Overlay
- 改共同邏輯,只改 Base

### 環境的本質是「差異」

當設計多環境配置時,不要思考「每個環境有什麼」,而要思考「環境之間有什麼差異」。

**範例:**

- ❌ 錯誤思維:「Production 需要 3 個 replicas、200m CPU、1Gi Memory...」
- ✅ 正確思維:「Production 相比 Dev,replicas 多 2 個,CPU 多 100m...」

**這就是 Overlay 的哲學:基於差異構建配置。**

---

## 下週預告

多環境管理策略建立後,下週要面對一個實際痛點:**如何透過 ArgoCD UI 快速排除故障?**

主題:**日誌監控:從 ArgoCD UI 排除故障**。

我們要學習:

- Resource Tree 視覺化排錯
- 如何分析 Sync Fail 原因
- 如何用 Diff 檢視定位問題

當部署失敗時,能在 5 分鐘內找到根因,是資深工程師的必備技能。

---

**作者的話:**  
這週把所有應用都轉換成 Kustomize 結構,過程中深刻體會到「設計好的結構,比寫更多的代碼重要」。Kustomize 不只是工具,更是一種思維方式:把配置視為可組合的模組。

**Tags:** #Kustomize #Overlays #Multi-Environment #GitOps #DRY
