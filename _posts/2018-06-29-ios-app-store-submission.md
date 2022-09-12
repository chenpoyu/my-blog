---
layout: post
title: "iOS App Store 上架流程"
date: 2018-06-29
categories: [iOS, Swift]
tags: [Swift, iOS, App Store, Distribution, Certificate]
---

這週研究如何把 app 發布到 App Store。在 Java Web 開發中，部署通常是把 WAR 檔上傳到伺服器。但 iOS app 的發布流程複雜很多，涉及憑證、描述檔、審查機制等。這是把 app 交到使用者手中的最後也是最關鍵的一步。

## App Store 發布的完整流程

整個流程可以分成幾個階段：

**準備階段**：設定 App ID、憑證、描述檔  
**開發階段**：實作功能、測試  
**提交前準備**：截圖、描述、隱私權政策  
**提交審查**：上傳二進位檔、填寫資訊  
**審查階段**：Apple 審查（1-7 天）  
**發布**：審查通過後上架

這個流程比 Web 應用複雜，因為 Apple 對 app 品質和內容有嚴格把關。好處是確保生態系品質，壞處是增加了開發者的工作量。

## 憑證與描述檔

### Apple Developer Program

首先要加入 Apple Developer Program，年費 99 美元。沒有註冊就無法發布到 App Store。

這讓我想到 Java 開發不需要特殊授權就能部署應用。但 Apple 的封閉生態系透過授權費用維護平台品質。

### 憑證類型

Apple 的憑證系統相當複雜：

**開發憑證（Development Certificate）**：用於本機開發和測試  
**發布憑證（Distribution Certificate）**：用於提交 App Store  
**推送憑證（Push Certificate）**：用於發送推送通知

這些憑證在 Apple Developer 網站的 Certificates, Identifiers & Profiles 區域管理。

建立憑證的流程：
1. 在 Keychain Access 建立 Certificate Signing Request（CSR）
2. 上傳 CSR 到 Apple Developer 網站
3. 下載產生的憑證
4. 雙擊安裝到 Keychain

這個流程確保私鑰只存在開發者的電腦上，Apple 無法冒充開發者簽署 app。這是公開金鑰基礎建設（PKI）的標準做法。

### Provisioning Profile

描述檔定義了「哪些裝置可以執行這個 app」：

**Development Profile**：包含特定測試裝置的 UDID  
**Ad Hoc Profile**：用於內部測試，最多 100 台裝置  
**App Store Profile**：用於 App Store 發布，無裝置限制

描述檔包含：
- App ID
- 憑證
- 裝置列表（Development 和 Ad Hoc）
- 權限（Capabilities）

Xcode 會自動管理描述檔，但了解原理有助於解決簽署問題。我遇過很多次「Code signing error」，都是因為憑證或描述檔設定錯誤。

## 設定 App 資訊

在 Xcode 專案設定中：

**General 標籤**：
- **Display Name**：顯示在手機桌面的名稱
- **Bundle Identifier**：唯一識別碼，格式如 `com.company.appname`
- **Version**：給使用者看的版本號（如 1.0.0）
- **Build**：內部版本號，每次上傳要遞增

**Signing & Capabilities**：
- 選擇 Team
- 勾選 Automatically manage signing（讓 Xcode 自動處理）
- 加入需要的 Capabilities（推送通知、iCloud 等）

**Info.plist 設定**：
- 權限使用說明（相機、定位等）
- 支援的裝置方向
- 狀態列樣式

這些設定類似 Android 的 AndroidManifest.xml 或 Java Web 的 web.xml，定義 app 的基本資訊和權限。

## 準備 App Store 素材

### 截圖

需要為不同裝置尺寸準備截圖：
- **6.5 吋**：iPhone 14 Pro Max
- **5.5 吋**：iPhone 8 Plus
- **iPad Pro (12.9 吋)**
- **iPad Pro (11 吋)**

每種尺寸至少 1 張，最多 10 張。截圖要展示 app 的核心功能和特色。

**最佳實踐**：
- 第一張截圖最重要，要立刻抓住注意力
- 使用文字說明功能，不要只放純畫面
- 展示真實內容而非空白狀態
- 保持視覺風格一致

