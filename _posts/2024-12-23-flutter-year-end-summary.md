---
layout: post
title: "Flutter 學習年終總結與展望"
date: 2024-12-23 13:40:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Summary, Career]
---

從七月開始到現在，半年的 Flutter 專案告一段落。這週來做個年終總結，回顧學到什麼、遇到什麼挑戰，以及對未來的展望。

## 學習歷程回顧

### 第一個月（7-8月）：打基礎

**重點：**
- MVVM + DDD 架構規劃
- SQLite 本地資料庫
- Google Maps 整合
- Firebase Push Notifications
- 執行流程控制（Isolate、async/await）

**心得：**

一開始最大的挑戰是架構設計。從 Java Spring 轉到 Flutter，思維模式要調整。後端習慣分層很清楚（Controller、Service、Repository、Entity），但 Flutter 的 Presentation、Domain、Data 分層一開始不太習慣。

SQLite 和 SharedPreferences 的選擇也花了時間研究。最後決定用「冷熱資料分離」策略：常用的放記憶體快取，不常用的存 SQLite，設定類的用 SharedPreferences。

Google Maps 整合是第一個「哇！」的時刻。看到自訂 Marker、Clustering 在地圖上運作，很有成就感。但也踩了不少坑，例如 iOS 的權限設定、Marker Icon 的記憶體問題等。

### 第二個月（9月）：進階功能

**重點：**
- Testing（Unit、Widget、Integration）
- 效能分析與優化
- CI/CD（GitHub Actions + Fastlane）
- 國際化（i18n）
- 資料持久化（Hive、Drift）

**心得：**

測試是最不想做但又最重要的事。一開始覺得寫測試很浪費時間，但後來幾次重構時，測試救了我。有測試保護，重構很放心。

CI/CD 設定花了不少時間，尤其是 iOS 的憑證和 Provisioning Profile。但自動化後很爽，推 code 就自動測試、自動 build、自動部署到 TestFlight。

國際化比想像中複雜。不只是翻譯文字，還有日期格式、貨幣格式、文字方向（RTL）、圖片本地化等。學到「一開始就要考慮 i18n」這個教訓。

### 第三個月（10月）：生產準備

**重點：**
- Deep Link & App Links
- 背景任務（WorkManager）
- App 上架準備
- Analytics & Monitoring
- 安全性（混淆、加密、SSL Pinning）

**心得：**

Deep Link 是個大坑。iOS 的 Universal Links 和 Android 的 App Links 設定完全不同。iOS 要設定 AASA 檔案在伺服器，Android 要設定 assetlinks.json。測試也很麻煩，真機和模擬器行為不一樣。

背景任務在 iOS 很受限。Android 的 WorkManager 相對彈性，但 iOS 的 Background Fetch 有很多限制。最後用 Foreground Service + Local Notification 解決。

App 上架準備比寫程式更煩。截圖、描述、隱私政策、審查指南...一堆文件。還好有 Fastlane 自動化截圖，省不少時間。

### 第四個月（11月）：進階主題

**重點：**
- 自訂動畫（CustomPainter）
- Platform Channels（與 Native 整合）
- Riverpod 進階技巧
- APM 工具（Firebase Performance、Sentry）

**心得：**

CustomPainter 很有趣，可以畫任何想要的東西。用來做自訂進度條、波浪動畫、圖表等。但效能要注意，`shouldRepaint` 要實作好。

Platform Channels 讓我重新寫 Kotlin 和 Swift。藍牙、感應器這些功能 Flutter 原生不支援，要用 Native 實作。雖然麻煩，但也加深對平台的理解。

Riverpod 用久了真的回不去 Provider。型別安全、Compile-time 檢查、測試友善，都是優點。`autoDispose`、`family`、`select` 這些功能很實用。

### 第五個月（12月）：回顧與優化

**重點：**
- 架構回顧與重構
- 團隊協作規範
- 效能優化實戰
- 年終總結

**心得：**

回頭看幾個月前的程式碼，發現很多可以改進的地方。過度抽象、檔案拆太細、不必要的 UseCase 等。重構後程式碼簡潔很多。

建立團隊規範很重要。Lint、Code Review、Commit Message、PR 流程，這些標準化讓協作更順暢。一開始會覺得麻煩，但長期來看絕對值得。

效能優化是永無止境的。從 Widget rebuild、列表渲染、圖片快取、記憶體管理，每個環節都能優化。重點是用工具量測，不要憑感覺。

## 技術棧總結

### 核心技術

**狀態管理：**
- Riverpod（主要）
- 局部狀態用 StatefulWidget

**網路請求：**
- Dio + Interceptors
- Either<Failure, T> 錯誤處理

**資料持久化：**
- sqflite（結構化資料）
- SharedPreferences（設定）
- Hive（NoSQL 快取）
- flutter_secure_storage（敏感資料）

**UI/UX：**
- Material Design
- CustomPainter（自訂繪製）
- 隱式/顯式動畫
- Hero 動畫

**地圖：**
- google_maps_flutter
- Marker Clustering
- Custom Marker

**推播：**
- Firebase Cloud Messaging
- Local Notifications

**測試：**
- Unit Tests（mockito）
- Widget Tests
- Integration Tests
- Golden Tests

**CI/CD：**
- GitHub Actions
- Fastlane

**監控：**
- Firebase Analytics
- Firebase Crashlytics
- Firebase Performance
- Sentry

### 第三方套件

