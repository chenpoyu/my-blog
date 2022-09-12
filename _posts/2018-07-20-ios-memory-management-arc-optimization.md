---
layout: post
title: "iOS 記憶體管理深入：ARC 與效能優化"
date: 2018-07-20
categories: [iOS, Performance]
tags: [ARC, Memory, Performance, Optimization]
---

這週深入研究 iOS 的記憶體管理，發現 ARC（Automatic Reference Counting）雖然自動化，但還是需要理解原理才能避免問題。這和 Java 的 GC 有本質上的不同。

## ARC vs GC：運作機制的差異

**Java 的 GC（Garbage Collection）**：
- 執行時期的背景處理程序
- 定期掃描物件，找出不可達的物件並回收
- 有 Stop-The-World 的暫停時間
- 開發者幾乎不用管記憶體

**Swift 的 ARC（Automatic Reference Counting）**：
- 編譯時期插入 retain/release 程式碼
- 物件被參照時計數 +1，不被參照時 -1
- 計數變成 0 時立即釋放
- 需要注意循環參照問題

```swift
class Person {
    let name: String
    
    init(name: String) {
        self.name = name
        print("\(name) 被初始化")
    }
    
    deinit {
        print("\(name) 被釋放")
    }
}

var person1: Person? = Person(name: "Alice")  // 參照計數 = 1
var person2 = person1  // 參照計數 = 2
person1 = nil  // 參照計數 = 1
person2 = nil  // 參照計數 = 0，立即呼叫 deinit
```

**為什麼 iOS 用 ARC 而不用 GC？**

1. **即時性**：ARC 在物件不被參照時立即釋放，GC 需要等掃描週期
2. **可預測性**：ARC 的行為可預測，GC 的暫停時間不可控
3. **效能**：ARC 沒有 GC 的背景掃描成本
4. **電池壽命**：行動裝置重視電池，GC 會消耗額外能源

但代價是**開發者需要理解參照關係，避免循環參照**。

## Strong、Weak、Unowned：參照類型的選擇

**Strong（強參照）**：預設的參照類型，會增加計數

```swift
class Car {
    var name: String
    init(name: String) { self.name = name }
    deinit { print("\(name) 被釋放") }
}

var car1: Car? = Car(name: "Tesla")  // strong reference
var car2 = car1  // 兩個 strong reference
car1 = nil  // 還有 car2 參照，不會釋放
car2 = nil  // 沒有參照了，釋放
```

**Weak（弱參照）**：不增加計數，物件釋放時自動變成 nil

```swift
class Owner {
    var name: String
    init(name: String) { self.name = name }
    deinit { print("Owner \(name) 被釋放") }
}

class Car {
    var name: String
    weak var owner: Owner?  // weak reference
    
    init(name: String) {
        self.name = name
    }
    
    deinit { print("Car \(name) 被釋放") }
}

var owner: Owner? = Owner(name: "Alice")
var car = Car(name: "Tesla")
car.owner = owner  // weak reference 不增加計數

owner = nil  // Owner 被釋放，car.owner 自動變成 nil
print(car.owner)  // nil
```

**Unowned（無主參照）**：不增加計數，但假設物件永遠存在

```swift
class Customer {
    var name: String
    var card: CreditCard?
    
    init(name: String) {
        self.name = name
    }
    
    deinit { print("Customer \(name) 被釋放") }
}

class CreditCard {
    var number: String
    unowned let customer: Customer  // unowned reference
    
    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    
    deinit { print("Card \(number) 被釋放") }
}

var customer: Customer? = Customer(name: "Alice")
customer?.card = CreditCard(number: "1234", customer: customer!)

customer = nil  // Customer 和 Card 都被釋放
```

**何時用哪個？**

- **Strong**：預設選擇，擁有關係（parent 擁有 child）
- **Weak**：雙向關係且生命週期可能不同（delegate pattern）
- **Unowned**：一定同時存在或同時消失（CreditCard 不可能沒有 Customer）

