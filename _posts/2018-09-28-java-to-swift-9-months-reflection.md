---
layout: post
title: "從 Java 到 Swift：9 個月的轉換心得"
date: 2018-09-28
categories: [iOS, Swift, Career]
tags: [Swift, Java, Learning, Reflection]
---

今天是 2018 年 9 月的最後一週，距離公司交辦學習 iOS 開發已經過了 9 個月。回顧這段旅程，從完全不懂 Swift 和 iOS 框架，到現在能獨立開發並上架 app，有很多心得想記錄下來。

## 語言層面的差異

### Swift vs Java：核心概念對比

作為一個寫了多年 Java 的工程師，轉到 Swift 最大的挑戰不是語法，而是**思維模式的轉換**。

**Optional 的強制性**

Java 的 null 是隱藏的地雷，任何參照都可能是 null，要自己記得檢查。Swift 的 Optional 把「可能沒有值」這件事**明確化**在型別系統中：

```swift
// Swift 強迫你處理 nil 的情況
var name: String? = nil
if let unwrappedName = name {
    print(unwrappedName)
} else {
    print("name 是 nil")
}
```

一開始我覺得 Optional 很煩，到處都要解包。但用久了發現，這大幅減少了 NullPointerException 類型的錯誤。在 Java 專案中，我遇過太多次「突然噴 NPE」的情況，往往是因為某個深層方法回傳了 null。Swift 的 Optional 讓這種問題在編譯期就被抓出來。

**值型別 vs 參照型別**

Java 除了基本型別，其他都是參照。Swift 則大量使用值型別：

```swift
struct Point {  // struct 是值型別
    var x: Int
    var y: Int
}

var p1 = Point(x: 10, y: 20)
var p2 = p1  // 複製，不是參照
p2.x = 30
print(p1.x)  // 10，p1 不受影響
```

這個設計讓多執行緒程式更安全，因為值型別不會有共享狀態的問題。但一開始我常搞混 struct 和 class 的行為，花時間才適應。

**Protocol-Oriented Programming**

Java 的介面（interface）主要用於抽象化。Swift 的 protocol 則更強大，配合 extension 能實現**帶有預設實作的介面**：

```swift
protocol Drivable {
    func drive()
}

extension Drivable {
    func drive() {
        print("預設的駕駛實作")
    }
}

struct Car: Drivable {}
let car = Car()
car.drive()  // 使用預設實作
```

這讓程式碼重用更靈活，不需要抽象類別就能共享實作。這是 Java 8 的 default method 概念，但 Swift 更徹底地實踐了這個理念。

### 記憶體管理的轉變

Java 的 GC（Garbage Collection）是背景自動執行，開發者幾乎不用管記憶體。Swift 的 ARC（Automatic Reference Counting）則需要理解參照計數：

```swift
class Person {
    var name: String
    weak var friend: Person?  // 用 weak 避免循環參照
    
    init(name: String) {
        self.name = name
    }
}
```

一開始我不理解為什麼需要 `weak` 和 `unowned`，直到遇到第一個記憶體洩漏。原來閉包捕獲 self 會建立強參照，造成循環：

```swift
class ViewController: UIViewController {
    var closure: (() -> Void)?
    
    func setupClosure() {
        closure = { [weak self] in  // 必須用 weak
            self?.view.backgroundColor = .red
        }
    }
}
```

這個概念在 Java 中不存在，因為 GC 能處理循環參照。但在 Swift 中，循環參照會導致物件永遠不被釋放。

## 框架生態的學習曲線

### UIKit 的陡峭學習曲線

從 Java 的 Swing 或 Android 轉到 UIKit，最大的挑戰是 **Auto Layout 和生命週期管理**。

**Auto Layout**

Java Swing 的佈局管理器（BorderLayout、GridLayout）相對簡單。UIKit 的 Auto Layout 用約束（Constraints）表達佈局關係，一開始很難理解：

```swift
NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: view.centerYAnchor),
    label.widthAnchor.constraint(equalToConstant: 200)
])
```

