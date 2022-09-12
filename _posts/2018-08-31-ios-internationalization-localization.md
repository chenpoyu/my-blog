---
layout: post
title: "iOS 國際化與在地化：多語言 App 開發"
date: 2018-08-31
categories: [iOS, Localization]
tags: [i18n, l10n, Localization, Internationalization]
---

這週研究如何讓 app 支援多語言和不同地區，這對全球化產品很重要。iOS 提供完整的國際化（Internationalization, i18n）和在地化（Localization, l10n）支援。

## 基本概念

**國際化（i18n）**：讓程式碼能支援多語言，不寫死文字
**在地化（l10n）**：為特定語言和地區提供翻譯資源

## 第一步：建立 Localizable.strings

1. File → New → File → Strings File
2. 命名為 `Localizable.strings`
3. 選擇檔案，在右側 Inspector 點選 "Localize"
4. 選擇語言（English、繁體中文等）

**Localizable.strings (English)**：

```
"welcome_message" = "Welcome to our app!";
"login_button" = "Login";
"username_placeholder" = "Enter username";
```

**Localizable.strings (Chinese Traditional)**：

```
"welcome_message" = "歡迎使用我們的 app！";
"login_button" = "登入";
"username_placeholder" = "輸入使用者名稱";
```

**程式碼中使用**：

```swift
// 基本用法
let message = NSLocalizedString("welcome_message", comment: "Welcome message on home screen")
label.text = message

// 簡化的 helper
extension String {
    var localized: String {
        return NSLocalizedString(self, comment: "")
    }
}

label.text = "welcome_message".localized
```

## 字串插值與複數規則

**帶參數的字串**：

```swift
// Localizable.strings (English)
"items_count" = "You have %d items";

// 程式碼
let count = 5
let message = String(format: NSLocalizedString("items_count", comment: ""), count)
// "You have 5 items"
```

**複數規則（Plurals）**：

不同語言的複數規則不同，用 `.stringsdict` 處理：

**Localizable.stringsdict**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>items_count</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%#@item_count@</string>
        <dict>
            <key>item_count</key>
            <dict>
                <key>NSStringFormatSpecTypeKey</key>
                <string>NSStringPluralRuleType</string>
                <key>NSStringFormatValueTypeKey</key>
                <string>d</string>
                <key>zero</key>
                <string>No items</string>
                <key>one</key>
                <string>One item</string>
                <key>other</key>
                <string>%d items</string>
            </dict>
        </dict>
    </dict>
</dict>
</plist>
```

**程式碼使用**：

```swift
let count = 0
let message = String.localizedStringWithFormat(
    NSLocalizedString("items_count", comment: ""),
    count
)
// count = 0: "No items"
// count = 1: "One item"
// count = 5: "5 items"
```

## Storyboard 和 XIB 的在地化

**自動產生字串檔案**：

1. 選擇 Storyboard，在 Inspector 點選 "Localize"
2. Xcode 會產生 `Main.strings` 檔案
3. 為每個語言提供翻譯

**Main.strings (English)**：

```
"Login.button.title" = "Login";
"Username.textField.placeholder" = "Username";
```

**Main.strings (Chinese)**：

```
"Login.button.title" = "登入";
"Username.textField.placeholder" = "使用者名稱";
```

更好的做法是用 IBOutlet 在程式碼中設定，避免 Storyboard 字串難以維護。

## 圖片資源的在地化

有些圖片需要針對不同語言調整（如包含文字的圖片）：

1. 選擇圖片資產，在 Inspector 點選 "Localize"
2. 為不同語言提供不同的圖片

**程式碼中使用**：

```swift
// 自動載入對應語言的圖片
let image = UIImage(named: "welcome_banner")
```

## 日期、數字、貨幣的格式化

**日期格式**：

```swift
let date = Date()
let formatter = DateFormatter()
formatter.dateStyle = .long
formatter.timeStyle = .short

// 自動使用使用者的語言設定
let dateString = formatter.string(from: date)
// English: "August 31, 2018 at 10:30 AM"
// Chinese: "2018年8月31日 上午10:30"
```

**數字格式**：

```swift
let number = 1234567.89
let formatter = NumberFormatter()
formatter.numberStyle = .decimal

let numberString = formatter.string(from: NSNumber(value: number))
// English: "1,234,567.89"
// Chinese: "1,234,567.89"（可能根據地區不同）
```

**貨幣格式**：

```swift
let price = 1234.56
let formatter = NumberFormatter()
formatter.numberStyle = .currency
formatter.locale = Locale.current

let priceString = formatter.string(from: NSNumber(value: price))
// US: "$1,234.56"
// Taiwan: "NT$1,234.56"
// Japan: "¥1,235"（日圓沒有小數）
```

## 程式碼中處理不同語言

**取得當前語言**：

```swift
let currentLanguage = Locale.current.languageCode
// "en", "zh", "ja" etc.

