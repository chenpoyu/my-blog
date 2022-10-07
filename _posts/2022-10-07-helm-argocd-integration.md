---
layout: post
title: "Helm 與 ArgoCD 的完美結合"
date: 2022-10-07 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [Helm, ArgoCD, Value Files, Template, GitOps]
---

進入十月,我們要開始處理一個實際問題:**如何管理多環境的配置差異?**

之前的文章中,我們用 Plain YAML 來管理 Manifest,但隨著應用數量增加,會發現大量重複。這週要引入 **Helm**,它能透過模板化 (Templating) 來重用配置,並透過 `values.yaml` 管理不同環境的差異。

ArgoCD 原生支援 Helm,但如何設計 Value Files 的結構、如何避免 Helm 的陷阱,這些都需要經驗。

---

## 本週目標

將現有的 Plain YAML Manifest 轉換為 Helm Chart,並透過 ArgoCD 部署。

---

## 技術實作重點

### 1. 為什麼需要 Helm?

**問題場景:**

我們有 3 個環境 (Dev/Staging/Production),每個環境的差異:

| 環境 | Replicas | CPU Request | Memory Limit | Image Tag |
|------|----------|-------------|--------------|-----------|
| Dev | 1 | 50m | 128Mi | latest |
| Staging | 2 | 100m | 256Mi | v1.2.3-rc |
| Production | 3 | 200m | 512Mi | v1.2.3 |

**Plain YAML 的問題:**

需要維護 3 份幾乎相同的 `deployment.yaml`,只有幾個欄位不同。

**Helm 的解決方案:**

用**模板 (Template)** + **變數 (Values)** 來管理:

```
Chart (Template) + values-dev.yaml = Dev Manifest
Chart (Template) + values-prod.yaml = Prod Manifest
```

### 2. 建立 Helm Chart 結構

```
helm-charts/
└── app-a/
    ├── Chart.yaml
    ├── values.yaml          # 預設值
    ├── values-dev.yaml      # Dev 覆蓋
    ├── values-staging.yaml  # Staging 覆蓋
    ├── values-prod.yaml     # Production 覆蓋
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

### 3. Chart.yaml

```yaml
apiVersion: v2
name: app-a
description: My Application A
type: application
version: 1.0.0
appVersion: "1.2.3"
```

### 4. templates/deployment.yaml (模板化)

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.appName }}
    env: {{ .Values.environment }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
        env: {{ .Values.environment }}
    spec:
      containers:
      - name: {{ .Values.appName }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
```
{% endraw %}

### 5. values.yaml (基礎預設值)

```yaml
appName: app-a
namespace: default
environment: dev
replicaCount: 1

image:
  repository: myregistry.azurecr.io/app-a
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

env:
  - name: LOG_LEVEL
    value: info
```

### 6. values-prod.yaml (Production 覆蓋)

```yaml
namespace: production
environment: production
replicaCount: 3

image:
  tag: v1.2.3  # 固定版本,不用 latest

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi

env:
  - name: LOG_LEVEL
    value: warn  # Production 降低日誌等級
  - name: DATABASE_URL
    value: "postgres://prod-db:5432/myapp"
```

### 7. 在 ArgoCD 中使用 Helm

#### 方式一:透過 CLI 建立

```bash
argocd app create app-a-prod \
  --repo https://github.com/my-org/helm-charts.git \
  --path app-a \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --helm-set-file values=values-prod.yaml \
  --sync-policy automated
```

#### 方式二:透過 Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/helm-charts.git
    targetRevision: main
    path: app-a
    helm:
      # 指定使用哪個 values 檔案
      valueFiles:
      - values-prod.yaml
      
      # 也可以直接覆蓋特定值
      parameters:
      - name: image.tag
        value: v1.2.4
      
      # 忽略 Helm Hooks (避免 Job 重複執行)
      skipCrds: false
  
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

### 8. 驗證 Helm 渲染結果

在推送到 ArgoCD 之前,先本地驗證:

```bash
# 渲染 Dev 環境
helm template app-a ./helm-charts/app-a -f ./helm-charts/app-a/values-dev.yaml

# 渲染 Production 環境
helm template app-a ./helm-charts/app-a -f ./helm-charts/app-a/values-prod.yaml
```