花了好幾週才掌握約束的優先權、內容壓縮阻力（Content Hugging）、內容擁抱優先權（Compression Resistance）等概念。但一旦理解，Auto Layout 在處理不同螢幕尺寸時比傳統佈局管理器強大太多。

**生命週期**

iOS 的 ViewController 生命週期比 Java Servlet 的生命週期複雜：

```
init → loadView → viewDidLoad → viewWillAppear → 
viewDidAppear → viewWillDisappear → viewDidDisappear → deinit
```

每個階段該做什麼事，一開始常搞錯。比如在 `viewDidLoad` 設定 Auto Layout 約束，卻在 `viewWillAppear` 才能正確取得 view 的 frame。這些細節需要實戰經驗累積。

### Delegate Pattern 無所不在

Java 的事件處理通常用 Listener 介面。iOS 則大量使用 Delegate 模式：

```swift
class MyTableViewController: UITableViewController, UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        // 處理選擇
    }
}
```

一開始我覺得為什麼不用閉包或回呼更簡單？後來發現 Delegate 的好處是**方法很多時能集中管理**。UITableView 的 delegate 有幾十個方法，如果都用閉包會很亂。

而且 Delegate 通常用 `weak` 參照避免循環，這是設計模式層面對記憶體管理的考量。

## 開發工具的適應

### Xcode vs Eclipse

從 Eclipse 轉到 Xcode，有些功能讓我懷念 IDE 的熟悉感：

**Eclipse 更強的地方**：
- 重構工具（Extract Method、Rename 等）更成熟
- 自動完成功能配合習慣
- Maven 專案管理很熟悉
- SVN/Git 整合用得順手

**Xcode 更強的地方**：
- Interface Builder 視覺化設計 UI
- Instruments 效能分析工具
- Simulator 模擬器整合完美
- StoryBoard 管理畫面流程

整體來說，Xcode 在視覺化工具上勝出，但在純程式碼編輯上需要適應。好在 Xcode 的程式碼補全和導航功能也不差，習慣後也很順手。

### 除錯體驗

Xcode 的 LLDB debugger 功能強大，但一開始不習慣命令列介面。Eclipse 的圖形化 debugger 用了多年，轉換需要時間適應。

但 Instruments 是 Xcode 的殺手級工具，能深入分析記憶體、CPU、網路等。這是 Java 開發中用 JProfiler 或 VisualVM 的體驗，但整合得更緊密。

## 架構思維的轉變

### 從分層架構到 MVVM

Java 企業開發習慣分層架構：

```
Controller → Service → DAO → Database
```

每層職責清楚，依賴方向單向。iOS 開發則更強調 **View 和 Model 的分離**，MVVM 是主流：

```
View ← ViewModel ← Model
```

ViewModel 不依賴 UIKit，可以純粹用 Swift 實作，這大幅提升可測試性。一開始我把 ViewModel 當成 Java 的 Service 層，但實際上 ViewModel 更關注「View 需要什麼資料和行為」，而不是「業務邏輯是什麼」。

這個思維轉換花了我幾個月。現在寫 ViewModel 時，我會先思考「這個畫面要顯示什麼」，再決定 ViewModel 提供什麼屬性和方法。

### 非同步程式設計

Java 的非同步通常用 CompletableFuture 或 RxJava。iOS 則有：

- **Grand Central Dispatch (GCD)**：低階的執行緒管理
- **OperationQueue**：高階的任務佇列
- **RxSwift**：第三方響應式程式設計框架

我最常用 GCD，語法簡潔：

```swift
DispatchQueue.global().async {
    // 背景執行
    let result = heavyComputation()
    
    DispatchQueue.main.async {
        // 回主執行緒更新 UI
        self.updateUI(result)
    }
}
```

但要注意 GCD 的陷阱：主執行緒同步呼叫會死鎖、強參照 self 會造成循環。這些都是實戰中踩過的坑。

## 測試文化的差異

Java 社群對測試的重視程度很高，TDD（Test-Driven Development）是常見實踐。iOS 社群則相對不那麼重視測試。

我發現很多 iOS 開源專案沒有測試，或測試覆蓋率很低。這可能因為：
1. UI 測試比單元測試難寫
2. 早期 Objective-C 的測試工具不夠好
3. 行動 app 迭代快，常常「先上線再說」

