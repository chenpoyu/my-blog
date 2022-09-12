---
layout: post
title: "Jenkins Pipeline 程式碼化建置流程"
date: 2018-12-03 09:45:00 +0800
categories: [DevOps, CI/CD]
tags: [Jenkins, Pipeline, Groovy, Jenkinsfile]
---

上週建立了基本的 Jenkins CI 環境（參考 [Jenkins 持續整合環境建置](/posts/2018/11/26/jenkins-continuous-integration/)），但所有設定都是透過網頁介面點選完成的。這樣有個問題：如果 Jenkins 掛了重裝，所有 Job 的設定都要重新來過。

## Freestyle Job 的困擾

目前我們團隊有 5 個專案，每個專案都要：
1. 登入 Jenkins 網頁
2. 點「New Item」
3. 設定 Git repository
4. 設定建置觸發條件
5. 設定 Maven goals
6. 設定測試報告
7. 設定通知

設定一個 Job 要花 10 分鐘，而且很難追蹤變更歷史。上週同事不小心改錯設定，花了半天才找出問題。

## Pipeline as Code 的理念

Jenkins Pipeline 讓我們可以用程式碼（Groovy）定義整個建置流程，放在專案的 `Jenkinsfile` 中。

好處：
- **版本控制**：Jenkinsfile 跟程式碼一起進 Git
- **可追蹤**：知道誰改了什麼
- **可重複**：其他專案可以參考或複製
- **程式碼審查**：透過 Pull Request review

## 第一個 Jenkinsfile

在專案根目錄建立 `Jenkinsfile`：

```groovy
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }
    }
}
```

### 在 Jenkins 中建立 Pipeline Job

1. 點「New Item」
2. 輸入名稱：`my-shop-api-pipeline`
3. 選擇「Pipeline」
4. 在「Pipeline」區塊：
   - Definition: 選「Pipeline script from SCM」
   - SCM: 選「Git」
   - Repository URL: `https://github.com/mycompany/my-shop-api.git`
   - Script Path: `Jenkinsfile`（預設值）

點「Build Now」，Jenkins 會自動：
1. 從 Git 拉 code
2. 讀取 Jenkinsfile
3. 執行定義的 stages

## Pipeline 語法詳解

### agent - 指定執行環境

```groovy
// 任何可用的 agent
agent any

// 指定 label
agent {
    label 'linux'
}

// 使用 Docker
agent {
    docker {
        image 'maven:3.5.4-jdk-8'
    }
}
```

我們公司還沒有多台 Jenkins node，所以用 `agent any`。

### stages 和 steps

Pipeline 由多個 stage 組成，每個 stage 包含多個 step：

```groovy
stages {
    stage('Build') {
        steps {
            sh 'mvn clean compile'
            echo 'Build completed'
        }
    }
}
```

### environment - 環境變數

```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = 'my-shop-api'
        APP_VERSION = '1.0.0'
        MAVEN_OPTS = '-Xmx1024m'
    }
    
    stages {
        stage('Build') {
            steps {
                echo "Building ${APP_NAME} version ${APP_VERSION}"
                sh 'mvn clean package'
            }
        }
    }
}
```

### post - 建置後動作

不管建置成功或失敗，都要執行的動作：

```groovy
pipeline {
    agent any
    
    stages {
        // ...
    }
    
    post {
        always {
            // 總是執行
            junit 'target/surefire-reports/*.xml'
        }
        
        success {
            // 建置成功時
            echo 'Build succeeded!'
            archiveArtifacts 'target/*.jar'
        }
        
        failure {
            // 建置失敗時
            echo 'Build failed!'
            mail to: 'team@mycompany.com',
                 subject: "Build Failed: ${env.JOB_NAME}",
                 body: "Check ${env.BUILD_URL}"
        }
    }
}
```

## 實際專案的 Jenkinsfile

