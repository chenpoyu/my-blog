---
layout: post
title: "持續整合與部署：Fastlane 自動化 iOS 工作流"
date: 2018-09-21
categories: [iOS, DevOps]
tags: [Fastlane, CI/CD, Automation, Jenkins]
---

這週研究 iOS app 的持續整合和部署（CI/CD），重點是 Fastlane 工具。手動建置、測試、上架很耗時且容易出錯，自動化能大幅提升效率和品質。

## 為什麼需要 CI/CD

**手動流程的問題**：

1. 建置 app
2. 執行測試
3. 截圖不同語言版本
4. 上傳到 TestFlight
5. 提交到 App Store
6. 填寫發布說明

每次發布要 1-2 小時，而且容易漏步驟。

**自動化的好處**：
- 節省時間
- 減少人為錯誤
- 確保一致性
- 提早發現問題

## Fastlane 簡介

Fastlane 是 iOS/Android 自動化工具，整合了建置、測試、部署流程。

**安裝**：

```bash
# 用 Homebrew
brew install fastlane

# 或用 RubyGems
sudo gem install fastlane

# 初始化專案
cd /path/to/your/project
fastlane init
```

Fastlane 會問幾個問題，然後產生 `fastlane` 資料夾和配置檔。

## Fastfile：定義自動化流程

`Fastfile` 定義不同的 lane（工作流程）：

```ruby
# fastlane/Fastfile

default_platform(:ios)

platform :ios do
  # 執行測試
  lane :test do
    scan(
      scheme: "MyApp",
      devices: ["iPhone 11"],
      clean: true
    )
  end
  
  # 建置並上傳到 TestFlight
  lane :beta do
    # 確保 Git working directory 乾淨
    ensure_git_status_clean
    
    # 增加 build number
    increment_build_number(xcodeproj: "MyApp.xcodeproj")
    
    # 建置
    build_app(
      scheme: "MyApp",
      export_method: "app-store"
    )
    
    # 上傳到 TestFlight
    upload_to_testflight(
      skip_waiting_for_build_processing: true
    )
    
    # Commit version bump
    commit_version_bump(
      message: "Version bump to #{lane_context[SharedValues::BUILD_NUMBER]}"
    )
    
    # Push to git
    push_to_git_remote
  end
  
  # 發布到 App Store
  lane :release do
    # 先執行測試
    test
    
    # 建置
    build_app(scheme: "MyApp")
    
    # 上傳到 App Store
    upload_to_app_store(
      submit_for_review: true,
      automatic_release: false,
      force: true,
      metadata_path: "./fastlane/metadata"
    )
    
    # 發送 Slack 通知
    slack(
      message: "Successfully released version #{get_version_number}!",
      success: true
    )
  end
end
```

**執行 lane**：

```bash
fastlane test      # 執行測試
fastlane beta      # 上傳到 TestFlight
fastlane release   # 發布到 App Store
```

## Match：管理憑證和 Provisioning Profiles

iOS 的憑證管理很麻煩，特別是團隊協作時。Match 把憑證存在 Git repository，團隊成員共用。

**設定 Match**：

```bash
fastlane match init
```

選擇儲存方式（git / google_cloud / s3），通常用 private git repo。

**Matchfile 配置**：

```ruby
# fastlane/Matchfile

git_url("https://github.com/your-company/certificates")
storage_mode("git")

type("development")  # 或 "appstore", "adhoc"
app_identifier("com.example.myapp")
username("your_apple_id@example.com")
```

**使用 Match**：

```bash
# 第一次設定（產生憑證）
fastlane match development
fastlane match appstore

# 其他成員同步憑證
fastlane match development --readonly
```

**在 Fastfile 中使用**：

```ruby
lane :beta do
  match(type: "appstore")  # 自動取得憑證
  build_app(scheme: "MyApp")
  upload_to_testflight
end
```

## Gym：建置 App

Gym 簡化 Xcode 建置：

```ruby
gym(
  scheme: "MyApp",
  clean: true,
  export_method: "app-store",
  output_directory: "./build",
  output_name: "MyApp.ipa"
)

# 或用別名
build_app(scheme: "MyApp")
```

**常用選項**：
- `scheme`：Xcode scheme 名稱
- `configuration`："Debug" 或 "Release"
- `export_method`："app-store", "ad-hoc", "development"
- `include_bitcode`：是否包含 bitcode
- `include_symbols`：是否包含 debug symbols

## Scan：執行測試

Scan 執行單元測試和 UI 測試：

```ruby
scan(
  scheme: "MyApp",
  devices: ["iPhone 11", "iPhone SE (2nd generation)"],
  code_coverage: true,
  output_directory: "./test_results",
  fail_build: true  # 測試失敗時中止 lane
)

# 或用別名
run_tests(scheme: "MyApp")
```

**產生測試報告**：

```ruby
lane :test do
  scan(scheme: "MyApp", code_coverage: true)
  
  # 產生 HTML 報告
  slather(
    proj: "MyApp.xcodeproj",
    workspace: "MyApp.xcworkspace",
    scheme: "MyApp",
    output_directory: "./coverage",
    html: true
  )
end
```

## Snapshot：自動截圖

為不同語言和裝置自動截圖，用於 App Store 頁面：

**設定 Snapshot**：

```bash
fastlane snapshot init
```

產生 `Snapfile` 和 `SnapshotHelper.swift`。

**Snapfile 配置**：

```ruby
devices([
  "iPhone 11 Pro Max",
  "iPhone 11",
  "iPhone SE (2nd generation)",
  "iPad Pro (12.9-inch) (3rd generation)"
])

languages([
  "en-US",
  "zh-Hant"
])

scheme("MyApp")
output_directory("./screenshots")
clear_previous_screenshots(true)
```