我看過很多 app 的截圖只是隨便抓幾張畫面，沒有說明，使用者根本看不懂 app 在做什麼。好的截圖能大幅提升下載率。

### App Icon

Icon 有多種尺寸需求：
- 1024x1024：App Store 展示
- 180x180：iPhone (@3x)
- 120x120：iPhone (@2x)
- 還有更多 iPad、Spotlight、Settings 用的尺寸

Xcode 的 Assets.xcassets 中有 AppIcon 資源夾，把各尺寸的圖示拖進去即可。

**設計要點**：
- 簡潔明瞭，一眼就知道 app 用途
- 避免文字（小圖示看不清楚）
- 使用鮮明的顏色
- 不要直接用系統圖示

### App 描述

**名稱**：最多 30 字元，要有辨識度且好記

**副標題**：最多 30 字元，補充說明 app 特色

**描述**：最多 4000 字元，詳細說明功能。要包含：
- app 的核心價值
- 主要功能列表
- 使用者評價（如果有）
- 聯絡資訊

**關鍵字**：最多 100 字元，用逗號分隔。這影響搜尋排名，要精心挑選。

**分類**：選擇最相關的類別（如「工具程式」、「健康與健身」）

這些文案很重要，會直接影響轉換率。我的經驗是**第一段要直接說明核心價值**，不要囉嗦背景故事。使用者沒耐心看長篇大論。

## 設定 App Store Connect

