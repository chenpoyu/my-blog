---
layout: post
title: "Flutter App 上架準備：商店素材與審核要點"
date: 2024-10-14 10:15:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, App Store, Google Play, Publishing]
---

終於要上架了！這週整理 App Store 和 Google Play 的上架流程。除了 App 本身，還需要準備截圖、圖示、描述等素材，以及通過審核。

## App 圖示

### 尺寸需求

**iOS**:
- 1024x1024 (App Store)
- 180x180 (iPhone)
- 167x167 (iPad Pro)
- 152x152 (iPad)
- 120x120 (iPhone 小)
- 87x87 (設定)
- 80x80 (Spotlight)
- 76x76 (iPad)
- 58x58、40x40、29x29 (各種小圖示)

**Android**:
- 512x512 (Play Store)
- xxxhdpi: 192x192
- xxhdpi: 144x144
- xhdpi: 96x96
- hdpi: 72x72
- mdpi: 48x48

### 使用 flutter_launcher_icons

自動產生各尺寸圖示：

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.0

flutter_launcher_icons:
  android: true
  ios: true
  image_path: "assets/icon/app_icon.png"  # 1024x1024
  adaptive_icon_background: "#FFFFFF"
  adaptive_icon_foreground: "assets/icon/foreground.png"
```

執行：

```bash
flutter pub run flutter_launcher_icons
```

## App 截圖

### 需求

**iOS**:
- iPhone 6.7" (Pro Max): 1290 x 2796
- iPhone 6.5" (Plus): 1242 x 2688
- iPhone 5.5": 1242 x 2208
- iPad Pro 12.9": 2048 x 2732

每個尺寸 2-10 張截圖。

**Android**:
- 手機: 最少 2 張，建議 1080 x 1920
- 平板: 至少 1 張 (optional)
- 7 吋平板: 1024 x 600
- 10 吋平板: 1280 x 800

### 截圖工具

用模擬器截圖，然後用 **Fastlane Frameit** 加裝置邊框。

或用線上工具：
- [AppMockUp](https://app-mockup.com/)
- [Previewed](https://previewed.app/)
- [Shotbot](https://shotbot.io/)

### 截圖內容建議

1. **首頁/主要功能**：讓人一眼看懂 App 做什麼
2. **核心流程**：充電站搜尋 → 詳情 → 開始充電
3. **特色功能**：地圖、收藏、歷史紀錄
4. **使用者體驗**：精美 UI、流暢動畫

### 加文字說明

截圖不只是畫面，加上文字說明更清楚：

- "快速找到附近充電站"
- "即時查看可用充電樁"
- "追蹤充電進度與費用"
- "收藏常用充電站"

用 Figma 或 Sketch 製作統一風格的截圖範本。

## App 描述

### 標題

**iOS**: 最多 30 字元
**Android**: 最多 50 字元

範例：`充電 App - 電動車充電站搜尋`

### 副標題 (iOS)

最多 30 字元

範例：`快速找到附近充電站`

### 簡短描述 (Android)

最多 80 字元

範例：`快速搜尋附近電動車充電站，即時查看可用充電樁，追蹤充電進度`

### 完整描述

**iOS**: 最多 4000 字元
**Android**: 最多 4000 字元

結構：
1. **開場**：解決什麼問題
2. **主要功能**：條列式說明
3. **特色**：為什麼選擇這個 App
4. **支援資訊**：客服、網站

範例：

```
【找充電站，就是這麼簡單】

開電動車最怕找不到充電站？充電 App 讓您輕鬆找到附近所有充電站，即時查看可用充電樁數量，再也不用擔心白跑一趟！

【主要功能】

📍 地圖搜尋
• 顯示附近所有充電站位置
• 即時查看可用 / 總充電樁數
• 一鍵導航到充電站

⚡ 充電管理
• 掃描 QR Code 開始充電
• 即時追蹤充電進度
• 充電完成推播通知

📊 歷史紀錄
• 查看所有充電紀錄
• 統計總充電次數與費用
• 匯出充電報表

⭐ 個人化
• 收藏常用充電站
• 設定充電提醒
• 多語言支援（繁中、簡中、英文）

【為什麼選擇充電 App】

✓ 資料即時更新，不用擔心資訊過時
✓ 介面簡潔易用，老少咸宜
✓ 支援離線地圖，沒網路也能用
✓ 完全免費，無廣告

【需要協助？】