**UI Test 中截圖**：

```swift
import XCTest

class SnapshotUITests: XCTestCase {
    override func setUp() {
        super.setUp()
        let app = XCUIApplication()
        setupSnapshot(app)  // SnapshotHelper 提供
        app.launch()
    }
    
    func testScreenshots() {
        let app = XCUIApplication()
        
        // 首頁
        snapshot("01HomeScreen")
        
        // 點選按鈕
        app.buttons["Login"].tap()
        snapshot("02LoginScreen")
        
        // 填寫表單
        app.textFields["Username"].tap()
        app.textFields["Username"].typeText("testuser")
        snapshot("03LoginForm")
    }
}
```

**執行截圖**：

```bash
fastlane snapshot
```

會自動在不同裝置和語言執行 UI 測試並截圖。

## Deliver：上傳到 App Store

Deliver 自動上傳 app 和 metadata：

**產生 metadata**：

```bash
fastlane deliver init
```

產生 `fastlane/metadata` 資料夾，包含：

```
metadata/
  en-US/
    description.txt
    keywords.txt
    marketing_url.txt
    name.txt
    release_notes.txt
    subtitle.txt
  zh-Hant/
    description.txt
    ...
  screenshots/
    en-US/
      iPhone 11 Pro Max-01HomeScreen.png
    zh-Hant/
      ...
```

**上傳**：

```ruby
lane :release do
  build_app(scheme: "MyApp")
  
  deliver(
    submit_for_review: true,
    automatic_release: true,
    submission_information: {
      add_id_info_uses_idfa: false  # 是否使用 IDFA
    }
  )
end
```

## 整合 CI 服務

### Jenkins

**Jenkinsfile**：

```groovy
pipeline {
    agent any
    
    environment {
        FASTLANE_SKIP_UPDATE_CHECK = "1"
        LANG = "en_US.UTF-8"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'bundle install'
            }
        }
        
        stage('Test') {
            steps {
                sh 'bundle exec fastlane test'
            }
        }
        
        stage('Build & Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'bundle exec fastlane beta'
            }
        }
    }
    
    post {
        success {
            slackSend(color: 'good', message: "Build Successful: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(color: 'danger', message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
    }
}
```

### GitHub Actions

**.github/workflows/ios.yml**：

```yaml
name: iOS CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: macos-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: 2.7
    
    - name: Install Fastlane
      run: |
        gem install bundler
        bundle install
    
    - name: Run tests
      run: bundle exec fastlane test
    
    - name: Build and upload to TestFlight
      if: github.ref == 'refs/heads/main'
      env:
        FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
      run: bundle exec fastlane beta
```

### CircleCI

**.circleci/config.yml**：

```yaml
version: 2.1

jobs:
  test:
    macos:
      xcode: 12.5.0
    steps:
      - checkout
      - run:
          name: Install Fastlane
          command: |
            gem install bundler
            bundle install
      - run:
          name: Run tests
          command: bundle exec fastlane test

  deploy:
    macos:
      xcode: 12.5.0
    steps:
      - checkout
      - run:
          name: Install Fastlane
          command: bundle install
      - run:
          name: Deploy to TestFlight
          command: bundle exec fastlane beta

workflows:
  version: 2
  test_and_deploy:
    jobs:
      - test
      - deploy:
          requires:
            - test
          filters:
            branches:
              only: main
```

## 最佳實踐

### 1. 使用環境變數保護敏感資訊

```ruby
# 不要直接寫在 Fastfile
# upload_to_app_store(username: "your_email@example.com", password: "secret")

# 用環境變數
upload_to_app_store(
  username: ENV["APPLE_ID"],
  password: ENV["FASTLANE_PASSWORD"]
)
```

### 2. 版本號自動化

```ruby
lane :bump_version do
  # 自動增加版本號
  increment_version_number(
    bump_type: "patch"  # "major", "minor", "patch"
  )
  
  # 或手動指定
  increment_version_number(version_number: "2.0.0")
  
  # Build number 用時間戳
  increment_build_number(build_number: Time.now.to_i.to_s)
end
```

### 3. 錯誤通知

```ruby
error do |lane, exception|
  slack(
    message: "Error in lane #{lane}: #{exception}",
    success: false
  )
end
```

### 4. 使用 Gemfile 固定版本

```ruby
# Gemfile
source "https://rubygems.org"

gem "fastlane", "~> 2.190.0"
gem "cocoapods", "~> 1.10.0"
```

```bash
bundle install
bundle exec fastlane beta  # 用固定版本
```

## 與 Java CI/CD 的對比

**Java（Spring Boot）**：
- Jenkins、GitLab CI 主流
- Maven/Gradle 建置
- Docker 容器化
- 部署到伺服器或雲端

**iOS**：
- Fastlane + CI 服務
- Xcode 建置
- 上傳到 App Store / TestFlight
- 憑證管理更複雜

iOS 的 CI/CD 受限於 Apple 生態系，需要 macOS 執行環境和 Apple Developer 帳號。

## 學習心得

Fastlane 大幅簡化 iOS 的發布流程，一開始設定需要時間，但長期來看非常值得。

**建議**：
- 小專案也該用 Fastlane，至少自動化測試和建置
- Match 解決團隊的憑證管理難題
- 搭配 CI 服務實現完全自動化
- 截圖工具對多語言 app 很有用

從 Java 後端轉到 iOS，發現行動 app 的 CI/CD 更複雜，但 Fastlane 把很多痛點都解決了。現在每次提交到 main branch 都會自動測試、建置、上傳到 TestFlight，省下大量時間。
