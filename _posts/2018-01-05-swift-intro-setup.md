---
layout: post
title: "Swift 入門與環境設置"
date: 2018-01-05
categories: [Swift, iOS]
tags: [Swift, iOS, Xcode]
---

## 前言

作為一個長期從事 Java 開發的工程師，公司最近指派我開始接觸 iOS 開發。Swift 4 在去年九月正式發布，相較於 Objective-C，Swift 的語法對於有物件導向背景的開發者來說更加親和。這是我學習 Swift 的第一週筆記。

老實說，一開始聽到要學新語言心裡有些抗拒。Java 寫得好好的，為什麼要跳到另一個完全不同的生態系？但深入了解後發現，Swift 其實借鑑了很多現代語言的優點，包括 Java 8 的一些特性。而且 iOS 市場確實很大，多學一個平台對職涯發展有幫助。

## 為什麼選擇 Swift 而不是 Objective-C

研究後發現 iOS 開發有兩個選擇：

**Objective-C**：從 1980 年代就存在的老牌語言，語法比較特殊（方法呼叫用中括號），學習曲線陡峭。但很多舊專案還在用，文件和第三方函式庫也很豐富。

**Swift**：Apple 在 2014 年推出，現在已經 4.0 版。語法現代、安全性高、效能好，是 Apple 主推的方向。

我選擇 Swift 的原因：
- **現代語言特性**：Optional、型別推斷、閉包等，和 Java 8 有相通之處
- **安全性**：編譯期就能抓到很多錯誤，減少執行時期崩潰
- **效能**：接近 C 的速度，比 Objective-C 快
- **未來趨勢**：Apple 會持續投資 Swift 生態系統

雖然學 Swift 代表很多 Objective-C 的資源用不上，但長遠來看是值得的投資。而且 Swift 能和 Objective-C 互通，真的需要時可以混用。

## 開發環境設置

### 安裝 Xcode

iOS 開發必須使用 macOS 系統，主要開發工具是 Xcode。從 Mac App Store 下載 Xcode 9.2，這是目前支援 Swift 4 的最新版本。Xcode 包含了完整的 IDE、模擬器、以及各種開發工具。

這點和 Java 開發不太一樣。Java 用 Eclipse 或其他 IDE，可以在 Windows、Linux、Mac 任何平台跑。但 iOS 開發被綁死在 macOS 和 Xcode 上，沒得選。好處是工具整合得很好，壞處是自由度低，而且 Mac 硬體要求讓開發成本變高。

Xcode 檔案超過 5GB，下載和安裝都要花不少時間。第一次啟動還會要求安裝額外元件。耐心等待吧。

Xcode 不只是編輯器，它是完整的開發環境：
- **IDE**：程式碼編輯、語法提示、重構工具
- **Interface Builder**：視覺化設計 UI，像 Java Swing 的 GUI Builder
- **iOS Simulator**：在 Mac 上模擬 iPhone/iPad，不用每次都裝到實機
- **Debugger**：LLDB 除錯器和 Instruments 效能分析工具
- **SDK**：iOS、macOS、watchOS、tvOS 的框架和 API

這種一站式環境對初學者友善，不用像 Java 那樣自己組合 JDK + IDE + 應用伺服器。但也代表你被 Apple 的生態系綁住，想用其他工具很困難。

安裝完成後，可以透過 Terminal 確認 Swift 版本：

```bash
swift --version
# Swift version 4.0.3 (swiftlang-900.0.74.1 clang-900.0.39.2)
```

### Playground 初體驗

Xcode 提供了 Playground 功能，這是一個即時編譯執行的環境，非常適合學習語法和測試程式碼片段。建立新的 Playground 專案，選擇 iOS 平台，就可以開始撰寫 Swift 程式碼。

Playground 的概念有點像 Java 9 的 JShell（JShell 在去年 9 月隨 Java 9 發布，不過 Swift Playground 早在 2014 就有了）。你寫一行程式碼，右側立刻顯示結果，不需要完整的編譯-執行循環。對學語法來說非常方便。

我建議：
- **學語法時用 Playground**：快速實驗，立即回饋
- **做完整 app 時建專案**：需要 UI、檔案管理、多個類別時

File > New > Playground > iOS > Blank，就能開始實驗了。

## Swift 基礎語法

### 變數與常數

Swift 使用 `var` 宣告變數，`let` 宣告常數：

```swift
var userName = "John"
let maxAttempts = 3

userName = "Jane"  // 可以修改
// maxAttempts = 5  // 編譯錯誤，常數不可修改
```

這點和 Java 的 `final` 關鍵字概念相似，但 Swift 更鼓勵使用不可變的值。

**為什麼強調不可變性？**

在 Java 中，我們也知道多用 `final` 是好習慣，但實務上很少這麼做，因為要多打字。Swift 的 `let` 只有 3 個字元，而且 Xcode 會主動建議「這個變數可以改用 let」，讓你養成好習慣。

不可變性的好處：
- **執行緒安全**：不會被其他執行緒意外修改
- **易於理解**：看到 let 就知道這個值不會變，減少認知負擔
- **效能優化**：編譯器知道值不變，可以做更多優化

