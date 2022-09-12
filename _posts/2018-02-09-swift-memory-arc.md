---
layout: post
title: "Swift 記憶體管理與 ARC"
date: 2018-02-09
categories: [Swift, iOS]
tags: [Swift, ARC, Memory Management]
---

這週深入研究 Swift 的記憶體管理機制。這對 Java 工程師來說是個需要轉換思維的主題。

Java 用的是 **Garbage Collection (GC)**：程式執行時，GC 會定期掃描記憶體，找出沒人用的物件並回收。優點是開發者不用管記憶體，缺點是 GC 執行時會造成停頓（Stop-the-World）。

Swift 用的是 **Automatic Reference Counting (ARC)**：編譯器會在適當的地方插入 retain/release 程式碼，當物件的參考計數降為 0 時立即釋放。優點是效能好、可預測，缺點是要小心 retain cycle（循環參考）。

一開始我以為 ARC 和 GC 差不多，但實際使用後發現差異很大。ARC 需要開發者理解 strong、weak、unowned 這些概念，否則很容易記憶體洩漏。

## 自動參考計數 (ARC)

Swift 使用 ARC (Automatic Reference Counting) 管理記憶體。雖然都是「自動」，但和 Java 的 GC 運作機制完全不同。

### ARC 基本原理

```swift
class Person {
    let name: String
    
    init(name: String) {
        self.name = name
        print("\(name) is being initialized")
    }
    
    deinit {
        print("\(name) is being deinitialized")
    }
}

var person1: Person?
var person2: Person?
var person3: Person?

person1 = Person(name: "John")
// John is being initialized

person2 = person1  // 參考計數 +1
person3 = person1  // 參考計數 +1

person1 = nil  // 參考計數 -1
person2 = nil  // 參考計數 -1
person3 = nil  // 參考計數 -1，觸發 deinit
// John is being deinitialized
```

當物件的參考計數降為 0 時，記憶體立即被釋放。

## 強參考循環 (Strong Reference Cycle)

### 問題示範

```swift
class Person {
    let name: String
    var apartment: Apartment?
    
    init(name: String) {
        self.name = name
        print("\(name) is initialized")
    }
    
    deinit {
        print("\(name) is deinitialized")
    }
}

class Apartment {
    let unit: String
    var tenant: Person?
    
    init(unit: String) {
        self.unit = unit
        print("Apartment \(unit) is initialized")
    }
    
    deinit {
        print("Apartment \(unit) is deinitialized")
    }
}

var john: Person? = Person(name: "John")
var unit4A: Apartment? = Apartment(unit: "4A")

john?.apartment = unit4A
unit4A?.tenant = john

john = nil
unit4A = nil
// deinit 都不會被呼叫，發生記憶體洩漏
```

Person 和 Apartment 互相持有強參考，形成循環參考。

## 弱參考 (Weak Reference)

### 使用 weak 解決循環參考

```swift
class Person {
    let name: String
    var apartment: Apartment?
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) is deinitialized")
    }
}

class Apartment {
    let unit: String
    weak var tenant: Person?  // 使用 weak
    
    init(unit: String) {
        self.unit = unit
    }
    
    deinit {
        print("Apartment \(unit) is deinitialized")
    }
}

var john: Person? = Person(name: "John")
var unit4A: Apartment? = Apartment(unit: "4A")

john?.apartment = unit4A
unit4A?.tenant = john

john = nil
// John is deinitialized
// tenant 自動變為 nil

unit4A = nil
// Apartment 4A is deinitialized
```

`weak` 參考不增加參考計數，當物件被釋放時自動變為 nil。

### weak 的特性

```swift
class Node {
    let value: Int
    weak var next: Node?
    
    init(value: Int) {
        self.value = value
    }
    
    deinit {
        print("Node \(value) is deinitialized")
    }
}

var node1: Node? = Node(value: 1)
var node2: Node? = Node(value: 2)

node1?.next = node2
// node2 的參考計數仍然是 1

node2 = nil
// Node 2 is deinitialized
// node1?.next 自動變為 nil
```

weak 參考必須是 Optional，因為可能變為 nil。

## 無主參考 (Unowned Reference)

### 使用 unowned

```swift
class Customer {
    let name: String
    var card: CreditCard?
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) is deinitialized")
    }
}

class CreditCard {
    let number: UInt64
    unowned let customer: Customer  // 使用 unowned
    
    init(number: UInt64, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    
    deinit {
        print("Card #\(number) is deinitialized")
    }
}

var john: Customer? = Customer(name: "John")
john?.card = CreditCard(number: 1234_5678_9012_3456, customer: john!)

john = nil
// John is deinitialized
// Card #1234567890123456 is deinitialized
```

`unowned` 用於當另一個實例的生命週期相同或更長時。

### weak vs unowned

```swift
// weak：可能為 nil，必須是 Optional
class A {
    weak var b: B?
}

// unowned：假設永遠不為 nil，非 Optional
class B {
    unowned let a: A
    
    init(a: A) {
        self.a = a
    }
}
```

選擇原則：
- 使用 `weak`：參考可能在某個時間點變為 nil
- 使用 `unowned`：參考在初始化後永遠有值

### 不安全的 unowned