這是我們 Spring Boot 專案的完整 Jenkinsfile：

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven3'
        jdk 'JDK8'
    }
    
    environment {
        APP_NAME = 'my-shop-api'
        DEPLOY_ENV = 'staging'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Pulling code from GitHub...'
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'mvn clean compile'
            }
        }
        
        stage('Unit Test') {
            steps {
                echo 'Running unit tests...'
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging application...'
                sh 'mvn package -DskipTests'
            }
        }
        
        stage('Archive') {
            steps {
                echo 'Archiving artifacts...'
                archiveArtifacts artifacts: 'target/*.jar',
                                fingerprint: true
            }
        }
    }
    
    post {
        success {
            echo "✓ Build #${env.BUILD_NUMBER} succeeded"
            slackSend channel: '#dev-builds',
                      color: 'good',
                      message: "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        
        failure {
            echo "✗ Build #${env.BUILD_NUMBER} failed"
            slackSend channel: '#dev-builds',
                      color: 'danger',
                      message: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
        }
        
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
```

## 進階技巧

### 條件執行

只在特定分支執行某些 stage：

```groovy
stage('Deploy to Production') {
    when {
        branch 'master'
    }
    steps {
        sh './deploy-prod.sh'
    }
}
```

只在工作日執行：

```groovy
stage('Deploy') {
    when {
        expression {
            def day = new Date().day
            return day >= 1 && day <= 5  // Monday to Friday
        }
    }
    steps {
        sh './deploy.sh'
    }
}
```

### 平行執行

同時執行多個測試：

```groovy
stage('Parallel Tests') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'mvn verify -Pintegration'
            }
        }
        stage('Code Quality') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }
    }
}
```

### 輸入確認

部署到生產環境前需要人工確認：

```groovy
stage('Deploy to Production') {
    steps {
        input message: 'Deploy to production?',
              ok: 'Deploy',
              submitter: 'admin,devops'
        
        sh './deploy-production.sh'
    }
}
```

### 使用 Credentials

安全地使用密碼或 API key：

```groovy
environment {
    // 從 Jenkins Credentials 讀取
    GITHUB_TOKEN = credentials('github-token')
    AWS_KEY = credentials('aws-access-key')
}

stages {
    stage('Deploy') {
        steps {
            sh '''
                curl -H "Authorization: token ${GITHUB_TOKEN}" \
                     https://api.github.com/...
            '''
        }
    }
}
```

在 Jenkins 中設定 Credentials：
1. 「Manage Jenkins」→「Manage Credentials」
2. 點「Add Credentials」
3. Kind: 選「Secret text」
4. ID: `github-token`
5. Secret: 貼上你的 token

## 遇到的問題

### 問題一：Jenkinsfile 語法錯誤

寫錯一個大括號，整個 pipeline 就掛了。

解決方法：使用 Blue Ocean plugin 的 Pipeline Editor，有語法檢查和視覺化編輯器。

安裝 Blue Ocean：
```
「Manage Jenkins」→「Manage Plugins」→ 搜尋「Blue Ocean」
```

### 問題二：權限問題

Pipeline 執行 shell script 時遇到 Permission denied。

```groovy
stage('Deploy') {
    steps {
        sh 'chmod +x deploy.sh'
        sh './deploy.sh'
    }
}
```

### 問題三：多分支管理

不同分支要跑不同的流程。

解決方法：建立 Multibranch Pipeline Job

1. 新增「Multibranch Pipeline」
2. 設定 Git repository
3. Jenkins 會自動掃描所有分支
4. 每個有 Jenkinsfile 的分支都會建立對應的 Job

這樣 `develop`、`master`、`feature/*` 分支都會自動建立 Pipeline。

## 實際效果

現在我們的流程：

1. 開發者在 feature 分支開發
2. Commit 並 push
3. Jenkins 自動偵測並執行該分支的 Jenkinsfile
4. 建置、測試、打包
5. 結果通知到 Slack

如果是 `master` 分支，還會自動部署到 staging 環境。

最棒的是，Jenkinsfile 跟程式碼一起 review，CI/CD 流程的變更也變得透明且可追蹤了。

## Declarative vs Scripted Pipeline

Jenkins Pipeline 有兩種語法：

**Declarative Pipeline**（我用的）
- 結構化、易讀
- 官方推薦
- 適合大多數情況

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'make'
            }
        }
    }
}
```

**Scripted Pipeline**
- 更靈活
- 完整的 Groovy 語法
- 適合複雜場景

```groovy
node {
    stage('Build') {
        sh 'make'
    }
}
```

除非有特殊需求，建議用 Declarative Pipeline。

## 心得

把建置流程寫成程式碼後，整個 CI/CD 變得更可靠了。以前要改設定都很緊張，怕改錯東西。現在改 Jenkinsfile 可以先在 feature 分支測試，確認沒問題才合併到 master。

下週打算研究 Docker，把應用程式容器化。這樣可以確保開發、測試、生產環境的一致性，也更容易擴展和部署。
