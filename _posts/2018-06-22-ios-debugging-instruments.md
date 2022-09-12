---
layout: post
title: "iOS 除錯工具與 Instruments"
date: 2018-06-22
categories: [iOS, Swift]
tags: [Swift, iOS, Debugging, Instruments, Profiling, Performance]
---

這週研究 Xcode 的除錯工具。在 Java 開發中，我習慣用 Eclipse 的 debugger 和 JProfiler 做效能分析。iOS 的 Xcode 也提供了強大的工具集，特別是 **Instruments**，能深入分析記憶體、CPU、網路等各種效能指標。

## 基本除錯技巧

### 中斷點（Breakpoints）

最基本的除錯工具就是中斷點。在程式碼行號左側點擊就能設定：

```swift
func processData(_ data: [Int]) -> Int {
    var sum = 0
    for value in data {  // 在這裡設定中斷點
        sum += value
    }
    return sum
}
```

程式執行到中斷點會暫停，此時可以：
- 查看變數值
- 執行表達式
- 單步執行
- 修改變數值繼續執行

這和 Java debugger 的功能幾乎一樣，但 Xcode 的介面更直觀。

### 條件中斷點

有時只想在特定情況下暫停，可以設定條件：

右鍵點擊中斷點 → Edit Breakpoint → 加入 Condition：

```swift
value > 100  // 只有當 value 大於 100 時才暫停
```

這在迴圈中特別有用。比如陣列有 1000 個元素，但只關心某個特定值，就不用手動跳過 999 次。

### 符號中斷點（Symbolic Breakpoint）

可以設定「某個方法被呼叫時暫停」，不需要知道確切的程式碼位置：

Debug → Breakpoints → Create Symbolic Breakpoint

設定 Symbol 為：`-[UIViewController viewDidLoad]`

這會在任何 UIViewController 的 `viewDidLoad` 被呼叫時暫停。對於追蹤生命週期或第三方函式庫很有用。

### Exception Breakpoint

設定「任何例外發生時暫停」：

Debug → Breakpoints → Create Exception Breakpoint

這會在 app 崩潰前暫停，能看到完整的呼叫堆疊。我遇過很多次「app 突然閃退但不知道哪裡出錯」，設定這個中斷點後立刻找到問題。

## LLDB 除錯器指令

Xcode 底層使用 **LLDB** 除錯器，在暫停時可以在 console 輸入指令：

```lldb
# 印出變數值
(lldb) po myVariable
(lldb) p myVariable

# 印出物件的詳細資訊
(lldb) po self

# 執行 Swift 程式碼
(lldb) expr let result = 2 + 2
(lldb) expr print(result)

# 修改變數值
(lldb) expr myVariable = 100

# 查看記憶體位址
(lldb) p &myVariable

# 顯示視圖層級
(lldb) po view.recursiveDescription()
```

`po` 是 "print object" 的縮寫，能印出物件的描述。`p` 則是印出原始值。

我特別喜歡 `expr` 指令，能在暫停狀態下執行任意程式碼，測試假設而不用重新編譯。這在 Java debugger 中也有類似功能，叫做 "Evaluate Expression"。

### 客製化物件的除錯輸出

預設的物件描述常常不夠清楚，可以實作 `CustomDebugStringConvertible`：

```swift
struct User: CustomDebugStringConvertible {
    let id: Int
    let name: String
    let email: String
    
    var debugDescription: String {
        return "User(id: \(id), name: \(name), email: \(email))"
    }
}
```

這樣用 `po` 印出時會顯示客製化的格式，而不是記憶體位址。

## View Debugger

Xcode 的 View Debugger 能**視覺化 UI 層級結構**：

執行 app 後點擊 Debug View Hierarchy 按鈕（或 Cmd+Shift+D）

這會產生 3D 的 view 層級圖，可以：
- 旋轉查看 view 的疊加關係
- 選擇某個 view 查看屬性
- 找出被遮擋或超出邊界的 view

我遇過一個 bug：按鈕明明加到畫面上了卻點不到。用 View Debugger 發現按鈕被另一個透明的 view 擋住了。這種問題用傳統 debug 很難找，但視覺化工具一眼就看出來。

在 View Debugger 中還能在 console 直接操作 view：

```lldb
# 改變背景顏色
(lldb) expr self.view.backgroundColor = UIColor.red

# 隱藏某個 view
(lldb) expr someView.isHidden = true

# 這些改變會立即反映在畫面上，不用重新執行 app
```

## Instruments：深度效能分析

**Instruments** 是 Xcode 附帶的效能分析工具，功能極其強大。它能追蹤 app 的各種運行時指標。

開啟方式：Xcode → Product → Profile（Cmd+I）

### Time Profiler：找出 CPU 瓶頸

Time Profiler 顯示每個方法佔用的 CPU 時間：

執行 app 並操作一段時間後，停止錄製。Instruments 會顯示：
- 哪些方法佔用最多 CPU
- 呼叫堆疊
- 每個執行緒的狀況

我用 Time Profiler 發現一個效能問題：主執行緒花了 80% 時間在一個圖片解碼方法上。原來是我在主執行緒同步載入大圖片，改成背景執行緒後 UI 變流暢了。

**解讀技巧**：
- 關注「Self」欄位高的方法，代表該方法本身（不含子方法）耗時多
- 雙擊方法名可以跳到對應的程式碼
- 用 Call Tree 選項過濾系統框架，只看自己的程式碼

### Allocations：追蹤記憶體使用

Allocations 工具顯示 app 配置了多少記憶體：

```swift
class ImageCache {
    private var cache: [String: UIImage] = [:]
    
    func store(_ image: UIImage, forKey key: String) {
        cache[key] = image
    }
}
```