在 [App Store Connect](https://appstoreconnect.apple.com) 建立 app：

1. **我的 App → 新增 App**
2. 填寫基本資訊：平台、名稱、語言、Bundle ID、SKU
3. **定價與供應狀況**：選擇免費或付費，以及上架地區
4. **App 隱私權**：說明 app 收集哪些資料、如何使用

隱私權設定在 App Store 審核中越來越重要，必須詳細說明資料使用方式。這包括：
- 收集哪些資料（位置、聯絡資訊、使用數據等）
- 資料是否連結到使用者身分
- 資料是否用於追蹤
- 資料使用目的（分析、廣告、app 功能等）

不誠實申報會被拒絕或下架，要謹慎填寫。

## 上傳二進位檔

### Archive 建置

在 Xcode 中：
1. 選擇 **Generic iOS Device** 或實體裝置
2. **Product → Archive**
3. 等待建置完成（會產生 .xcarchive 檔案）

Archive 過程會：
- 編譯程式碼為 release 模式
- 最佳化效能
- 去除除錯符號（但保留 dSYM）
- 簽署 app

### 上傳到 App Store Connect

Archive 完成後會開啟 Organizer 視窗：
1. 選擇剛建立的 archive
2. 點擊 **Distribute App**
3. 選擇 **App Store Connect**
4. 選擇上傳選項：
   - Upload symbols：上傳符號檔（用於 crash 分析）
   - Manage version and build number：自動管理版本號
5. 簽署選項：通常選自動簽署
6. 點擊 **Upload**

上傳後約 5-10 分鐘會在 App Store Connect 看到新的建置版本。如果超過 30 分鐘還沒出現，可能上傳失敗，檢查信箱是否有錯誤通知。

### 常見上傳錯誤

**ITMS-90562: Invalid Bundle**：Bundle ID 不匹配  
**ITMS-90171: Invalid Bundle Structure**：缺少必要檔案  
**ITMS-90174: Missing Provisioning Profile**：描述檔問題

遇到錯誤時，Xcode 會顯示詳細訊息，根據指示修正即可。

## 提交審查

二進位檔上傳後，在 App Store Connect 填寫審查資訊：

**建置版本**：選擇要提交的 build

**審查資訊**：
- 聯絡資訊（Apple 有問題會聯繫）
- Demo 帳號（如果 app 需要登入）
- 備註（告訴審查人員測試重點）

**版本資訊**：
- 本次更新內容（What's New）
- 行銷用 URL（可選）
- 支援 URL（必填）
- 隱私權政策 URL（如果收集個人資料，必填）

**年齡分級**：回答問卷決定 app 的年齡限制（4+、9+、12+、17+）

填寫完畢後點擊**提交審查**。

## App 審查機制

Apple 會審查 app 是否符合 [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)。

主要審查項目：

**安全性**：
- 不能有惡意程式碼
- 不能繞過 iOS 安全機制
- 加密要合規

**效能**：
- 不能崩潰或有明顯 bug
- 不能過度耗電或耗流量
- 載入時間要合理

**商業模式**：
- 內購要用 Apple 的系統
- 不能引導使用者去外部購買
- 訂閱要清楚說明條款

**設計**：
- UI 要符合 Human Interface Guidelines
- 不能抄襲其他 app 的 UI
- 要完整實作功能，不能是半成品

**法律**：
- 不能侵犯版權或商標
- 隱私權政策要完整
- 兒童 app 有特殊規範

**內容**：
- 不能有冒犯性內容
- 不能涉及政治敏感議題
- 賭博類 app 有特殊限制

審查通常需要 1-7 天。如果被拒絕，會收到詳細的拒絕原因，修正後可以再次提交。

### 常見拒絕原因

**2.1 App Completeness**：功能不完整，有明顯 bug  
**4.2 Minimum Functionality**：功能太簡單，沒有提供價值  
**5.1.1 Data Collection and Storage**：違反隱私權規範  
**2.3.10 Accurate Metadata**：截圖或描述與實際功能不符

我第一次提交的 app 被拒絕，原因是「測試帳號無法登入」。原來我給的帳號過期了。修正後第二次就通過了。

## 加速審查

如果有緊急需求（如修復嚴重 bug），可以申請**加速審查**：

在 App Store Connect → 聯絡我們 → Request an Expedited App Review

需要說明：
- 為什麼需要加速
- 影響哪些使用者
- 商業理由

Apple 不保證會接受，而且一年只能申請少數幾次。不要濫用。

## 發布與更新

審查通過後，app 狀態變成「待發售」。可以：

**立即發布**：手動點擊「發布此版本」  
**排程發布**：設定特定日期時間自動上架  
**審查通過後自動發布**：在提交時選擇此選項

發布後，使用者通常在幾小時內就能搜尋到 app 並下載。

### 版本更新

更新 app 的流程和初次提交類似：
1. 增加版本號（如從 1.0 改成 1.1）
2. 增加 build 號
3. Archive 並上傳
4. 在 App Store Connect 建立新版本
5. 填寫「本次更新內容」
6. 提交審查

更新審查通常比初次審查快，因為 app 已經在商店中了。

## 發布後的監控

### Crash 報告

在 App Store Connect → Analytics → Crashes 可以看到崩潰報告。

要注意：
- Crash-free users 百分比（應保持在 99% 以上）
- 最常見的崩潰原因
- 影響的 iOS 版本和裝置

崩潰率太高會影響 app 排名，甚至被 Apple 警告。

### 使用者評論

使用者評論對 app 成功很關鍵。要：
- 定期回覆評論（特別是負評）
- 根據反饋改進功能
- 鼓勵滿意的使用者留下好評

可以用 `SKStoreReviewController` 在適當時機請求評分：

```swift
import StoreKit

func requestReview() {
    // 在適當時機呼叫（如完成重要任務後）
    if let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene {
        SKStoreReviewController.requestReview(in: scene)
    }
}
```

注意不要太頻繁請求，否則使用者會反感。

## 小結

這週研究 App Store 上架流程，最大的感受是 **Apple 對品質的嚴格把關**。

雖然審查流程增加了開發者的工作量，但也確保了生態系的整體品質。作為使用者，我確實覺得 App Store 的 app 品質普遍比其他平台高。

關鍵要點：
1. **提前準備**：憑證、截圖、描述都要事先準備好
2. **遵守規範**：仔細閱讀 Review Guidelines，避免被拒
3. **提供測試資訊**：Demo 帳號、測試步驟要清楚
4. **即時回應**：審查過程中 Apple 可能有問題，要及時回覆
5. **持續優化**：根據使用者反饋和 crash 報告改進 app

下週是最後一週的學習，計劃總結這 9 個月的 iOS 開發經驗，整理從 Java 轉到 iOS 的心得，以及未來的學習方向。
