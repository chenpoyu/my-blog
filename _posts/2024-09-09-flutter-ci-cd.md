---
layout: post
title: "Flutter CI/CD：自動化測試與部署流程"
date: 2024-09-09 13:40:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, CI/CD, GitHub Actions, Fastlane]
---

測試和效能都搞定後，這週來建立 CI/CD 流程。每次要手動執行測試、打包、上傳很煩，而且容易出錯。自動化之後，push code 就自動跑測試、建置、部署。

我們用 **GitHub Actions** 做 CI，**Fastlane** 處理 iOS/Android 的簽章和上傳。

## GitHub Actions 基礎

GitHub Actions 是 GitHub 內建的 CI/CD 工具，設定檔放在 `.github/workflows/`。

### 基本結構

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
          channel: 'stable'
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Run tests
        run: flutter test
      
      - name: Check code format
        run: flutter format --set-exit-if-changed .
      
      - name: Analyze code
        run: flutter analyze
```

這個 workflow 會在 push 或 PR 時執行測試、檢查格式、分析程式碼。

## 完整的 CI Pipeline

### 多平台測試

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Test on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
          channel: 'stable'
          cache: true
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Run code generation
        run: flutter pub run build_runner build --delete-conflicting-outputs
      
      - name: Run tests
        run: flutter test --coverage
      
      - name: Upload coverage to Codecov
        if: matrix.os == 'ubuntu-latest'
        uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info
  
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
          channel: 'stable'
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Check format
        run: flutter format --set-exit-if-changed .
      
      - name: Analyze
        run: flutter analyze --fatal-infos
      
      - name: Check for outdated dependencies
        run: flutter pub outdated
```

### Integration Test

```yaml
# .github/workflows/integration_test.yml
name: Integration Test

on:
  push:
    branches: [ main ]

jobs:
  integration-test:
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Run integration tests (iOS)
        run: |
          flutter drive \
            --driver=test_driver/integration_test.dart \
            --target=integration_test/app_test.dart \
            -d iPhone
      
      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: integration-test-screenshots
          path: screenshots/
```

## Android 建置與部署

### Fastlane 設定

安裝 Fastlane：

```bash
# macOS
brew install fastlane

# 初始化
cd android
fastlane init
```

設定 Fastlane：

```ruby
# android/fastlane/Fastfile
default_platform(:android)

platform :android do
  desc "Build and upload to Google Play Internal Testing"
  lane :internal do
    # 取得版本號
    version_code = flutter_version()["version_code"]
    version_name = flutter_version()["version_name"]
    
    # 建置 AAB
    sh("cd ../.. && flutter build appbundle --release")
    
    # 上傳到 Google Play
    upload_to_play_store(
      track: 'internal',
      aab: '../build/app/outputs/bundle/release/app-release.aab',
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true
    )
    
    # 發送通知
    slack(
      message: "Android v#{version_name} (#{version_code}) uploaded to Internal Testing",
      success: true
    )
  end
  
  desc "Build and upload to Beta"
  lane :beta do
    sh("cd ../.. && flutter build appbundle --release")
    
    upload_to_play_store(
      track: 'beta',
      aab: '../build/app/outputs/bundle/release/app-release.aab'
    )
  end
  
  desc "Promote from Beta to Production"
  lane :production do
    upload_to_play_store(
      track: 'beta',
      track_promote_to: 'production',
      skip_upload_apk: true,
      skip_upload_aab: true
    )
  end
end
```

### GitHub Actions 自動部署

```yaml
# .github/workflows/deploy_android.yml
name: Deploy Android

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true
      
      - name: Decode keystore
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
        run: |
          echo $KEYSTORE_BASE64 | base64 --decode > android/app/keystore.jks
      
      - name: Create key.properties
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: |
          cat > android/key.properties << EOF
          storePassword=$KEYSTORE_PASSWORD
          keyPassword=$KEY_PASSWORD
          keyAlias=$KEY_ALIAS
          storeFile=keystore.jks
          EOF
      
      - name: Setup service account
        env:
          GOOGLE_PLAY_SERVICE_ACCOUNT: ${{ secrets.GOOGLE_PLAY_SERVICE_ACCOUNT }}
        run: |
          echo $GOOGLE_PLAY_SERVICE_ACCOUNT > android/service-account.json
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Build and deploy
        run: |
          cd android
          bundle install
          bundle exec fastlane internal
```

## iOS 建置與部署

### Fastlane 設定

```bash
cd ios
fastlane init
```

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  desc "Push to TestFlight"
  lane :beta do
    # 同步憑證
    setup_ci if ENV['CI']
    
    match(
      type: "appstore",
      readonly: true
    )
    
    # 建置
    sh("cd ../.. && flutter build ios --release --no-codesign")
    
    # 簽章並建置
    build_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.example.chargingapp" => "match AppStore com.example.chargingapp"
        }
      }
    )
    
    # 上傳到 TestFlight
    upload_to_testflight(
      skip_waiting_for_build_processing: true
    )
    
    # 通知
    slack(
      message: "iOS build uploaded to TestFlight",
      success: true
    )
  end
  
  desc "Upload to App Store"
  lane :release do
    upload_to_app_store(
      submit_for_review: true,
      automatic_release: false,
      force: true,
      skip_metadata: false,
      skip_screenshots: false,
      precheck_include_in_app_purchases: false
    )
  end