如果這個 cache 沒有大小限制，記憶體會不斷增長。Allocations 能清楚看到記憶體使用趨勢：
- 如果記憶體持續上升不下降，可能有記憶體洩漏
- 如果某個操作後記憶體暴增，可能該操作有問題

點擊「Mark Generation」按鈕，執行某個操作，再點一次。Instruments 會顯示這段時間內配置的物件，幫助找出問題來源。

### Leaks：找出記憶體洩漏

Leaks 工具自動偵測記憶體洩漏。雖然 Swift 有 ARC 自動管理記憶體，但**循環參照**仍會造成洩漏：

```swift
class ViewController: UIViewController {
    var onComplete: (() -> Void)?
    
    func setupCallback() {
        onComplete = {
            self.dismiss(animated: true)  // 強參照 self
        }
    }
}
```

這個閉包捕獲了 `self`，而 `self` 又持有 `onComplete`，形成循環。正確的做法是用 `[weak self]`：

```swift
onComplete = { [weak self] in
    self?.dismiss(animated: true)
}
```

Leaks 工具會標示出洩漏的物件，並顯示保留週期（Retain Cycle）。這比手動找記憶體洩漏有效太多。

在 Java 中我用 VisualVM 或 JProfiler 找記憶體洩漏，概念類似但 Swift 的 ARC 讓問題更集中在循環參照上。

### Network：監控網路活動

Network 工具顯示所有網路請求：
- URL 和狀態碼
- 請求和回應的大小
- 耗時
- HTTP headers

我發現一個 app 載入很慢，用 Network 工具看到某個 API 回傳了 10MB 的 JSON。原來後端沒做分頁，一次回傳所有資料。改成分頁請求後體驗好很多。

### Core Animation：分析 UI 效能

Core Animation 工具幫助找出 UI 卡頓的原因：

**Color Blended Layers**：標示需要混合的圖層（紅色表示需要優化）
**Color Offscreen-Rendered**：標示離屏渲染的 view（黃色表示有效能損耗）
**Color Misaligned Images**：標示像素不對齊的圖片（藍色表示需要調整）

我曾經有個 UITableView 滑動時卡頓，用這個工具發現所有 cell 都在做離屏渲染（因為設定了圓角和陰影）。優化後滑動變得非常流暢。

優化技巧：
- 避免在圖層上同時設定圓角和遮罩
- 使用不透明背景色
- 圖片大小應該匹配顯示大小，避免縮放

## Memory Graph Debugger

記憶體圖除錯器能視覺化物件間的參照關係：

執行 app 後點擊 Debug Memory Graph 按鈕

這會顯示所有物件及其參照關係。選擇某個物件可以看到：
- 誰持有它（Incoming references）
- 它持有誰（Outgoing references）

這對找循環參照特別有用。如果看到 A → B → A 的參照鏈，就知道有循環了。

我用這個工具發現過一個隱藏的循環：ViewController → Delegate → Closure → ViewController。單看程式碼不容易發現，但視覺化後一目了然。

## 模擬網路狀況

開發時通常網路很快，但使用者可能在慢速網路或不穩定的連線下使用 app。Xcode 能模擬各種網路狀況：

Settings → Developer → Network Link Conditioner

可以模擬：
- 3G、LTE 等不同網路速度
- 高延遲
- 封包遺失

在這些條件下測試能發現很多問題：
- 圖片載入太久沒有 loading 指示器
- 請求超時沒有重試機制
- 網路錯誤沒有友善的錯誤訊息

## 模擬記憶體警告

iOS 在記憶體不足時會發送警告，app 應該釋放不必要的快取。可以在模擬器中手動觸發：

Debug → Simulate Memory Warning

這會呼叫 `didReceiveMemoryWarning()`，測試 app 的反應是否正確：

```swift
override func didReceiveMemoryWarning() {
    super.didReceiveMemoryWarning()
    
    // 清空圖片快取
    imageCache.removeAll()
    
    // 釋放不必要的資源
    print("記憶體警告：已清理快取")
}
```

## Crash Logs 分析

當 app 崩潰時，iOS 會產生 crash log。在 Xcode 中查看：

Window → Devices and Simulators → 選擇裝置 → View Device Logs

Crash log 包含：
- 崩潰時間和 app 版本
- 例外類型和原因
- 完整的呼叫堆疊
- 各執行緒的狀態

理解 crash log 的關鍵是**符號化（Symbolication）**。原始的 crash log 只有記憶體位址，需要用 dSYM 檔案轉換成可讀的方法名。

Xcode 通常會自動符號化，但如果是使用者回報的崩潰，可能需要手動處理。這在 Java 中類似於用 stack trace 定位問題，但 iOS 多了符號化這一步。

## 小結

這週研究除錯和效能分析工具，最大的體會是**工欲善其事必先利其器**。

很多效能問題和 bug 用肉眼看程式碼很難發現，但用對工具就能快速定位。特別是 Instruments，雖然學習曲線陡峭，但掌握後能大幅提升開發效率。

關鍵要點：
1. **善用中斷點**：條件中斷點、符號中斷點、例外中斷點
2. **LLDB 指令**：暫停時執行程式碼測試假設
3. **View Debugger**：視覺化 UI 層級找出佈局問題
4. **Instruments**：Time Profiler 找 CPU 瓶頸、Leaks 找記憶體洩漏
5. **模擬真實環境**：慢速網路、記憶體警告

下週計劃研究 App Store 上架流程，包括憑證管理、截圖準備、審查指南等。這是把 app 交到使用者手中的最後一哩路。
