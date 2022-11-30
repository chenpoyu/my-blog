---
layout: post
title: "Helm Charts 套件管理"
date: 2019-03-11 10:00:00 +0800
categories: [DevOps, Kubernetes]
tags: [Helm, Kubernetes, Package Management, Charts]
---

上週研究了 Kubernetes 自動擴展（參考 [Kubernetes 自動擴展](/posts/2019/03/04/kubernetes-autoscaling-hpa-vpa/)），這週來解決部署的複雜性問題。

目前部署一個應用到 Kubernetes，需要：
1. 寫 Deployment YAML
2. 寫 Service YAML
3. 寫 ConfigMap YAML
4. 寫 Ingress YAML
5. 寫 HPA YAML
6. 針對不同環境（Dev/Staging/Prod）修改一堆參數

太麻煩了！而且：
- 檔案太多，容易漏
- 難以版本管理
- 無法輕鬆分享給其他團隊
- 升級很痛苦

需要一個套件管理工具，就像 Maven 管理 Java 依賴、npm 管理 JavaScript 套件一樣。

Helm 就是 Kubernetes 的套件管理工具！

> 使用版本：Helm 2.13.0（2019 年初的穩定版）

## Helm 是什麼

Helm 幫助我們：
- **打包**：把所有 Kubernetes YAML 打包成一個 Chart
- **部署**：一個指令部署整個應用
- **升級**：輕鬆升級應用版本
- **回滾**：出問題立刻回滾到上一版
- **分享**：透過 Chart Repository 分享

核心概念：
- **Chart**：Helm 套件，包含所有 Kubernetes 資源定義
- **Release**：Chart 的部署實例
- **Repository**：存放 Charts 的地方

## 安裝 Helm

### 安裝 Helm Client

```bash
# macOS
brew install helm@2

# Linux
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash

# 驗證
helm version --client
# Client: &version.Version{SemVer:"v2.13.0", ...}
```

### 安裝 Tiller (Helm Server)

Helm 2 使用 client-server 架構，需要在 Kubernetes 安裝 Tiller。

```bash
# 初始化 Helm
helm init

# Tiller 會安裝到 kube-system namespace
kubectl get pods -n kube-system | grep tiller
# tiller-deploy-5d6cc99fc-abcde   1/1     Running   0          1m
```

檢查版本：
```bash
helm version

# Client: &version.Version{SemVer:"v2.13.0", ...}
# Server: &version.Version{SemVer:"v2.13.0", ...}
```

### 設定 RBAC（重要！）

Kubernetes 1.6+ 預設啟用 RBAC，需要給 Tiller 權限。

`tiller-rbac.yaml`：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
```

套用：
```bash
kubectl apply -f tiller-rbac.yaml

# 重新安裝 Tiller with service account
helm init --service-account tiller --upgrade
```

## 使用現有的 Charts

Helm Hub 有很多現成的 Charts，不用自己寫。

### 搜尋 Chart

```bash
helm search redis

# NAME                    CHART VERSION   APP VERSION     DESCRIPTION
# stable/redis            7.1.0           4.0.14          Open source, advanced key-value store.
# stable/redis-ha         3.4.2           5.0.3           Highly available Kubernetes implementation...
```

### 部署 Redis

```bash
helm install stable/redis --name my-redis

# NAME:   my-redis
# LAST DEPLOYED: Mon Mar 11 10:00:00 2019
# NAMESPACE: default
# STATUS: DEPLOYED
# 
# RESOURCES:
# ==> v1/ConfigMap
# NAME                    DATA  AGE
# my-redis-configuration  3     1s
# 
# ==> v1/Service
# NAME                TYPE        CLUSTER-IP      PORT(S)    AGE
# my-redis-master     ClusterIP   10.100.200.10   6379/TCP   1s
# my-redis-slave      ClusterIP   10.100.200.11   6379/TCP   1s
# 
# ==> v1/StatefulSet
# NAME              READY   AGE
# my-redis-master   0/1     1s
# my-redis-slave    0/2     1s
```

一個指令，Helm 自動建立：
- ConfigMap（Redis 設定）
- Service（Master 和 Slave）
- StatefulSet（Redis Pod）
- Secret（密碼）

等待啟動：
```bash
kubectl get pods | grep redis
# my-redis-master-0   1/1     Running   0          2m
# my-redis-slave-0    1/1     Running   0          2m
# my-redis-slave-1    1/1     Running   0          1m
```

### 查看 Release

```bash
helm list

# NAME      REVISION  STATUS    CHART         NAMESPACE
# my-redis  1         DEPLOYED  redis-7.1.0   default
```

### 自訂參數

每個 Chart 都有 `values.yaml` 定義預設值，可以覆蓋：

```bash
# 查看可設定的參數
helm inspect values stable/redis