end
```

### GitHub Actions 自動部署

```yaml
# .github/workflows/deploy_ios.yml
name: Deploy iOS

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true
      
      - name: Install Fastlane
        run: |
          cd ios
          bundle install
      
      - name: Setup App Store Connect API
        env:
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
        run: |
          echo $APP_STORE_CONNECT_API_KEY > ios/app_store_connect_api_key.json
      
      - name: Setup Match certificates
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
        run: |
          cd ios
          bundle exec fastlane match appstore --readonly
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Build and deploy
        env:
          FASTLANE_USER: ${{ secrets.FASTLANE_USER }}
          FASTLANE_PASSWORD: ${{ secrets.FASTLANE_PASSWORD }}
        run: |
          cd ios
          bundle exec fastlane beta
```

## 版本號管理

自動更新版本號：

```yaml
# .github/workflows/bump_version.yml
name: Bump Version

on:
  workflow_dispatch:
    inputs:
      version_type:
        description: 'Version type (major, minor, patch)'
        required: true
        default: 'patch'

jobs:
  bump:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.PAT }}
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
      
      - name: Bump version
        run: |
          # 取得目前版本
          VERSION=$(grep "version:" pubspec.yaml | sed 's/version: //')
          
          # 分解版本號
          MAJOR=$(echo $VERSION | cut -d. -f1)
          MINOR=$(echo $VERSION | cut -d. -f2)
          PATCH=$(echo $VERSION | cut -d. -f3 | cut -d+ -f1)
          BUILD=$(echo $VERSION | cut -d+ -f2)
          
          # 依據類型更新
          if [ "${{ github.event.inputs.version_type }}" == "major" ]; then
            MAJOR=$((MAJOR + 1))
            MINOR=0
            PATCH=0
          elif [ "${{ github.event.inputs.version_type }}" == "minor" ]; then
            MINOR=$((MINOR + 1))
            PATCH=0
          else
            PATCH=$((PATCH + 1))
          fi
          
          BUILD=$((BUILD + 1))
          
          NEW_VERSION="$MAJOR.$MINOR.$PATCH+$BUILD"
          
          # 更新 pubspec.yaml
          sed -i "s/version: .*/version: $NEW_VERSION/" pubspec.yaml
          
          echo "NEW_VERSION=$NEW_VERSION" >> $GITHUB_ENV
      
      - name: Commit and push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add pubspec.yaml
          git commit -m "Bump version to ${{ env.NEW_VERSION }}"
          git tag "v${{ env.NEW_VERSION }}"
          git push origin main --tags
```

## Secrets 管理

在 GitHub Repository Settings → Secrets and variables → Actions 設定：

**Android:**
- `KEYSTORE_BASE64`: Keystore 檔案的 base64 編碼
- `KEYSTORE_PASSWORD`: Keystore 密碼
- `KEY_ALIAS`: Key alias
- `KEY_PASSWORD`: Key 密碼
- `GOOGLE_PLAY_SERVICE_ACCOUNT`: Service account JSON

**iOS:**
- `APP_STORE_CONNECT_API_KEY`: App Store Connect API Key
- `MATCH_PASSWORD`: Match password
- `MATCH_GIT_BASIC_AUTHORIZATION`: Match Git 認證
- `FASTLANE_USER`: Apple ID
- `FASTLANE_PASSWORD`: Apple ID 密碼

產生 Keystore base64：

```bash
base64 -i keystore.jks | pbcopy
```

## 建置快取

加速建置時間：

```yaml
- name: Cache Flutter dependencies
  uses: actions/cache@v3
  with:
    path: |
      ~/.pub-cache
      **/.packages
      **/.flutter-plugins
      **/.flutter-plugin-dependencies
      **/.dart_tool
    key: ${{ runner.os }}-pub-${{ hashFiles('**/pubspec.lock') }}
    restore-keys: |
      ${{ runner.os }}-pub-

- name: Cache build
  uses: actions/cache@v3
  with:
    path: |
      build
    key: ${{ runner.os }}-build-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-build-
```

## 通知整合

建置完成後發送通知：

### Slack

```ruby
# Fastfile
slack(
  message: "Build successful",
  slack_url: ENV['SLACK_WEBHOOK_URL'],
  payload: {
    "Build Date" => Time.new.to_s,
    "Version" => version_name
  },
  default_payloads: [:git_branch, :git_author]
)
```

### Discord

```yaml
- name: Discord notification
  if: always()
  uses: sarisia/actions-status-discord@v1
  with:
    webhook: ${{ secrets.DISCORD_WEBHOOK }}
    status: ${{ job.status }}
    title: "Build ${{ job.status }}"
```

## 實務心得

CI/CD 初期設定麻煩，但長期來說省超多時間。

自動測試很重要，避免 broken code 進到 main branch。設定 branch protection rule，必須通過測試才能 merge。

Fastlane 的學習曲線陡，但值得投資。處理 iOS 的簽章和憑證管理很方便。

Secrets 管理要小心，不要把敏感資訊 commit 到 repo。用 GitHub Secrets 或 environment variables。

建置時間要控制，太長會降低開發效率。用 cache 和平行執行來加速。

下週來研究 App 的多語言支援和在地化，準備上架到不同國家市場。