**Java 開發者的陷阱**：在 Java 中不用考慮這些，所有參照都是 strong。轉到 Swift 需要主動思考參照關係。

## 循環參照：最常見的記憶體洩漏

**案例 1：雙向參照**

```swift
class Person {
    var name: String
    var apartment: Apartment?
    
    init(name: String) { self.name = name }
    deinit { print("Person \(name) 被釋放") }
}

class Apartment {
    var unit: String
    var tenant: Person?  // 這裡應該用 weak
    
    init(unit: String) { self.unit = unit }
    deinit { print("Apartment \(unit) 被釋放") }
}

var person: Person? = Person(name: "Alice")
var apartment: Apartment? = Apartment(unit: "A1")

person?.apartment = apartment  // Person -> Apartment (strong)
apartment?.tenant = person  // Apartment -> Person (strong)

person = nil  // Person 不會釋放，因為 apartment.tenant 還參照它
apartment = nil  // Apartment 不會釋放，因為 person.apartment 還參照它

// 記憶體洩漏！兩個物件都不會被釋放
```

**解決方法**：讓其中一個參照用 weak

```swift
class Apartment {
    var unit: String
    weak var tenant: Person?  // 改用 weak
    
    init(unit: String) { self.unit = unit }
    deinit { print("Apartment \(unit) 被釋放") }
}

// 現在 person = nil 時，Person 會被釋放
// apartment = nil 時，Apartment 也會被釋放
```

**案例 2：閉包捕獲 self**

這是最常遇到的循環參照：

```swift
class ViewController: UIViewController {
    var name = "ViewController"
    var closure: (() -> Void)?
    
    func setupClosure() {
        closure = {
            print(self.name)  // 閉包捕獲 self，形成循環參照
        }
    }
    
    deinit {
        print("\(name) 被釋放")
    }
}

var vc: ViewController? = ViewController()
vc?.setupClosure()
vc = nil  // ViewController 不會被釋放！
```

**原因**：
- ViewController 持有 closure（strong）
- closure 捕獲 self（strong）
- 形成循環：ViewController → closure → ViewController

**解決方法**：在閉包中用 `[weak self]` 或 `[unowned self]`

```swift
func setupClosure() {
    closure = { [weak self] in
        guard let self = self else { return }
        print(self.name)
    }
}

// 或用 unowned（確定 closure 執行時 self 一定存在）
closure = { [unowned self] in
    print(self.name)
}
```

**實務中的常見場景**：

```swift
// 網路請求
URLSession.shared.dataTask(with: url) { [weak self] data, response, error in
    guard let self = self else { return }
    self.handleResponse(data)
}

// 動畫
UIView.animate(withDuration: 0.3, animations: { [weak self] in
    self?.view.alpha = 0
})

// Timer
timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
    self?.updateTime()
}
```

## 效能優化：減少不必要的記憶體分配

### 1. 用值型別減少堆積分配

**Class（參照型別）**：分配在堆積（heap），有記憶體管理成本

```swift
class Point {
    var x: Double
    var y: Double
    init(x: Double, y: Double) {
        self.x = x
        self.y = y
    }
}

var points: [Point] = []
for i in 0..<1000 {
    points.append(Point(x: Double(i), y: Double(i)))  // 1000 次堆積分配
}
```

**Struct（值型別）**：分配在堆疊（stack），效能更好

```swift
struct Point {
    var x: Double
    var y: Double
}

var points: [Point] = []
for i in 0..<1000 {
    points.append(Point(x: Double(i), y: Double(i)))  // 在堆疊上分配，更快
}
```

**原則**：
- 小型的、不需要繼承的資料結構用 struct
- 需要參照語意、繼承、或很大的物件用 class

### 2. Copy-On-Write 優化

Swift 的陣列、字典等集合型別都有 Copy-On-Write 優化：