常用且推薦的：

```yaml
dependencies:
  # 狀態管理
  flutter_riverpod: ^2.4.0
  
  # 網路
  dio: ^5.4.0
  
  # 功能型別
  dartz: ^0.10.1
  
  # 依賴注入
  get_it: ^7.6.4
  
  # 路由
  go_router: ^13.0.0
  
  # 圖片
  cached_network_image: ^3.3.0
  
  # 資料庫
  sqflite: ^2.3.0
  hive: ^2.2.3
  
  # 安全
  flutter_secure_storage: ^9.0.0
  
  # 地圖
  google_maps_flutter: ^2.5.0
  
  # Firebase
  firebase_core: ^2.24.0
  firebase_messaging: ^14.7.0
  firebase_analytics: ^10.7.0
  
  # UI
  shimmer: ^3.0.0
  lottie: ^2.7.0
  
  # 工具
  intl: ^0.18.1
  url_launcher: ^6.2.2
```

## 最大的收穫

### 1. 架構思維

從「寫功能」進步到「設計系統」。學會如何規劃可擴展、可維護的架構，而不是堆砌功能。

### 2. 跨平台開發

一套程式碼跑 iOS/Android，但也要理解平台差異（權限、背景任務、App Links 等）。不是「Write Once, Run Everywhere」，而是「Write Once, Test Everywhere」。

### 3. 測試的重要性

一開始覺得測試浪費時間，現在覺得沒測試才是浪費時間。有測試保護，重構很安心，Debug 很快。

### 4. 效能意識

不只是「能跑」，還要「跑得好」。Widget rebuild、列表渲染、圖片快取，每個細節都影響使用者體驗。

### 5. 團隊協作

程式不是只有你在寫。規範、文件、Code Review，這些「軟技能」和技術一樣重要。

## 遇到的挑戰

### 1. iOS 開發環境

Mac、Xcode、憑證、Provisioning Profile...iOS 開發門檻真的高。相比之下 Android 友善很多。

### 2. 平台差異

很多細節 iOS/Android 不一樣。權限、背景任務、推播、Deep Link，都要分開處理。

### 3. 套件品質參差不齊

有些套件很久沒更新，或者有 Bug。要學會看原始碼，必要時自己 Fork 修改。

### 4. 效能調校

Flutter 預設效能不錯，但要達到「絲滑」還是要優化。DevTools 是好工具，但要懂得看。

### 5. 版本升級

Flutter 更新很快，有時升級會 breaking changes。要謹慎評估是否升級，測試很重要。

## 對 Flutter 的看法

### 優點

- **開發效率高**：熱重載、豐富的 Widget、狀態管理簡單
- **跨平台**：一套程式碼，iOS/Android/Web 都能跑
- **效能好**：接近原生效能，比 React Native 流暢
- **社群活躍**：套件多、文件豐富、問題好找答案
- **UI 靈活**：想要什麼 UI 都能做出來

### 缺點

- **學習曲線**：Dart 語言、Widget 樹、狀態管理，初學要時間
- **套件品質**：有些套件品質不穩，要自己踩坑
- **包大小**：初始包比較大（10-15MB），Release 後會小一點
- **平台限制**：某些 Native 功能還是要自己實作
- **Web 支援**：雖然能跑，但體驗不如原生 Web

### 適合什麼專案？

**適合：**
- MVP 快速驗證
- 中小型 App
- 需要快速迭代的專案
- UI 要求高的 App

**不適合：**
- 重度遊戲（用 Unity）
- 需要極致效能的 App
- 大量原生功能的 App
- 純 Web 應用（用 React/Vue）

## 未來展望

### 短期（3-6 個月）

- **深入 Native 整合**：iOS/Android 原生開發能力
- **Flutter Web**：研究 Web 端的最佳實踐
- **狀態管理**：嘗試其他方案（Bloc、MobX）
- **效能極致優化**：達到 60fps 穩定幀率

### 長期（1 年以上）

- **跨平台架構師**：能設計大型跨平台系統
- **開源貢獻**：貢獻 Flutter 套件或核心
- **技術分享**：寫文章、演講、教學
- **新技術**：關注 Flutter 新特性（Impeller、WebAssembly）

## 給想學 Flutter 的人

### 建議

1. **紮實的 Dart 基礎**：先學好 Dart，再學 Flutter
2. **理解 Widget 樹**：Flutter 的核心概念
3. **選好狀態管理**：Riverpod 或 Bloc，不要用 setState 到處飛
4. **寫測試**：從一開始就寫，不要偷懶
5. **讀文件**：Flutter 文件很完整，有問題先查文件
6. **實作專案**：看再多文章不如寫一個 App
7. **持續學習**：Flutter 更新快，要跟上

### 學習資源

- **官方文件**：flutter.dev
- **Cookbook**：實用範例
- **YouTube**：Flutter 官方頻道
- **套件**：pub.dev
- **社群**：Reddit、Discord、Stack Overflow

## 結語

半年的 Flutter 開發，從零到能獨立開發完整的 App，成長很多。

技術只是工具，重要的是解決問題的思維。架構設計、效能優化、測試、團隊協作，這些經驗比特定技術更有價值。

Flutter 是個好工具，但不是萬能。要根據專案需求選擇合適的技術，不要盲目追求新技術。

2024 年就這樣過了。感謝這個專案讓我學到這麼多，也感謝團隊一起努力。

2025 年，繼續加油！🚀
