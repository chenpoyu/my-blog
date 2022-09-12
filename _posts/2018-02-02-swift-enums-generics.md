---
layout: post
title: "Swift 列舉與泛型"
date: 2018-02-02
categories: [Swift, iOS]
tags: [Swift, Enum, Generics]
---

這週研究 Swift 的列舉和泛型。這兩個概念在 Java 中也有，但 Swift 的實作方式讓我大開眼界。

**列舉（Enum）**：Java 的 enum 已經比 C 的 enum 強大很多，但 Swift 的列舉更是進化到另一個層次。它可以有關聯值（Associated Values）、可以有方法、甚至可以遞迴定義。

**泛型（Generics）**：概念和 Java 泛型類似，但語法更簡潔，而且沒有 type erasure 的問題（Java 泛型在執行時會被抹除，Swift 保留型別資訊）。

## 列舉 (Enumerations)

Swift 的列舉比 Java 更強大，是 first-class type，可以有方法、屬性、初始化方法。

### 基本列舉

```swift
enum Direction {
    case north
    case south
    case east
    case west
}

// 可以寫在同一行
enum CompassPoint {
    case north, south, east, west
}

var heading = Direction.north
heading = .south  // 型別已知，可以省略型別名稱
```

### Switch 配對

```swift
enum Direction {
    case north, south, east, west
}

func navigate(to direction: Direction) {
    switch direction {
    case .north:
        print("Going north")
    case .south:
        print("Going south")
    case .east:
        print("Going east")
    case .west:
        print("Going west")
    }
}
```

Switch 必須涵蓋所有 case，或提供 default，編譯器會檢查。

### 關聯值 (Associated Values)

這是 Swift 列舉的殺手級特性：

```swift
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

var productBarcode = Barcode.upc(8, 85909, 51226, 3)
productBarcode = .qrCode("ABCDEFGHIJKLMNOP")

switch productBarcode {
case .upc(let numberSystem, let manufacturer, let product, let check):
    print("UPC: \(numberSystem), \(manufacturer), \(product), \(check)")
case .qrCode(let code):
    print("QR code: \(code)")
}
```

每個 case 可以儲存不同型別的關聯值，類似 tagged union。Java 需要使用類別階層來達成類似效果。

### 簡化語法

```swift
switch productBarcode {
case let .upc(numberSystem, manufacturer, product, check):
    print("UPC: \(numberSystem), \(manufacturer), \(product), \(check)")
case let .qrCode(code):
    print("QR code: \(code)")
}
```

當所有關聯值都要提取時，可以將 `let` 放在 case 前面。

### 原始值 (Raw Values)

```swift
enum Planet: Int {
    case mercury = 1
    case venus
    case earth
    case mars
    case jupiter
    case saturn
    case uranus
    case neptune
}

let earthOrder = Planet.earth.rawValue  // 3
```

原始值在編譯時就確定，所有 case 的原始值必須唯一。

### 從原始值初始化

```swift
enum Planet: Int {
    case mercury = 1, venus, earth, mars
}

if let planet = Planet(rawValue: 2) {
    print(planet)  // venus
}

let unknown = Planet(rawValue: 10)  // nil
```

原始值初始化是可失敗的，回傳 Optional。

### 遞迴列舉

```swift
indirect enum ArithmeticExpression {
    case number(Int)
    case addition(ArithmeticExpression, ArithmeticExpression)
    case multiplication(ArithmeticExpression, ArithmeticExpression)
}

// 或者標記整個列舉
// indirect enum ArithmeticExpression { ... }

let five = ArithmeticExpression.number(5)
let four = ArithmeticExpression.number(4)
let sum = ArithmeticExpression.addition(five, four)
let product = ArithmeticExpression.multiplication(sum, ArithmeticExpression.number(2))

func evaluate(_ expression: ArithmeticExpression) -> Int {
    switch expression {
    case let .number(value):
        return value
    case let .addition(left, right):
        return evaluate(left) + evaluate(right)
    case let .multiplication(left, right):
        return evaluate(left) * evaluate(right)
    }
}

let result = evaluate(product)  // (5 + 4) * 2 = 18
```

`indirect` 關鍵字允許列舉的 case 包含自己的實例。

### 列舉的屬性與方法

```swift
enum Device {
    case iPhone, iPad, watch, tv
    
    var year: Int {
        switch self {
        case .iPhone:
            return 2007
        case .iPad:
            return 2010
        case .watch:
            return 2015
        case .tv:
            return 2015
        }
    }
    
    func info() -> String {
        return "Released in \(year)"
    }
}

let device = Device.iPhone
print(device.info())  // Released in 2007
```

列舉可以有計算屬性和方法，但不能有儲存屬性。

## 泛型 (Generics)

泛型讓程式碼更具重用性和型別安全性。

### 泛型函式

```swift
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

var x = 10
var y = 20
swapValues(&x, &y)
print(x, y)  // 20 10

var str1 = "Hello"
var str2 = "World"
swapValues(&str1, &str2)
print(str1, str2)  // World Hello
```

使用角括號 `<T>` 定義型別參數，類似 Java 的泛型。

### 泛型型別

