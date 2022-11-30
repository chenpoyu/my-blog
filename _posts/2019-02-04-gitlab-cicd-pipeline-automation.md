---
layout: post
title: "GitLab CI/CD Pipeline 自動化部署"
date: 2019-02-04 10:30:00 +0800
categories: [DevOps, CI/CD]
tags: [GitLab, CI/CD, Pipeline, Automation, Docker]
---

前面幾週建立了完整的 Kubernetes 環境、監控系統和日誌收集（參考 [ELK Stack 集中式日誌管理](/posts/2019/01/28/elk-stack-centralized-logging/)）。但部署流程還是很手動：

1. 本機測試
2. Commit & Push
3. 手動觸發 Jenkins build
4. Build 成功後，手動執行部署腳本
5. 在 Slack 通知大家「已部署到測試環境」

這週把整個流程自動化，使用 GitLab CI/CD Pipeline。

> 我們公司剛好在評估從 GitHub 遷移到 GitLab（自架），所以順便研究 GitLab CI/CD

## 為什麼選擇 GitLab CI/CD

我們之前用 Jenkins，但有些痛點：

### Jenkins 的問題

1. **設定複雜**：要裝一堆 plugin，設定很繁瑣
2. **維護成本高**：Jenkins 本身也要維護和更新
3. **Pipeline 和程式碼分離**：Jenkinsfile 寫錯很難 review

### GitLab CI/CD 的優勢

1. **內建**：安裝 GitLab 就有，不用額外維護
2. **設定簡單**：一個 `.gitlab-ci.yml` 搞定
3. **與 Git 整合**：Push 就自動觸發，支援 Merge Request Pipeline
4. **視覺化好**：Pipeline 圖很清楚
5. **免費**：即使是 Community Edition 也有完整的 CI/CD 功能

## GitLab CI/CD 基本概念

### Pipeline
一次 CI/CD 執行的完整流程。

### Stage
Pipeline 的階段，例如：build、test、deploy。同一個 stage 的 job 會並行執行。

### Job
實際執行的任務，例如：編譯程式碼、跑測試、部署。

### Runner
執行 job 的機器。可以是實體機器、虛擬機或容器。

### 流程示意

```
Pipeline
├── Stage: Build
│   └── Job: compile
├── Stage: Test
│   ├── Job: unit-test (並行)
│   └── Job: integration-test (並行)
└── Stage: Deploy
    └── Job: deploy-to-staging
```

## 安裝 GitLab Runner

GitLab 本身不執行 job，需要 Runner。

### 在 Kubernetes 上安裝 Runner

使用 Helm 安裝：

```bash
# 加入 GitLab Helm repo
helm repo add gitlab https://charts.gitlab.io

# 安裝 GitLab Runner
helm install gitlab/gitlab-runner \
  --name gitlab-runner \
  --namespace gitlab \
  --set gitlabUrl=https://gitlab.mycompany.com \
  --set runnerRegistrationToken="YOUR_REGISTRATION_TOKEN" \
  --set rbac.create=true
```

Registration Token 在 GitLab 的「Admin Area」→「Runners」可以找到。

查看 Runner：
```bash
kubectl get pods -n gitlab
# NAME                             READY   STATUS    RESTARTS   AGE
# gitlab-runner-xxxxx              1/1     Running   0          1m
```

在 GitLab UI 上應該會看到 Runner 變成 active（綠色圓點）。

## 第一個 Pipeline

在專案根目錄建立 `.gitlab-ci.yml`：

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building the application..."
    - mvn clean compile

test-job:
  stage: test
  script:
    - echo "Running tests..."
    - mvn test

deploy-job:
  stage: deploy
  script:
    - echo "Deploying application..."
    - echo "Application deployed!"
```

Commit 並 push：
```bash
git add .gitlab-ci.yml
git commit -m "Add GitLab CI/CD pipeline"
git push
```

GitLab 會自動偵測到 `.gitlab-ci.yml`，觸發 Pipeline。

### 查看 Pipeline

在 GitLab UI：
1. 專案頁面 → 左側「CI/CD」→「Pipelines」
2. 點擊最新的 Pipeline

可以看到：
- 🟢 build-job: passed
- 🟢 test-job: passed
- 🟢 deploy-job: passed

點擊每個 job 可以看到執行日誌。

## 實際專案的 Pipeline

這是我們 Spring Boot 專案的完整 Pipeline：

`.gitlab-ci.yml`：
```yaml
# 定義 Docker image
image: maven:3.5.4-jdk-8

# 定義變數
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  DOCKER_REGISTRY: "registry.mycompany.com"
  IMAGE_NAME: "myshop/my-shop-api"
  KUBECONFIG: /root/.kube/config

# 快取 Maven 依賴
cache:
  paths:
    - .m2/repository

# 定義 stages
stages:
  - build
  - test
  - package
  - deploy-dev
  - deploy-staging
  - deploy-prod