# ... 很多參數

# 自訂安裝
helm install stable/redis --name my-redis \
  --set password=MySecretPassword \
  --set master.persistence.size=20Gi \
  --set slave.replicaCount=3
```

或使用 YAML 檔案：

`redis-values.yaml`：
```yaml
password: MySecretPassword

master:
  persistence:
    enabled: true
    size: 20Gi

slave:
  replicaCount: 3
  persistence:
    enabled: true
    size: 20Gi
```

安裝：
```bash
helm install stable/redis --name my-redis -f redis-values.yaml
```

### 升級 Release

修改參數：
```bash
helm upgrade my-redis stable/redis --set slave.replicaCount=5
```

查看歷史：
```bash
helm history my-redis

# REVISION  STATUS      DESCRIPTION
# 1         SUPERSEDED  Install complete
# 2         DEPLOYED    Upgrade complete
```

### 回滾

出問題了，回滾到上一版：
```bash
helm rollback my-redis 1

# Rollback was a success! Happy Helming!
```

### 刪除 Release

```bash
helm delete my-redis

# release "my-redis" deleted
```

完全刪除（包括歷史）：
```bash
helm delete --purge my-redis
```

## 建立自己的 Chart

### 建立 Chart 結構

```bash
helm create myapp

# Creating myapp
```

產生結構：
```
myapp/
├── Chart.yaml           # Chart 的 metadata
├── values.yaml          # 預設值
├── charts/              # 依賴的 Charts
├── templates/           # Kubernetes YAML 模板
│   ├── NOTES.txt        # 部署後的說明
│   ├── _helpers.tpl     # 輔助函式
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   └── tests/
└── .helmignore
```

### Chart.yaml

`Chart.yaml`：
```yaml
apiVersion: v1
name: myapp
version: 1.0.0
appVersion: 1.0.0
description: My Spring Boot application
keywords:
  - spring-boot
  - java
home: https://github.com/mycompany/myapp
maintainers:
  - name: PoYu Chen
    email: poyu@example.com
```

### values.yaml

定義預設值：

`values.yaml`：
```yaml
# 副本數
replicaCount: 2

# 映像
image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ingress
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - /
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

# 資源
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi

# HPA
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# 環境變數
env:
  - name: SPRING_PROFILES_ACTIVE
    value: production
  - name: JAVA_OPTS
    value: "-Xms256m -Xmx512m"
```

### templates/deployment.yaml

使用 Go template 語法：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        env:
        {{- toYaml .Values.env | nindent 8 }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: http
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 5
```

### templates/_helpers.tpl

定義可重複使用的模板：

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 驗證 Chart

```bash
# 檢查語法
helm lint myapp

# ==> Linting myapp
# [INFO] Chart.yaml: icon is recommended
# 
# 1 chart(s) linted, no failures
```

### 測試渲染

看看實際產生的 YAML：

```bash
helm template myapp

# 或指定 release name
helm template myapp --name my-release

# 或只看特定檔案
helm template myapp --show-only templates/deployment.yaml
```

### 部署 Chart

```bash
helm install myapp --name myapp-prod

# 或使用自訂 values
helm install myapp --name myapp-prod -f prod-values.yaml
```

## 多環境管理

針對不同環境使用不同的 values 檔案。

### 開發環境

`values-dev.yaml`：
```yaml
replicaCount: 1

image:
  tag: "dev"

ingress:
  enabled: false

autoscaling:
  enabled: false

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  - name: SPRING_PROFILES_ACTIVE
    value: development
  - name: JAVA_OPTS
    value: "-Xms128m -Xmx256m"
```

部署：
```bash
helm install myapp --name myapp-dev -f values-dev.yaml
```

### Staging 環境

`values-staging.yaml`：
```yaml
replicaCount: 2

image:
  tag: "staging"

ingress:
  enabled: true
  hosts:
    - host: myapp-staging.example.com
      paths:
        - /

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5

env:
  - name: SPRING_PROFILES_ACTIVE
    value: staging
```

### Production 環境

`values-prod.yaml`：
```yaml
replicaCount: 3

image:
  tag: "1.0.0"

ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
      paths:
        - /

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

env:
  - name: SPRING_PROFILES_ACTIVE
    value: production
  - name: JAVA_OPTS
    value: "-Xms512m -Xmx1024m"
```

## Chart Dependencies

應用可能依賴其他 Charts（例如 Redis、MySQL）。

### 定義依賴

`requirements.yaml`：
```yaml
dependencies:
  - name: redis
    version: 7.1.0
    repository: https://kubernetes-charts.storage.googleapis.com/
    condition: redis.enabled
  
  - name: mysql
    version: 1.6.2
    repository: https://kubernetes-charts.storage.googleapis.com/
    condition: mysql.enabled
```