let preferredLanguage = Locale.preferredLanguages.first
// "en-US", "zh-Hant-TW" etc.
```

**判斷 RTL（從右到左）語言**：

```swift
let isRTL = UIApplication.shared.userInterfaceLayoutDirection == .rightToLeft

// 或用 Locale
let locale = Locale.current
let isRTL = Locale.characterDirection(forLanguage: locale.languageCode!) == .rightToLeft

// 自動支援 RTL
// UIStackView 和 Auto Layout 會自動調整
```

## 測試不同語言

**在模擬器測試**：
1. Settings → General → Language & Region
2. 選擇不同語言
3. 重啟 app

**在 Xcode 測試**：
1. Edit Scheme → Run → Options
2. Application Language 選擇語言
3. 不用修改系統設定

**截圖不同語言版本**：

```swift
// 用 UI Testing 自動截圖
class LocalizationUITests: XCTestCase {
    func testScreenshotsInDifferentLanguages() {
        let languages = ["en", "zh-Hant", "ja"]
        
        for language in languages {
            // 設定語言
            let app = XCUIApplication()
            app.launchArguments = ["-AppleLanguages", "(\(language))"]
            app.launch()
            
            // 截圖
            let screenshot = app.screenshot()
            let attachment = XCTAttachment(screenshot: screenshot)
            attachment.name = "Screenshot_\(language)"
            add(attachment)
        }
    }
}
```

## 最佳實踐

### 1. 不要拼接字串

```swift
// 錯誤
let message = "Hello " + userName + ", you have " + "\(count)" + " messages"

// 正確
// Localizable.strings: "greeting_message" = "Hello %@, you have %d messages";
let message = String(format: "greeting_message".localized, userName, count)
```

不同語言的語序可能不同，拼接會出問題。

### 2. 提供充足的 UI 空間

德文通常比英文長 30%，要預留空間：

```swift
// 不要固定寬度
button.widthAnchor.constraint(equalToConstant: 80)  // 可能放不下德文

// 用內容自適應
button.setContentHuggingPriority(.required, for: .horizontal)
```

### 3. 使用 genstrings 生成字串檔案

```bash
# 自動從程式碼中提取 NSLocalizedString
find . -name "*.swift" -print0 | xargs -0 genstrings -o en.lproj
```

### 4. 翻譯管理

大型專案可用工具管理翻譯：
- **Lokalise**：協作翻譯平台
- **Crowdin**：群眾翻譯
- **POEditor**：翻譯管理系統

### 5. Context 很重要

```swift
// 不好：comment 空白
NSLocalizedString("OK", comment: "")

// 好：提供 context 給翻譯人員
NSLocalizedString("OK", comment: "Confirmation button on delete dialog")
```

同一個詞在不同 context 可能有不同翻譯。

## 地區特定功能

**電話號碼格式**：

```swift
let phoneNumber = "0912345678"
let formatter = PhoneNumberFormatter()  // 需要第三方庫如 PhoneNumberKit
let formattedNumber = formatter.format(phoneNumber, for: .TW)
// Taiwan: "0912-345-678"
// US: "(091) 234-5678"
```

**地址格式**：

不同國家地址格式差異很大：
- 美國：街道、城市、州、郵遞區號
- 台灣：郵遞區號、縣市、區、街道
- 日本：郵遞區號、都道府縣、市區町村、街道

用 `CNPostalAddress` 處理：

```swift
import Contacts

let address = CNMutablePostalAddress()
address.street = "忠孝東路一段"
address.city = "台北市"
address.postalCode = "100"
address.country = "台灣"

let formatter = CNPostalAddressFormatter()
let formattedAddress = formatter.string(from: address)
```

## 實戰經驗

我在實作多語言時遇到的問題：

**問題 1**：翻譯後 UI 跑版
- 原因：英文按鈕標題 "OK"，中文變成 "確定"，長度增加
- 解決：用 Auto Layout 讓按鈕寬度自適應

**問題 2**：日期格式不一致
- 原因：手動拼接日期字串
- 解決：用 DateFormatter 自動格式化

**問題 3**：複數規則錯誤
- 原因：簡單用 count == 1 判斷
- 解決：用 .stringsdict 處理不同語言的複數規則

## 學習心得

國際化不只是翻譯文字，還要考慮：
- UI 佈局要適應不同長度的文字
- 日期、數字、貨幣格式要符合當地習慣
- RTL 語言（阿拉伯文、希伯來文）需要鏡像 UI
- 圖片、顏色可能有文化差異

建議從專案一開始就規劃國際化，而非事後才加入。用 NSLocalizedString 而非寫死字串，未來要支援新語言會容易很多。
