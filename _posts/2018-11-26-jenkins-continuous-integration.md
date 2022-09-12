---
layout: post
title: "Jenkins 持續整合環境建置"
date: 2018-11-26 10:15:00 +0800
categories: [DevOps, CI/CD]
tags: [Jenkins, CI, Automation, Build]
---

前兩週學習了 DevOps 的基本概念和 Git 分支策略，但每次要部署還是得手動打包、上傳、重啟服務。這週來解決這個問題，建立自動化的持續整合環境。

> 使用版本：Jenkins 2.121.x（2018 年的 LTS 版本）+ Java 8

## 為什麼需要持續整合

先說說我們團隊現在的建置流程：

1. 小陳改完程式碼，commit 到 Git
2. 我 pull 最新的程式碼
3. 本機執行 `mvn clean package`
4. 結果編譯失敗，因為小陳的程式碼有問題
5. 找小陳溝通，他說：「奇怪，我這邊可以跑啊！」
6. 檢查後發現他用 Java 9，我用 Java 8
7. 修正問題後重新打包
8. 結果單元測試失敗...

整個過程花了一小時，才發現是環境不一致的問題。

## 持續整合的核心理念

**Continuous Integration (CI)** 的精神是：

1. **頻繁整合**：至少每天整合一次
2. **自動化建置**：每次 commit 都自動觸發建置
3. **自動化測試**：建置完自動跑測試
4. **快速反饋**：建置失敗立刻通知

重點是「盡早發現問題，盡快修復」。

## 安裝 Jenkins

### 在 macOS 上安裝

```bash
# 使用 Homebrew
brew install jenkins-lts

# 啟動 Jenkins
brew services start jenkins-lts

# 查看狀態
brew services list
```

Jenkins 會在 `http://localhost:8080` 啟動。

### 初次設定

1. **解鎖 Jenkins**

訪問 `http://localhost:8080`，需要輸入初始密碼。

```bash
# 查看初始密碼
cat ~/.jenkins/secrets/initialAdminPassword
```

2. **安裝插件**

選「Install suggested plugins」，會自動安裝常用插件：
- Git plugin
- Maven Integration
- Pipeline
- Email Extension
- GitHub plugin

安裝過程大概 5-10 分鐘，可以去泡杯咖啡。

3. **建立管理員帳號**

設定使用者名稱、密碼、信箱。我設定為：
- Username: `admin`
- Password: `****`（自己記得就好）
- Email: `dev@mycompany.com`

## 設定 JDK 和 Maven

Jenkins 需要知道 JDK 和 Maven 的位置。

1. 進入「Manage Jenkins」→「Global Tool Configuration」

2. **設定 JDK**
   - 點「Add JDK」
   - Name: `JDK8`
   - JAVA_HOME: `/Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home`
   - 取消勾選「Install automatically」（我已經裝好了）

查看 Java 路徑：
```bash
/usr/libexec/java_home -v 1.8
```

3. **設定 Maven**
   - 點「Add Maven」
   - Name: `Maven3`
   - MAVEN_HOME: `/usr/local/Cellar/maven/3.5.4/libexec`

查看 Maven 路徑：
```bash
mvn -version
# Apache Maven 3.5.4
# Maven home: /usr/local/Cellar/maven/3.5.4/libexec
```

## 建立第一個 Jenkins Job

我用公司的 Spring Boot 專案來測試。

### 1. 新增 Job

- 點「New Item」
- 輸入名稱：`my-shop-api`
- 選擇「Freestyle project」
- 點「OK」

### 2. 設定原始碼管理

**Source Code Management** 區塊：
- 選擇「Git」
- Repository URL: `https://github.com/mycompany/my-shop-api.git`
- Credentials: 點「Add」新增 GitHub 帳號密碼

**Branches to build**：
- Branch Specifier: `*/develop`

### 3. 設定建置觸發條件

**Build Triggers** 區塊：
- 勾選「Poll SCM」
- Schedule: `H/5 * * * *`（每 5 分鐘檢查一次）

這個 cron 語法：
- `H/5`：每 5 分鐘（H 是 Hash，避免所有 Job 同時執行）
- `* * * *`：每小時、每天、每月、每週

### 4. 設定建置步驟

**Build** 區塊：
- 點「Add build step」
- 選擇「Invoke top-level Maven targets」
- Maven Version: `Maven3`
- Goals: `clean package -DskipTests`