### 下載依賴

```bash
helm dependency update myapp

# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "stable" chart repository
# Update Complete.
# Saving 2 charts
# Downloading redis from repo https://kubernetes-charts.storage.googleapis.com/
# Downloading mysql from repo https://kubernetes-charts.storage.googleapis.com/
```

會產生 `charts/` 目錄，包含依賴的 Charts。

### 設定依賴

在 `values.yaml` 中設定：

```yaml
redis:
  enabled: true
  password: RedisPassword123
  master:
    persistence:
      size: 10Gi
  slave:
    replicaCount: 2

mysql:
  enabled: true
  mysqlRootPassword: RootPassword123
  mysqlDatabase: myapp
  mysqlUser: myapp
  mysqlPassword: MyAppPassword123
  persistence:
    size: 20Gi
```

部署時 Helm 會自動部署依賴的 Redis 和 MySQL。

## 打包和分享 Chart

### 打包 Chart

```bash
helm package myapp

# Successfully packaged chart and saved it to: /path/to/myapp-1.0.0.tgz
```

### 建立 Chart Repository

使用 GitHub Pages 或任何 Web 伺服器。

```bash
# 產生 index.yaml
helm repo index . --url https://charts.example.com

# 上傳 .tgz 和 index.yaml 到伺服器
```

### 使用自訂 Repository

```bash
# 新增 Repository
helm repo add mycompany https://charts.example.com

# 搜尋
helm search mycompany

# 安裝
helm install mycompany/myapp --name myapp
```

## 整合到 CI/CD

### GitLab CI

`.gitlab-ci.yml`：
```yaml
stages:
  - build
  - package
  - deploy

variables:
  CHART_NAME: myapp
  HELM_VERSION: "2.13.0"

build:
  stage: build
  image: maven:3.6-jdk-8
  script:
    - mvn clean package -DskipTests
  artifacts:
    paths:
      - target/*.jar

package_chart:
  stage: package
  image: alpine/helm:$HELM_VERSION
  script:
    - helm lint helm/$CHART_NAME
    - helm package helm/$CHART_NAME --version $CI_COMMIT_TAG
  artifacts:
    paths:
      - "*.tgz"
  only:
    - tags

deploy_staging:
  stage: deploy
  image: alpine/helm:$HELM_VERSION
  before_script:
    - kubectl config use-context staging
  script:
    - helm upgrade --install $CHART_NAME-staging helm/$CHART_NAME 
        -f helm/$CHART_NAME/values-staging.yaml
        --set image.tag=$CI_COMMIT_SHORT_SHA
  environment:
    name: staging
  only:
    - develop

deploy_production:
  stage: deploy
  image: alpine/helm:$HELM_VERSION
  before_script:
    - kubectl config use-context production
  script:
    - helm upgrade --install $CHART_NAME helm/$CHART_NAME
        -f helm/$CHART_NAME/values-prod.yaml
        --set image.tag=$CI_COMMIT_TAG
  environment:
    name: production
  when: manual
  only:
    - tags
```

## 遇到的問題

### 問題一：Tiller 權限不足

部署時出現 RBAC 錯誤。

解決：如前面所述，建立 ServiceAccount 並授予權限。

### 問題二：Chart 依賴版本衝突

應用依賴 Redis 7.x，但其他應用依賴 Redis 6.x。

解決：
1. 使用不同的 release name
2. 各自部署獨立的 Redis

### 問題三：升級失敗無法回滾

升級時改了 StatefulSet，導致無法更新。

解決：
```bash
# 強制刪除並重新安裝
helm delete --purge myapp
helm install myapp --name myapp -f values.yaml

# 或使用 --force
helm upgrade --force myapp ./myapp
```

### 問題四：values 太多，難以管理

各種環境的 values 檔案一大堆。

解決：使用 helmfile 或 kustomize 管理。

## 心得

Helm 徹底改變了我們部署 Kubernetes 應用的方式。以前要寫一堆 YAML，還要針對不同環境手動修改參數，容易出錯。現在用 Helm：

```bash
# 開發環境
helm install myapp --name myapp-dev -f values-dev.yaml

# 正式環境
helm install myapp --name myapp-prod -f values-prod.yaml
```

簡單清楚！

而且升級超方便：
```bash
helm upgrade myapp-prod myapp --set image.tag=1.1.0
```

出問題立刻回滾：
```bash
helm rollback myapp-prod
```

不用再手動 `kubectl apply` 一堆檔案了。

最棒的是，現在我們的 Chart 可以分享給其他團隊。其他人要部署類似的 Spring Boot 應用，直接用我們的 Chart，改幾個參數就好，不用從頭寫 YAML。

下週要研究 Service Mesh（Istio），管理微服務間的通訊。