```swift
class Country {
    let name: String
    var capital: City!  // 隱式解包 Optional
    
    init(name: String, capitalName: String) {
        self.name = name
        self.capital = City(name: capitalName, country: self)
    }
    
    deinit {
        print("\(name) is deinitialized")
    }
}

class City {
    let name: String
    unowned let country: Country
    
    init(name: String, country: Country) {
        self.name = name
        self.country = country
    }
    
    deinit {
        print("\(name) is deinitialized")
    }
}

var country: Country? = Country(name: "Taiwan", capitalName: "Taipei")
country = nil
// Taiwan is deinitialized
// Taipei is deinitialized
```

這種模式允許在初始化時建立雙向關聯。

## 閉包的循環參考

### 問題示範

```swift
class HTMLElement {
    let name: String
    let text: String?
    
    lazy var asHTML: () -> String = {
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }
    
    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }
    
    deinit {
        print("\(name) is deinitialized")
    }
}

var paragraph: HTMLElement? = HTMLElement(name: "p", text: "Hello")
print(paragraph!.asHTML())
// <p>Hello</p>

paragraph = nil
// deinit 不會被呼叫，發生記憶體洩漏
```

閉包捕獲 `self`，形成強參考循環。

### 捕獲列表 (Capture List)

```swift
class HTMLElement {
    let name: String
    let text: String?
    
    lazy var asHTML: () -> String = { [weak self] in
        guard let self = self else { return "" }
        
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }
    
    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }
    
    deinit {
        print("\(name) is deinitialized")
    }
}

var paragraph: HTMLElement? = HTMLElement(name: "p", text: "Hello")
print(paragraph!.asHTML())

paragraph = nil
// p is deinitialized
```

使用 `[weak self]` 或 `[unowned self]` 打破循環參考。

### 捕獲特定值

```swift
class Manager {
    var count = 0
    
    lazy var increment: () -> Void = { [unowned self] in
        self.count += 1
    }
    
    lazy var printCount: () -> Void = { [count] in
        print("Captured count: \(count)")  // 捕獲值，不是參考
    }
    
    deinit {
        print("Manager is deinitialized")
    }
}

var manager: Manager? = Manager()
manager?.increment()
manager?.increment()
print(manager!.count)  // 2

manager?.printCount()  // Captured count: 0

manager = nil
// Manager is deinitialized
```

捕獲列表可以捕獲值的副本，而非參考。

## 常見的記憶體管理場景

### Delegate 模式

```swift
protocol DataSourceDelegate: AnyObject {
    func dataDidChange()
}

class DataSource {
    weak var delegate: DataSourceDelegate?
    
    func updateData() {
        // 更新資料
        delegate?.dataDidChange()
    }
}

class ViewController: DataSourceDelegate {
    let dataSource = DataSource()
    
    init() {
        dataSource.delegate = self
    }
    
    func dataDidChange() {
        print("Data changed")
    }
    
    deinit {
        print("ViewController is deinitialized")
    }
}
```

Delegate 通常使用 `weak` 避免循環參考。

### 通知觀察者

```swift
class Observer {
    init() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleNotification),
            name: .dataUpdated,
            object: nil
        )
    }
    
    @objc func handleNotification() {
        print("Notification received")
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
        print("Observer is deinitialized")
    }
}

extension Notification.Name {
    static let dataUpdated = Notification.Name("dataUpdated")
}
```

必須在 deinit 中移除觀察者，否則可能造成問題。

### Timer 與循環參考

```swift
class TimerManager {
    var timer: Timer?
    var count = 0
    
    func startTimer() {
        // 錯誤：會造成循環參考
        // timer = Timer.scheduledTimer(timeInterval: 1.0, target: self, selector: #selector(tick), repeats: true)
        
        // 正確：使用閉包版本
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
            self?.tick()
        }
    }
    
    @objc func tick() {
        count += 1
        print("Tick: \(count)")
    }
    
    func stopTimer() {
        timer?.invalidate()
        timer = nil
    }
    
    deinit {
        stopTimer()
        print("TimerManager is deinitialized")
    }
}
```

Timer 會強參考 target，必須適當處理。

## 偵測記憶體洩漏

Xcode 提供 Instruments 工具檢測記憶體問題：

1. Product -> Profile 啟動 Instruments
2. 選擇 Leaks 模板
3. 執行應用程式並觀察記憶體洩漏
4. 使用 Allocations 追蹤物件分配

## 與 Java 的比較

1. **GC vs ARC**：Java 使用垃圾回收，Swift 使用參考計數
2. **決定性**：ARC 在參考計數歸零時立即釋放，GC 時間不確定
3. **循環參考**：Java GC 可以處理循環參考，Swift 需要手動使用 weak/unowned
4. **效能**：ARC 沒有 GC 的暫停時間，但需要維護參考計數
5. **開發複雜度**：Swift 需要更注意記憶體管理

## 小結

理解 ARC 是 Swift 開發的重要基礎。相較於 Java 的 GC，Swift 的 ARC 要求開發者更主動管理記憶體。使用 weak 和 unowned 打破循環參考是常見的模式，特別是在 delegate、閉包和 timer 等場景。

下週將開始學習 UIKit 框架，開始實際的 iOS UI 開發。