網站：https://example.com
客服信箱：support@example.com
客服時間：週一至週五 9:00-18:00
```

### 關鍵字 (iOS)

最多 100 字元，用逗號分隔。

範例：`充電站,電動車,EV,充電,Tesla,充電樁,電車,綠能`

選擇：
- 相關性高的詞
- 搜尋量大的詞
- 競爭度適中的詞

工具：
- [AppTweak](https://www.apptweak.com/)
- [Sensor Tower](https://sensortower.com/)

## 分類與年齡分級

### 分類

**主要分類**：
- iOS: Navigation (導航)
- Android: Maps & Navigation

**次要分類** (iOS)：Utilities (工具)

### 年齡分級

充電站 App 通常是 4+ / Everyone：
- 無暴力、色情、恐怖內容
- 無賭博、藥物相關
- 無社交功能或有適當過濾

## 隱私政策

必須提供隱私政策 URL。

內容應包含：
- 收集哪些資料
- 如何使用資料
- 是否分享給第三方
- 使用者權利（存取、刪除）
- 聯絡方式

範例架構：

```markdown
# 隱私政策

最後更新：2024 年 10 月

## 我們收集的資料

### 自動收集
- 位置資訊（用於搜尋附近充電站）
- 裝置資訊（型號、作業系統版本）
- 使用記錄（充電次數、時間）

### 使用者提供
- 電子郵件（註冊帳號）
- 個人檔案（姓名、照片）

## 資料用途

- 提供充電站搜尋服務
- 記錄充電歷史
- 改善 App 功能
- 發送充電完成通知

## 資料分享

我們不會出售您的個人資料。

可能分享的情況：
- 法律要求
- 保護使用者安全
- 與服務供應商（AWS、Firebase）

## 資料安全

- 使用 HTTPS 加密傳輸
- 密碼經過雜湊處理
- 定期備份資料

## 使用者權利

您有權：
- 存取您的資料
- 更正錯誤資料
- 刪除您的帳號
- 拒絕資料收集（但可能影響功能）

## 聯絡我們

電子郵件：privacy@example.com
地址：台北市信義區...
```

## 審核注意事項

### iOS App Store

**常見被拒原因**：

1. **Guideline 2.1 - App Completeness**
   - Bug 或 Crash
   - 功能不完整
   - Demo 或 Beta 版本

2. **Guideline 4.0 - Design**
   - UI 不符合 Human Interface Guidelines
   - 按鈕太小、觸控區域不足
   - 文字過小難以閱讀

3. **Guideline 5.1.1 - Privacy**
   - 未提供隱私政策
   - 未說明位置資訊用途
   - 未經同意收集資料

4. **Guideline 2.3.10 - Accurate Metadata**
   - 截圖與實際功能不符
   - 描述誇大不實

**避免方式**：
- 徹底測試，確保無 Crash
- 遵循 HIG 設計規範
- 提供清楚的隱私政策
- 截圖和描述要真實

### Google Play

**常見問題**：

1. **敏感權限**
   - 位置權限要說明用途
   - 相機、麥克風等要有明確理由

2. **誤導性內容**
   - 圖示或截圖誤導
   - 假冒知名品牌

3. **目標對象**
   - 兒童 App 有更嚴格規範
   - 不適合兒童的內容要標示

## 測試帳號

如果 App 需要登入，要提供測試帳號給審核人員。

在 App Store Connect 的 "App Review Information" 填寫：

```
測試帳號：reviewer@example.com
密碼：Test123456

測試流程：
1. 使用上述帳號登入
2. 允許位置權限
3. 在地圖上點擊任一充電站
4. 點擊「開始充電」測試充電流程
```

## 版本號管理

```yaml
# pubspec.yaml
version: 1.0.0+1
```

- `1.0.0`: 版本名稱 (Version Name / CFBundleShortVersionString)
- `+1`: 版本號碼 (Version Code / CFBundleVersion)

規則：
- 主要更新：2.0.0
- 新功能：1.1.0
- Bug 修復：1.0.1
- Build 號每次都要遞增

## 發布檢查清單

**程式碼**：
- [ ] 移除所有 debug code
- [ ] 關閉 debug 模式
- [ ] 更新版本號
- [ ] 執行所有測試
- [ ] 檢查效能

**素材**：
- [ ] App 圖示 (所有尺寸)
- [ ] 截圖 (所有裝置)
- [ ] 描述文字
- [ ] 隱私政策 URL
- [ ] 支援 URL

**設定**：
- [ ] 正確的 Bundle ID / Package Name
- [ ] 簽章設定
- [ ] API keys (Production)
- [ ] 分析追蹤設定

**審核**：
- [ ] 測試帳號
- [ ] 審核備註
- [ ] 聯絡資訊

## 實務心得

上架準備很煩，但第一次最煩，之後就熟了。

截圖和描述很重要，影響下載轉換率。多花時間打磨值得。

隱私政策不能馬虎，蘋果和 Google 都很重視。有疑慮就請教律師。

iOS 審核嚴格但一致，Android 較寬鬆。被拒不要氣餒，修正後再送。

版本號要有規律，不要亂跳。用戶看到版本號會判斷 App 成熟度。

預留審核時間：iOS 通常 1-3 天，Android 幾小時到 1 天。重大節日前會更久。

下週來研究 Flutter 的分析與監控，追蹤使用者行為和 App 健康度。