這在多執行緒程式設計中特別重要。iOS 的 UI 必須在主執行緒更新，非同步程式碼很常見，不可變性能避免很多 bug。

### 型別推斷與型別標註

Swift 具有強大的型別推斷能力，編譯器會自動判斷變數型別：

```swift
var age = 25              // 推斷為 Int
var price = 99.99         // 推斷為 Double
var isActive = true       // 推斷為 Bool
```

也可以明確標註型別：

```swift
var count: Int = 0
var rate: Double = 0.0
var name: String = ""
```

Java 目前還沒有這種型別推斷功能（聽說 Java 10 會加入），Swift 從一開始就有。這讓程式碼更簡潔，又不失型別安全。

不過要注意，Swift 是**強型別語言**，一旦型別確定就不能改：

```swift
var value = 10    // Int
value = "Hello"   // 編譯錯誤！不能把 String 指派給 Int
```

這和 JavaScript 的 `var` 完全不同，JavaScript 可以隨意改型別。Swift 的型別推斷只是省略型別宣告，不是動態型別。

### 基本資料型別

Swift 提供的基本型別包括：

- **Int**：整數，有 Int8、Int16、Int32、Int64
- **Double、Float**：浮點數，預設推斷為 Double
- **Bool**：布林值
- **String**：字串
- **Character**：單一字元

與 Java 最大的不同：Swift 的所有型別都是**結構體（Struct）**，不是原始型別（primitive type）。

在 Java 中，`int` 是原始型別，`Integer` 是包裝類別。你要在兩者間轉換，還有自動裝箱/拆箱的陷阱。Swift 沒有這個問題，`Int` 既是值型別又有方法，統一又簡潔。

```swift
let number = 42
let doubled = number * 2         // Int 可以直接運算
let text = String(number)        // 轉成字串
```

這種統一性讓程式碼更容易理解，不用記住哪些是 primitive、哪些是 object。

### 字串操作

Swift 的字串處理比 Java 更直觀：

```swift
let firstName = "John"
let lastName = "Doe"
let fullName = firstName + " " + lastName

// 字串插值
let greeting = "Hello, \(fullName)!"
let message = "You have \(count) new messages"
```

字串插值的語法比 Java 的字串串接更簡潔易讀。

## Optional 型別

這是 Swift 最重要的特性之一，用來處理值可能不存在的情況：

```swift
var optionalName: String? = "John"
optionalName = nil  // 可以為 nil

var normalName: String = "Jane"
// normalName = nil  // 編譯錯誤
```

Optional 的概念類似 Java 8 的 Optional 類別，但在 Swift 中是語言層級的特性。

### Optional Binding

安全地解開 Optional 值：

```swift
if let name = optionalName {
    print("Name is \(name)")
} else {
    print("Name is nil")
}
```

### Forced Unwrapping

強制解開 Optional（需確保值存在）：

```swift
let name = optionalName!  // 如果 optionalName 為 nil 會造成執行時期錯誤
```

## 控制流程

### if-else 語句

```swift
let temperature = 25

if temperature > 30 {
    print("Hot")
} else if temperature > 20 {
    print("Warm")
} else {
    print("Cold")
}
```

注意 Swift 的條件式不需要括號，但大括號必須存在。

### Switch 語句

Swift 的 switch 比 Java 強大許多：

```swift
let status = 200

switch status {
case 200:
    print("Success")
case 400...499:
    print("Client Error")
case 500...599:
    print("Server Error")
default:
    print("Unknown")
}
```

Swift 的 switch 不需要 break，也不會自動往下執行。可以使用範圍匹配，這點比 Java 方便。

## 迴圈

### For-In 迴圈

```swift
for i in 1...5 {
    print("Number: \(i)")
}

let fruits = ["Apple", "Banana", "Orange"]
for fruit in fruits {
    print(fruit)
}
```

`...` 是閉區間運算子，包含結尾值。`..<` 是半開區間運算子，不包含結尾值。

### While 迴圈

```swift
var counter = 0
while counter < 5 {
    print(counter)
    counter += 1
}
```

### Repeat-While 迴圈

```swift
var number = 0
repeat {
    print(number)
    number += 1
} while number < 5
```

這相當於 Java 的 do-while。

## 與 Java 的比較

作為 Java 工程師，以下是我觀察到的主要差異：

1. **型別安全**：Swift 更強調型別安全，Optional 機制強制處理 null 情況
2. **語法簡潔**：不需要分號、條件式不需要括號
3. **型別推斷**：減少冗餘的型別宣告
4. **值型別**：基本型別都是結構體，而非原始型別
5. **不可變性**：更鼓勵使用 let 宣告不可變值

## 小結

第一週主要專注在基礎語法的學習，Swift 的語法設計相當現代化，許多概念對 Java 工程師來說並不陌生。Optional 型別是需要特別注意的重點，這是 Swift 用來避免 null pointer 問題的核心機制。

下週計劃深入學習函式、閉包以及集合型別的使用。