```swift
var array1 = [1, 2, 3, 4, 5]
var array2 = array1  // 不會複製，共享儲存

print(array1)  // 只讀取，還是共享儲存
array2.append(6)  // 寫入時才複製
```

自己的 struct 也可以實作 COW：

```swift
final class Storage {
    var data: [Int]
    init(data: [Int]) { self.data = data }
}

struct MyArray {
    private var storage: Storage
    
    init(data: [Int]) {
        storage = Storage(data: data)
    }
    
    var data: [Int] {
        get { storage.data }
        set {
            if !isKnownUniquelyReferenced(&storage) {
                storage = Storage(data: storage.data)  // 有多個參照時才複製
            }
            storage.data = newValue
        }
    }
}
```

### 3. Autoreleasepool：批次釋放記憶體

處理大量物件時，用 autoreleasepool 避免記憶體峰值：

```swift
// 沒有 autoreleasepool
for i in 0..<1_000_000 {
    let data = generateLargeData()  // 記憶體持續累積
    process(data)
}

// 有 autoreleasepool
for i in 0..<1_000_000 {
    autoreleasepool {
        let data = generateLargeData()
        process(data)
        // 離開 autoreleasepool 時釋放 data
    }
}
```

這類似 Java 的區域變數作用域，但 Swift 需要手動指定。

### 4. Lazy 初始化：延遲分配

不一定用到的屬性用 lazy 延遲初始化：

```swift
class DataManager {
    lazy var cache: [String: Data] = {
        print("初始化 cache")
        return [:]
    }()
    
    lazy var database: Database = {
        print("初始化 database")
        return Database()
    }()
}

let manager = DataManager()
// 現在不會初始化 cache 和 database

manager.cache["key"] = data  // 第一次存取時才初始化
```

## 用 Instruments 檢測記憶體問題

Xcode 的 Instruments 工具能檢測記憶體洩漏：

**Leaks Instrument**：檢測循環參照

1. 執行 app 並操作各個功能
2. Leaks 會自動檢測洩漏
3. 點擊洩漏項目查看 call stack
4. 找出循環參照的原因

**Allocations Instrument**：監控記憶體使用

1. 記錄 app 的記憶體分配
2. 查看哪些物件佔用最多記憶體
3. 用 Generations 功能檢測是否正確釋放

**Memory Graph**：視覺化物件關係

1. 在 Xcode debug bar 點擊記憶體圖示
2. 查看物件的參照關係
3. 找出循環參照的鏈條

## 實務經驗

**最常見的記憶體洩漏**：
1. 閉包捕獲 self 忘記用 weak
2. Delegate 忘記宣告為 weak
3. Timer 沒有 invalidate
4. NotificationCenter 沒有 removeObserver

**最佳實踐**：
- 所有閉包預設加 `[weak self]`，除非確定不會循環參照
- Delegate protocol 繼承 AnyObject 並宣告為 weak
- Timer 在 deinit 中 invalidate
- NotificationCenter 用 block-based API 自動管理

**與 Java 的對比**：
- Java 開發者習慣不管記憶體，容易在 iOS 踩坑
- 但理解 ARC 後，記憶體問題變得可預測
- Java 的 GC 暫停時間不可控，ARC 沒有這個問題

## 學習心得

從 Java 轉到 Swift，記憶體管理是最大的挑戰之一。一開始覺得很麻煩，為什麼不像 Java 那樣自動處理？

但用久了發現，ARC 的好處是**可預測性**。知道物件何時被釋放，不用擔心 GC 突然暫停。而且一旦理解參照關係，記憶體問題變得很好 debug。

重點是養成習慣：
- 寫閉包就想到 `[weak self]`
- 寫 delegate 就宣告 weak
- 用 Instruments 定期檢查記憶體

記憶體管理是 iOS 開發的基本功，值得花時間深入理解。
