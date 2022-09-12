---
layout: post
title: "Swift 類別與結構"
date: 2018-01-19
categories: [Swift, iOS]
tags: [Swift, OOP, Class, Struct]
---

這週學習 Swift 的類別和結構。對 Java 工程師來說，類別的概念很熟悉，但 Swift 的結構（Struct）是個新東西。

Java 只有類別（Class），所有自訂型別都是類別。Swift 提供了類別和結構兩種選擇，而且**結構是值型別、類別是參考型別**，這個差異非常重要。

一開始我不太理解為什麼需要兩種，但深入學習後發現，這個設計讓程式碼更安全、效能更好。

## 類別與結構的定義

### 基本語法

Swift 同時支援類別和結構：

```swift
class Person {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
    func introduce() {
        print("I'm \(name), \(age) years old")
    }
}

struct Point {
    var x: Double
    var y: Double
    
    func distance() -> Double {
        return (x * x + y * y).squareRoot()
    }
}
```

### 實例建立

```swift
let person = Person(name: "John", age: 30)
person.introduce()

var point = Point(x: 3.0, y: 4.0)
print(point.distance())  // 5.0
```

結構自動獲得 memberwise initializer，類別需要自行定義初始化方法。

## 類別 vs 結構

這是 Swift 與 Java 最大的差異之一。理解這個差異對寫好 Swift 程式碼至關重要。

### 值型別 vs 參考型別

**結構是值型別**：複製時會產生完全獨立的副本  
**類別是參考型別**：複製時只是複製參考，指向同一個實例

在 Java 中，所有物件都是參考型別（除了 int、boolean 等 primitive types）。Swift 的設計更細緻：

```swift
struct PointStruct {
    var x: Int
    var y: Int
}

class PointClass {
    var x: Int
    var y: Int
    
    init(x: Int, y: Int) {
        self.x = x
        self.y = y
    }
}

// 結構 - 值型別
var point1 = PointStruct(x: 10, y: 20)
var point2 = point1
point2.x = 30
print(point1.x)  // 10 - 不受影響
print(point2.x)  // 30

// 類別 - 參考型別
let object1 = PointClass(x: 10, y: 20)
let object2 = object1
object2.x = 30
print(object1.x)  // 30 - 受影響
print(object2.x)  // 30
```

Java 的基本型別是值型別，物件都是參考型別。Swift 的結構和列舉都是值型別。

### 何時使用結構

Apple 建議優先使用結構，除非需要以下特性：

1. 需要繼承
2. 需要執行時期型別檢查
3. 需要 Deinitializer
4. 需要多個地方參考同一實例

常見的值型別場景：
- 幾何形狀（座標、尺寸）
- 範圍（起點、終點）
- 數學向量
- 簡單資料容器

## 屬性 (Properties)

### 儲存屬性 (Stored Properties)

```swift
struct Rectangle {
    var width: Double
    var height: Double
    let maxSize: Double = 100.0  // 常數屬性
}

var rect = Rectangle(width: 10, height: 20)
rect.width = 15  // 可以修改
// rect.maxSize = 200  // 編譯錯誤
```

### 計算屬性 (Computed Properties)

```swift
struct Rectangle {
    var width: Double
    var height: Double
    
    var area: Double {
        return width * height
    }
    
    var perimeter: Double {
        get {
            return 2 * (width + height)
        }
    }
}

let rect = Rectangle(width: 10, height: 20)
print(rect.area)       // 200.0
print(rect.perimeter)  // 60.0
```

計算屬性不儲存值，每次存取時動態計算。類似 Java 的 getter 方法，但語法更簡潔。

### Getter 和 Setter

```swift
struct Temperature {
    var celsius: Double
    
    var fahrenheit: Double {
        get {
            return celsius * 9 / 5 + 32
        }
        set {
            celsius = (newValue - 32) * 5 / 9
        }
    }
}

var temp = Temperature(celsius: 0)
print(temp.fahrenheit)  // 32.0
temp.fahrenheit = 98.6
print(temp.celsius)     // 37.0
```

`newValue` 是預設的 setter 參數名稱，也可以自訂。

### 屬性觀察器 (Property Observers)

```swift
class Account {
    var balance: Double = 0.0 {
        willSet {
            print("Balance will change from \(balance) to \(newValue)")
        }
        didSet {
            print("Balance changed from \(oldValue) to \(balance)")
            if balance < 0 {
                print("Warning: Negative balance")
            }
        }
    }
}

let account = Account()
account.balance = 100.0
account.balance = -50.0
```

`willSet` 在值改變前呼叫，`didSet` 在值改變後呼叫。這在需要追蹤狀態變化時很有用。

### 延遲儲存屬性 (Lazy Stored Properties)

```swift
class DataManager {
    lazy var database: Database = {
        print("Initializing database")
        return Database()
    }()
}

class Database {
    init() {
        print("Database created")
    }
}

let manager = DataManager()
print("Manager created")
// 此時 database 還未初始化

let db = manager.database
// 現在才會初始化 database
```