但我堅持在 iOS 專案中也寫測試，特別是 ViewModel 和 Service 層。這讓我在重構時更有信心，也幫助抓到很多潛在 bug。

## 生態系與社群

### 套件管理

Java 有 Maven 和 Gradle，生態系成熟。iOS 則有：

- **CocoaPods**：最老牌，但慢且笨重
- **Carthage**：輕量但需要手動整合
- **Swift Package Manager (SPM)**：Apple 官方，越來越主流

我現在優先用 SPM，簡單且整合良好。但一些老套件只支援 CocoaPods，不得不混用。

### 學習資源

Java 的學習資源豐富，Stack Overflow 上幾乎任何問題都能找到答案。iOS 的資源相對少，而且 Objective-C 和 Swift 混雜，搜尋結果常常過時。

Apple 的官方文件品質很高，但有時太簡略，缺少實戰範例。WWDC 影片是很好的學習資源，但要花大量時間觀看。

## 最大的收穫

這 9 個月最大的收穫不是技術本身，而是**跳出舒適圈的勇氣和能力**。

作為資深 Java 工程師，我在 Java 生態系中很comfortable。轉到 iOS 等於從零開始，很多時候像個新手一樣到處碰壁。但這個過程讓我：

**重新體驗學習的感覺**：每天都在學新東西，每個問題都是挑戰。這種成長感是做熟悉的技術時少有的。

**更理解框架設計**：看過 Spring、Hibernate 的設計後，再看 UIKit、GCD，更能理解不同框架的設計取捨。比如 Spring 的 IoC 和 iOS 的 Delegate，本質都在解決依賴管理問題，只是實作方式不同。

**提升問題解決能力**：遇到不懂的東西，不是直接找解答，而是先理解原理。比如遇到 Auto Layout 問題，我會先理解約束系統的運作機制，而不是亂試組合。

**欣賞不同生態系的優點**：Java 的企業級成熟度、iOS 的使用者體驗導向、Web 的開放生態，各有所長。沒有完美的技術棧，只有最適合特定場景的選擇。

## 給 Java 工程師的建議

如果你也是 Java 背景想學 iOS，我的建議是：

**不要急著寫程式碼**：先花時間理解 iOS 的設計哲學、生命週期、記憶體模型。這些基礎不穩，後面會一直踩坑。

**從小專案開始**：不要一開始就做複雜的 app。從待辦事項、計算機這類簡單 app 練習基本功。

**善用系統框架**：不要重新發明輪子。Apple 提供的框架很完整，大部分需求都有對應方案。

**注重 UI/UX**：iOS 使用者對介面品質要求很高。學習 Human Interface Guidelines，理解什麼是好的行動 app 體驗。

**寫測試**：即使 iOS 社群不那麼重視測試，但測試能讓你的程式碼更穩健。尤其是 ViewModel 層，沒藉口不寫測試。

**參與社群**：加入 iOS 相關的論壇、Discord、Meetup。看別人的程式碼，分享自己的經驗。

## 未來的方向

9 個月過去了，我能開發基本的 iOS app，但還有很多要學：

**RxSwift 深入**：響應式程式設計能讓非同步程式碼更優雅。

**Core Data 進階**：複雜的資料模型和查詢優化。

**ARKit**：擴增實境的應用探索。

**App Extensions**：Today Widget、Share Extension 等擴展功能。

**Metal**：高效能圖形渲染，遊戲開發會用到。

**跨平台方案**：Flutter、React Native 的了解，評估何時適合跨平台。

但最重要的是**保持學習的熱情**。技術會變，框架會更新，但學習的能力和解決問題的思維是永恆的。

## 結語

從 Java 到 Swift，從後端到行動端，這 9 個月是職涯中重要的轉折點。

我學到的不只是 Swift 語言或 iOS 框架，更是**如何快速學習新技術、如何適應不同的開發文化、如何保持開放的心態**。

技術人的職涯很長，不會永遠只用一種技術。保持學習、擁抱變化、享受挑戰，才是長久之道。

感謝這 9 個月的每一次挫折和突破。繼續前進，繼續學習。