```swift
struct Stack<Element> {
    private var items: [Element] = []
    
    mutating func push(_ item: Element) {
        items.append(item)
    }
    
    mutating func pop() -> Element? {
        return items.isEmpty ? nil : items.removeLast()
    }
    
    func peek() -> Element? {
        return items.last
    }
    
    var isEmpty: Bool {
        return items.isEmpty
    }
    
    var count: Int {
        return items.count
    }
}

var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)
intStack.push(3)
print(intStack.pop())  // Optional(3)

var stringStack = Stack<String>()
stringStack.push("A")
stringStack.push("B")
```

泛型型別的使用方式與 Java 類似。

### 型別約束

```swift
func findIndex<T: Equatable>(of valueToFind: T, in array: [T]) -> Int? {
    for (index, value) in array.enumerated() {
        if value == valueToFind {
            return index
        }
    }
    return nil
}

let numbers = [1, 2, 3, 4, 5]
if let index = findIndex(of: 3, in: numbers) {
    print("Found at index \(index)")  // Found at index 2
}
```

使用冒號指定型別約束，要求 T 必須實作 Equatable 協定。

### 關聯型別 (Associated Types)

```swift
protocol Container {
    associatedtype Item
    
    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

struct IntStack: Container {
    typealias Item = Int  // 可以省略，Swift 會自動推斷
    
    private var items: [Int] = []
    
    var count: Int {
        return items.count
    }
    
    mutating func append(_ item: Int) {
        items.append(item)
    }
    
    subscript(i: Int) -> Int {
        return items[i]
    }
}
```

關聯型別讓協定可以使用泛型，類似 Java interface 的泛型參數。

### 泛型約束中的 Where 子句

```swift
func allItemsMatch<C1: Container, C2: Container>
    (_ container1: C1, _ container2: C2) -> Bool
    where C1.Item == C2.Item, C1.Item: Equatable {
    
    if container1.count != container2.count {
        return false
    }
    
    for i in 0..<container1.count {
        if container1[i] != container2[i] {
            return false
        }
    }
    
    return true
}
```

`where` 子句可以指定更複雜的型別約束。

### 泛型擴展

```swift
extension Stack {
    var topItem: Element? {
        return items.isEmpty ? nil : items.last
    }
}

extension Stack where Element: Equatable {
    func contains(_ item: Element) -> Bool {
        return items.contains(item)
    }
}
```

可以為泛型型別新增擴展，也可以限制擴展只適用於特定型別。

## 錯誤處理

### 定義錯誤

```swift
enum NetworkError: Error {
    case badURL
    case requestFailed
    case invalidResponse
    case decodingFailed
}
```

錯誤型別需要實作 Error 協定，通常使用列舉定義。

### 拋出錯誤

```swift
func fetchData(from urlString: String) throws -> Data {
    guard let url = URL(string: urlString) else {
        throw NetworkError.badURL
    }
    
    guard let data = try? Data(contentsOf: url) else {
        throw NetworkError.requestFailed
    }
    
    return data
}
```

使用 `throws` 標記函式可能拋出錯誤，使用 `throw` 拋出錯誤。

### 處理錯誤

```swift
// do-catch
do {
    let data = try fetchData(from: "https://example.com")
    print("Received \(data.count) bytes")
} catch NetworkError.badURL {
    print("Invalid URL")
} catch NetworkError.requestFailed {
    print("Request failed")
} catch {
    print("Unknown error: \(error)")
}

// try?
if let data = try? fetchData(from: "https://example.com") {
    print("Success")
}

// try!
let data = try! fetchData(from: "https://example.com")  // 確定不會失敗時使用
```

提供三種處理方式：do-catch 捕捉、try? 轉換為 Optional、try! 強制執行。

### defer 語句

```swift
func processFile(filename: String) throws {
    let file = openFile(filename)
    defer {
        closeFile(file)
    }
    
    // 處理檔案
    let data = try readFile(file)
    
    // defer 區塊會在函式返回前執行
}
```

`defer` 區塊在離開當前作用域前執行，常用於清理資源。類似 Java 的 try-with-resources，但更靈活。

## Result 型別

Swift 5.0 引入 Result 型別，用於表示成功或失敗：

```swift
enum DataError: Error {
    case networkError
    case parsingError
}

func loadData() -> Result<String, DataError> {
    let success = Bool.random()
    
    if success {
        return .success("Data loaded")
    } else {
        return .failure(.networkError)
    }
}

let result = loadData()
switch result {
case .success(let data):
    print(data)
case .failure(let error):
    print("Error: \(error)")
}
```

雖然這是 Swift 5.0 的特性，但概念在這個時期已經被討論。

## 與 Java 的比較

1. **列舉**：Swift 列舉更強大，支援關聯值和遞迴
2. **泛型**：語法類似，但 Swift 沒有型別擦除問題
3. **錯誤處理**：Swift 使用 throws/try，Java 使用 checked/unchecked exception
4. **型別安全**：兩者都強調編譯時期的型別檢查

## 小結

Swift 的列舉和泛型展現了語言的表達能力。列舉的關聯值特性讓資料建模更直觀，泛型提供了型別安全的重用機制。錯誤處理採用類似 Java checked exception 的設計，但語法更簡潔。

下週將開始學習 iOS 開發的基礎，包含 UIKit 框架和基本的 UI 元件。