# Build stage
compile:
  stage: build
  script:
    - mvn clean compile
  artifacts:
    paths:
      - target/
    expire_in: 1 hour

# Test stage
unit-test:
  stage: test
  script:
    - mvn test
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

code-quality:
  stage: test
  script:
    - mvn checkstyle:check
  allow_failure: true

# Package stage
build-jar:
  stage: package
  script:
    - mvn package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week

build-docker:
  stage: package
  image: docker:19.03
  services:
    - docker:19.03-dind
  before_script:
    - docker login -u $DOCKER_USER -p $DOCKER_PASSWORD $DOCKER_REGISTRY
  script:
    - docker build -t $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker build -t $DOCKER_REGISTRY/$IMAGE_NAME:latest .
    - docker push $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_REGISTRY/$IMAGE_NAME:latest

# Deploy to Dev
deploy-dev:
  stage: deploy-dev
  image: alpine/k8s:1.13.0
  script:
    - echo $KUBECONFIG_CONTENT | base64 -d > $KUBECONFIG
    - kubectl set image deployment/my-shop-api my-shop-api=$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA -n dev
    - kubectl rollout status deployment/my-shop-api -n dev
  environment:
    name: development
    url: https://dev-api.mycompany.com
  only:
    - develop

# Deploy to Staging
deploy-staging:
  stage: deploy-staging
  image: alpine/k8s:1.13.0
  script:
    - echo $KUBECONFIG_CONTENT | base64 -d > $KUBECONFIG
    - kubectl set image deployment/my-shop-api my-shop-api=$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA -n staging
    - kubectl rollout status deployment/my-shop-api -n staging
  environment:
    name: staging
    url: https://staging-api.mycompany.com
  only:
    - master
  when: manual

# Deploy to Production
deploy-prod:
  stage: deploy-prod
  image: alpine/k8s:1.13.0
  script:
    - echo $KUBECONFIG_CONTENT | base64 -d > $KUBECONFIG
    - kubectl set image deployment/my-shop-api my-shop-api=$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA -n prod
    - kubectl rollout status deployment/my-shop-api -n prod
  environment:
    name: production
    url: https://api.mycompany.com
  only:
    - tags
  when: manual
```

### 重點說明

#### 變數

使用 CI/CD 變數：
- `$CI_COMMIT_SHORT_SHA`：Git commit 的短 hash
- `$DOCKER_USER`、`$DOCKER_PASSWORD`：從 GitLab 變數讀取
- 自訂變數：`DOCKER_REGISTRY`、`IMAGE_NAME`

#### Cache

快取 Maven 依賴，加速 build：
```yaml
cache:
  paths:
    - .m2/repository