`lazy` 屬性在第一次存取時才會初始化，適合處理耗資源的物件。類似 Java 的 lazy loading 模式。

### 型別屬性 (Type Properties)

```swift
struct Math {
    static let pi = 3.14159
    static var computationCount = 0
    
    static func calculateCircleArea(radius: Double) -> Double {
        computationCount += 1
        return pi * radius * radius
    }
}

print(Math.pi)
let area = Math.calculateCircleArea(radius: 5)
print(Math.computationCount)  // 1
```

使用 `static` 關鍵字定義型別屬性，相當於 Java 的 static 欄位。類別可以使用 `class` 關鍵字讓子類別可以覆寫。

## 方法 (Methods)

### 實例方法

```swift
class Counter {
    var count = 0
    
    func increment() {
        count += 1
    }
    
    func increment(by amount: Int) {
        count += amount
    }
    
    func reset() {
        count = 0
    }
}

let counter = Counter()
counter.increment()
counter.increment(by: 5)
print(counter.count)  // 6
```

### 修改值型別的方法

結構的方法預設不能修改屬性，需要標記 `mutating`：

```swift
struct Point {
    var x: Double
    var y: Double
    
    mutating func moveBy(deltaX: Double, deltaY: Double) {
        x += deltaX
        y += deltaY
    }
}

var point = Point(x: 0, y: 0)
point.moveBy(deltaX: 10, deltaY: 20)
print(point.x, point.y)  // 10.0 20.0
```

使用 `let` 宣告的結構實例無法呼叫 mutating 方法。

### 型別方法

```swift
class Utility {
    class func generateId() -> Int {
        return Int.random(in: 1000...9999)
    }
    
    static func formatDate(_ date: Date) -> String {
        return "\(date)"
    }
}

let id = Utility.generateId()
```

類別使用 `class` 或 `static`，結構只能使用 `static`。

## 初始化 (Initialization)

### 指定初始化方法

```swift
class Person {
    let name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
    init(name: String) {
        self.name = name
        self.age = 0
    }
}

let person1 = Person(name: "John", age: 30)
let person2 = Person(name: "Jane")
```

Swift 要求所有屬性在初始化完成前都必須有值。

### 便利初始化方法

```swift
class Rectangle {
    var width: Double
    var height: Double
    
    init(width: Double, height: Double) {
        self.width = width
        self.height = height
    }
    
    convenience init(size: Double) {
        self.init(width: size, height: size)
    }
}

let square = Rectangle(size: 10)
```

`convenience` 初始化方法必須呼叫同一類別的其他初始化方法。

### 可失敗的初始化方法

```swift
struct Temperature {
    var celsius: Double
    
    init?(fahrenheit: Double) {
        guard fahrenheit >= -459.67 else {
            return nil
        }
        celsius = (fahrenheit - 32) * 5 / 9
    }
}

if let temp = Temperature(fahrenheit: 100) {
    print("Valid temperature: \(temp.celsius)")
} else {
    print("Invalid temperature")
}
```

使用 `init?` 定義可能失敗的初始化，回傳 Optional。

## 解初始化 (Deinitialization)

```swift
class FileHandler {
    let fileName: String
    
    init(fileName: String) {
        self.fileName = fileName
        print("Opening file: \(fileName)")
    }
    
    deinit {
        print("Closing file: \(fileName)")
    }
}

var handler: FileHandler? = FileHandler(fileName: "data.txt")
handler = nil  // 觸發 deinit
```

`deinit` 在實例被釋放前自動呼叫，類似 Java 的 finalize，但更可靠。只有類別有 deinitializer。

## 下標 (Subscripts)

```swift
struct Matrix {
    let rows: Int
    let columns: Int
    var grid: [Double]
    
    init(rows: Int, columns: Int) {
        self.rows = rows
        self.columns = columns
        self.grid = Array(repeating: 0.0, count: rows * columns)
    }
    
    subscript(row: Int, column: Int) -> Double {
        get {
            return grid[row * columns + column]
        }
        set {
            grid[row * columns + column] = newValue
        }
    }
}

var matrix = Matrix(rows: 2, columns: 2)
matrix[0, 1] = 1.5
print(matrix[0, 1])  // 1.5
```

下標讓自訂型別可以使用中括號語法存取元素。

## 與 Java 的比較

1. **值型別**：Swift 的結構是值型別，Java 只有原始型別是值型別
2. **計算屬性**：Swift 原生支援，Java 需要 getter/setter 方法
3. **屬性觀察器**：Swift 內建機制，Java 需要手動在 setter 中實作
4. **延遲初始化**：Swift 使用 `lazy` 關鍵字，Java 通常手動實作
5. **可失敗初始化**：Swift 可以回傳 nil，Java 通常拋出異常

## 小結

Swift 的類別和結構提供了豐富的特性。結構作為值型別的設計是 Swift 的特色，鼓勵不可變性和安全的資料傳遞。計算屬性、屬性觀察器等特性讓程式碼更簡潔優雅。

下週將學習繼承、多型、協定等進階的物件導向特性。