**輸出應該是合法的 Kubernetes YAML。**

### 9. CI Pipeline 整合

Jenkins 更新 Image Tag 時,不再修改 YAML,而是修改 `values-prod.yaml`:

```groovy
stage('Update Helm Values') {
    steps {
        sh """
            cd helm-charts/app-a
            
            # 使用 yq 更新 image.tag
            yq eval '.image.tag = "${IMAGE_TAG}"' -i values-${TARGET_ENV}.yaml
            
            git add values-${TARGET_ENV}.yaml
            git commit -m "chore: update app-a image to ${IMAGE_TAG} in ${TARGET_ENV}"
            git push
        """
    }
}
```

---

## 遇到的挑戰與對策

### 挑戰一:Helm Values 的繼承關係混亂

當我們有 `values.yaml` + `values-prod.yaml` 時,最終的值是如何合併的?

**規則:**

```
Final Values = values.yaml (Base) + values-prod.yaml (Override)
```

**範例:**

```yaml
# values.yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# values-prod.yaml
resources:
  requests:
    cpu: 200m  # 只覆蓋 cpu
```

**最終結果:**

```yaml
resources:
  requests:
    cpu: 200m      # ← 被 prod 覆蓋
    memory: 128Mi  # ← 保留 base 的值
  limits:
    cpu: 500m
    memory: 512Mi
```

**對策:** 明確記錄「哪些值會被覆蓋」,避免團隊誤解。

### 挑戰二:Helm Hooks 導致 Sync 失敗

Helm Chart 中可能包含 Hooks (如 pre-install Job),ArgoCD 每次 Sync 都會重新執行,導致錯誤。

**解決方案:**

在 ArgoCD Application 中設定:

```yaml
source:
  helm:
    skipCrds: false
    # 忽略特定 Hooks
    ignoreMissingValueFiles: true
```

或是在 Helm Template 中加入條件判斷:

{% raw %}
```yaml
{{- if .Values.hooks.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-job
  annotations:
    helm.sh/hook: pre-upgrade
{{- end }}
```
{% endraw %}

### 挑戰三:如何管理 Secret?

`values-prod.yaml` 中可能包含敏感資訊 (如 Database Password),不能直接放在 Git 中。

**對策 (下下週會詳細講):**

1. **使用 Sealed Secrets:** 加密後的 Secret 可以放 Git
2. **使用 External Secrets Operator:** 從 Vault/AWS Secrets Manager 拉取
3. **使用 ArgoCD Secret Management Plugin**

---

## 給團隊的觀念分享

### Helm 不是銀彈

Helm 很強大,但也有陷阱:

**優點:**

- ✅ 減少重複代碼
- ✅ 易於管理多環境
- ✅ 社群有大量現成 Chart (如 nginx, mysql)

**缺點:**

- ❌ 模板語法複雜 (Go Template)
- ❌ 過度使用會導致「配置即程式碼」,難以理解
- ❌ Helm Release 狀態管理有時會出問題

**建議:**

- 簡單應用:用 Plain YAML 或 Kustomize
- 複雜應用且需要跨環境重用:用 Helm
- 不要為了用 Helm 而用 Helm

### Value Files 的設計哲學

我們選擇 `values-{env}.yaml` 而非 `values/{env}/values.yaml`,理由:

1. **扁平化結構更清晰**
2. **容易在 ArgoCD 中指定** (`valueFiles: [values-prod.yaml]`)
3. **避免過深的目錄嵌套**

**但也有團隊選擇目錄結構,各有優缺點。重點是「一致性」。**

---

## 下週預告

Helm 導入後,下週要探索 ArgoCD 的殺手級功能:**App-of-Apps 模式**。

這是資深工程師必學的進階技巧,能讓你:

- 用一個 Application 管理多個 Application
- 實現「集群初始化」自動化
- 建立應用依賴關係

**這也是我們實現「Infrastructure as Code」的關鍵一步。**

---

**作者的話:**  
這週把所有應用都轉換成 Helm Chart,過程中發現很多過去沒注意到的配置不一致問題。Helm 不只是模板工具,更是一種「強制標準化」的機制。下週的 App-of-Apps 更精彩,準備好你的筆記!

**Tags:** #Helm #ArgoCD #Value-Files #Template #Multi-Environment