```

#### Artifacts

保存 build 產物，供後續 stage 使用：
```yaml
artifacts:
  paths:
    - target/*.jar
```

#### Services

使用 Docker-in-Docker 建置映像檔：
```yaml
services:
  - docker:19.03-dind
```

#### Environment

定義環境和 URL：
```yaml
environment:
  name: production
  url: https://api.mycompany.com
```

在 GitLab UI 可以看到部署歷史。

#### Only / Except

控制 job 在哪些分支或標籤執行：
```yaml
only:
  - master
  - develop
```

```yaml
except:
  - tags
```

#### When

控制 job 執行時機：
- `on_success`：前面的 job 成功才執行（預設）
- `on_failure`：前面的 job 失敗才執行
- `always`：總是執行
- `manual`：手動觸發

## 設定 GitLab 變數

敏感資訊（密碼、token）不要寫在 `.gitlab-ci.yml`，使用 GitLab CI/CD Variables。

1. 專案 →「Settings」→「CI/CD」→「Variables」
2. 點「Add Variable」

新增：
- `DOCKER_USER`: `developer`
- `DOCKER_PASSWORD`: `dev123456`（勾選 Masked）
- `KUBECONFIG_CONTENT`: `<kubeconfig 內容的 base64>`（勾選 Masked）

Masked 的變數不會在日誌中顯示。

產生 `KUBECONFIG_CONTENT`：
```bash
cat ~/.kube/config | base64
```

複製輸出，貼到 GitLab 變數。

## Merge Request Pipeline

GitLab 支援在 Merge Request 時跑 Pipeline，確保 code review 前先過測試。

### 啟用 MR Pipeline

`.gitlab-ci.yml` 已經支援，不需要額外設定。

當建立 Merge Request 時，GitLab 會自動執行 Pipeline。

### 設定 Merge 前必須過 Pipeline

1. 專案 →「Settings」→「General」→「Merge requests」
2. 勾選「Pipelines must succeed」
3. 儲存

現在 Pipeline 失敗的 MR 無法合併。

## Pipeline 進階功能

### 並行執行

同一個 stage 的 job 會並行：

```yaml
test:unit:
  stage: test
  script:
    - mvn test

test:integration:
  stage: test
  script:
    - mvn verify -Pintegration

test:e2e:
  stage: test
  script:
    - npm run test:e2e
```

三個測試會同時執行，節省時間。

### 矩陣建置

測試多個版本：

```yaml
test:
  stage: test
  image: openjdk:$JAVA_VERSION
  script:
    - mvn test
  parallel:
    matrix:
      - JAVA_VERSION: ["8", "11"]
```

會執行兩個 job：
- test: [8]
- test: [11]

### 條件執行

只在特定條件下執行：

```yaml
deploy:
  stage: deploy
  script:
    - ./deploy.sh
  rules:
    - if: '$CI_COMMIT_BRANCH == "master"'
      when: on_success
    - if: '$CI_COMMIT_TAG'
      when: manual
    - when: never
```

### 依賴關係

跨 stage 的 job 依賴：

```yaml
deploy-frontend:
  stage: deploy
  needs: ["build-frontend"]
  script:
    - ./deploy-frontend.sh

deploy-backend:
  stage: deploy
  needs: ["build-backend"]
  script:
    - ./deploy-backend.sh
```

不需要等整個 stage 完成，有依賴的完成就可以開始。

### Include

重複使用 CI 設定：

`.gitlab-ci-template.yml`：
```yaml
.build_template:
  image: maven:3.5.4-jdk-8
  before_script:
    - echo "Setting up..."
  cache:
    paths:
      - .m2/repository
```

`.gitlab-ci.yml`：
```yaml
include:
  - local: '.gitlab-ci-template.yml'

build:
  extends: .build_template
  script:
    - mvn compile
```

## 通知整合

### Slack 通知

安裝 Slack 整合：
1. 專案 →「Settings」→「Integrations」
2. 選「Slack notifications」
3. 輸入 Webhook URL
4. 勾選要通知的事件（Pipeline、Merge request）

### 自訂通知

在 `.gitlab-ci.yml` 加入：

```yaml
notify-slack:
  stage: .post  # 特殊 stage，在最後執行
  script:
    - |
      curl -X POST $SLACK_WEBHOOK_URL \
        -H 'Content-Type: application/json' \
        -d "{
          \"text\": \"Pipeline $CI_PIPELINE_STATUS: $CI_PROJECT_NAME\",
          \"attachments\": [{
            \"color\": \"$([[ \"$CI_PIPELINE_STATUS\" == \"success\" ]] && echo good || echo danger)\",
            \"fields\": [
              {\"title\": \"Branch\", \"value\": \"$CI_COMMIT_REF_NAME\", \"short\": true},
              {\"title\": \"Commit\", \"value\": \"$CI_COMMIT_SHORT_SHA\", \"short\": true},
              {\"title\": \"Author\", \"value\": \"$GITLAB_USER_NAME\", \"short\": true}
            ],
            \"actions\": [{
              \"type\": \"button\",
              \"text\": \"View Pipeline\",
              \"url\": \"$CI_PIPELINE_URL\"
            }]
          }]
        }"
  when: always
```

## 遇到的問題

### 問題一：Runner 無法連接 Docker daemon

錯誤：`Cannot connect to the Docker daemon`

原因：使用 Docker-in-Docker 需要 privileged mode。

解決方法：編輯 Runner 設定
```bash
kubectl edit configmap gitlab-runner -n gitlab
```

加入：
```yaml
[[runners]]
  [runners.kubernetes]
    privileged = true
```

重啟 Runner：
```bash
kubectl delete pod -n gitlab -l app=gitlab-runner
```

### 問題二：Pipeline 很慢

每次都要下載所有 Maven 依賴，很花時間。

解決方法：使用 cache
```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .m2/repository
```

### 問題三：Kubernetes 部署失敗

錯誤：`error: You must be logged in to the server (Unauthorized)`

原因：KUBECONFIG 不正確或過期。

解決方法：
1. 產生新的 ServiceAccount token
2. 更新 GitLab 變數 `KUBECONFIG_CONTENT`

建立 ServiceAccount：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-deployer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-deployer
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: ServiceAccount
  name: gitlab-deployer
  namespace: default
```

## 實際效果

現在的流程：

1. 開發者在 feature 分支開發
2. Commit & Push → 自動跑測試
3. 建立 Merge Request → 自動跑完整 Pipeline
4. Code review
5. 合併到 develop → 自動部署到 Dev 環境
6. 測試通過後，合併到 master → 可以手動部署到 Staging
7. 確認沒問題，打 tag → 可以手動部署到 Production

完全自動化！而且 Pipeline 失敗會立刻通知 Slack，不會有「不知道部署失敗」的情況。

## 心得

GitLab CI/CD 真的比 Jenkins 簡單很多。以前設定 Jenkins Pipeline 要研究一堆 plugin，寫 Groovy 語法。現在只要一個 YAML 檔，而且語法很直覺。

最棒的是整合度很好，從 commit、MR、到部署，全部在 GitLab 上就能看到。不用在 GitHub 和 Jenkins 之間切來切去。

下週要研究容器安全掃描，整合到 CI/CD Pipeline 中，確保不會部署有漏洞的映像檔。