先跳過測試，確保基本建置能跑通。

### 5. 設定建置後動作

**Post-build Actions**：
- 點「Add post-build action」
- 選擇「Archive the artifacts」
- Files to archive: `target/*.jar`

這樣建置完的 JAR 檔會被保存起來。

### 6. 儲存並執行

點「Save」，然後點「Build Now」。

第一次建置大概等了 3 分鐘（下載 Maven 依賴）：

```
Started by user admin
Checking out Revision 3f2d8b9...
[INFO] Building jar: /var/jenkins/workspace/my-shop-api/target/my-shop-api-1.0.0.jar
[INFO] BUILD SUCCESS
Finished: SUCCESS
```

成功！

## 加入單元測試

現在把測試也加進來。

修改 Maven Goals：

```
clean test package
```

重新建置，這次看到：

```
[INFO] Tests run: 47, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

再加入測試報告：

**Post-build Actions** 新增：
- 「Publish JUnit test result report」
- Test report XMLs: `target/surefire-reports/*.xml`

重新建置後，在 Job 頁面可以看到測試趨勢圖，很方便！

## 建置失敗通知

建置失敗時要通知團隊。

### 方式一：Email 通知

1. 安裝「Email Extension Plugin」（應該已經裝了）

2. 設定 SMTP 伺服器（「Manage Jenkins」→「Configure System」）：
   - SMTP server: `smtp.gmail.com`
   - Use SMTP Authentication: 勾選
   - User Name: `dev@mycompany.com`
   - Password: `****`（應用程式專用密碼）
   - Use SSL: 勾選
   - SMTP Port: `465`

3. Job 設定中加入「Editable Email Notification」：
   - Project Recipient List: `team@mycompany.com`
   - Triggers: 選「Failure - Any」

### 方式二：Slack 通知（更推薦）

1. 安裝「Slack Notification Plugin」

2. 在 Slack 中新增 Jenkins integration，取得 Webhook URL

3. Jenkins 全域設定：
   - Workspace: `mycompany`
   - Credential: 新增 Secret text，貼上 Token

4. Job 設定中加入「Slack Notifications」：
   - 勾選「Notify Build Start」
   - 勾選「Notify Failure」

建置失敗時，Slack 會立刻收到通知。

## 遇到的問題

### 問題一：Maven 下載依賴很慢

第一次建置時，Maven 要下載一堆依賴，超級慢。

解決方法：設定 Maven mirror（台灣的 Maven repository）

在 Jenkins 全域設定的 Maven 區塊，新增 `settings.xml`：

```xml
<mirrors>
    <mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>central</mirrorOf>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
</mirrors>
```

### 問題二：權限不足

Jenkins 執行時遇到「Permission denied」。

原因是 Jenkins 使用專屬的使用者執行，可能沒有檔案存取權限。

解決方法：
```bash
sudo chown -R jenkins:jenkins /var/jenkins/workspace
```

### 問題三：記憶體不足

建置大型專案時 OOM（Out of Memory）。

解決方法：增加 Jenkins 的 JVM 記憶體。

編輯 `/usr/local/opt/jenkins-lts/homebrew.mxcl.jenkins-lts.plist`：

```xml
<string>-Xmx2048m</string>
```

重啟 Jenkins：
```bash
brew services restart jenkins-lts
```

## 目前的 CI 流程

現在我們的流程變成：

1. 開發者 commit 並 push 到 `develop` 分支
2. Jenkins 每 5 分鐘檢查一次 Git
3. 發現新的 commit 後自動：
   - 拉取最新程式碼
   - 執行 Maven 建置
   - 執行單元測試
   - 打包成 JAR
4. 建置成功：Slack 通知綠色訊息
5. 建置失敗：Slack 立刻通知紅色訊息

從 commit 到知道建置結果，只需要 5-10 分鐘，比以前手動建置快多了！

## 心得

建立 CI 環境後，最大的好處是**盡早發現問題**。以前可能要到部署時才發現程式碼有問題，現在 commit 後幾分鐘就知道了。

下週打算深入研究 Jenkins Pipeline，把整個建置流程寫成程式碼（Pipeline as Code），這樣更好維護和版本控制。

另外也想研究 Docker，這樣可以確保建置環境的一致性，不會再有「在我機器上可以跑」的問題了。
