---
layout: post
title: "Swift 函式與閉包"
date: 2018-01-12
categories: [Swift, iOS]
tags: [Swift, Function, Closure]
---

第二週深入學習 Swift 的函式和閉包。這兩個概念在 Swift 中非常重要，尤其是閉包，在處理非同步程式碼和集合操作時會大量使用。

從 Java 的角度來看，Swift 的函式類似方法（method），但更靈活。閉包則類似 Java 8 的 Lambda 表達式，但功能更強大。

## 函式基礎

### 函式定義

Swift 使用 `func` 關鍵字定義函式：

```swift
func greet(name: String) -> String {
    return "Hello, \(name)"
}

let message = greet(name: "John")
```

與 Java 方法相比，Swift 的函式定義更簡潔，參數名稱在呼叫時必須明確指定。

### 多參數函式

```swift
func calculateArea(width: Double, height: Double) -> Double {
    return width * height
}

let area = calculateArea(width: 10.0, height: 5.0)
```

### 無回傳值函式

```swift
func printMessage(message: String) -> Void {
    print(message)
}

// Void 可以省略
func logError(error: String) {
    print("Error: \(error)")
}
```

## 參數標籤

這是 Swift 函式設計的特色，也是一開始讓我覺得奇怪的地方。為什麼要有兩個名稱？

### Argument Label

Swift 支援外部參數名稱和內部參數名稱：

```swift
func greet(to name: String, from sender: String) -> String {
    return "Hello \(name), from \(sender)"
}

let message = greet(to: "John", from: "Jane")
```

`to` 和 `from` 是外部參數標籤，`name` 和 `sender` 是內部使用的參數名稱。

### 省略外部參數名稱

使用底線 `_` 可以省略外部參數標籤：

```swift
func add(_ a: Int, _ b: Int) -> Int {
    return a + b
}

let sum = add(5, 3)  // 不需要參數標籤
```

這種寫法接近 Java 方法的呼叫方式。

## 預設參數值

```swift
func connect(timeout: Int = 30, retry: Int = 3) {
    print("Connecting with timeout: \(timeout)s, retry: \(retry)")
}

connect()                    // 使用預設值
connect(timeout: 60)         // 只覆寫 timeout
connect(timeout: 45, retry: 5)  // 全部覆寫
```

Java 需要透過方法多載達成，Swift 的語法更直觀。

## 可變參數

```swift
func sum(_ numbers: Int...) -> Int {
    var total = 0
    for number in numbers {
        total += number
    }
    return total
}

let result1 = sum(1, 2, 3)        // 6
let result2 = sum(1, 2, 3, 4, 5)  // 15
```

類似 Java 的 varargs，但語法是使用三個點。

## In-Out 參數

Swift 的參數預設是常數，無法修改。若要修改參數值，需使用 `inout`：

```swift
func increment(_ value: inout Int) {
    value += 1
}

var count = 5
increment(&count)  // 需要加 & 符號
print(count)       // 6
```

這類似 C++ 的參考傳遞，Java 沒有對應的機制。

## 函式型別

Swift 的函式是 first-class citizen，可以作為變數、參數、回傳值：

```swift
func add(_ a: Int, _ b: Int) -> Int {
    return a + b
}

func multiply(_ a: Int, _ b: Int) -> Int {
    return a * b
}

// 函式型別：(Int, Int) -> Int
var mathOperation: (Int, Int) -> Int = add
print(mathOperation(5, 3))  // 8

mathOperation = multiply
print(mathOperation(5, 3))  // 15
```

### 函式作為參數

```swift
func calculate(_ a: Int, _ b: Int, operation: (Int, Int) -> Int) -> Int {
    return operation(a, b)
}

let result = calculate(10, 5, operation: add)  // 15
```

這種寫法類似 Java 8 的函式介面和 Lambda 表達式。

### 函式作為回傳值

```swift
func makeIncrementer(step: Int) -> (Int) -> Int {
    func incrementer(value: Int) -> Int {
        return value + step
    }
    return incrementer
}

let incrementByTwo = makeIncrementer(step: 2)
print(incrementByTwo(5))  // 7
print(incrementByTwo(10)) // 12
```

## 閉包 (Closure)

閉包是自包含的程式碼區塊，類似 Java 的 Lambda 表達式。

### 基本語法

```swift
let numbers = [1, 2, 3, 4, 5]

// 完整閉包語法
let doubled = numbers.map({ (number: Int) -> Int in
    return number * 2
})
```

閉包語法結構：
```
{ (參數) -> 回傳型別 in
    程式碼
}
```

### 型別推斷

Swift 可以推斷閉包的參數和回傳型別：

```swift
let doubled = numbers.map({ number in
    return number * 2
})
```

### 單行閉包的隱式回傳

```swift
let doubled = numbers.map({ number in number * 2 })
```

### 簡寫參數名稱

使用 `$0`, `$1`, `$2` 等代表參數：

```swift
let doubled = numbers.map({ $0 * 2 })
```

### Trailing Closure

當閉包是函式的最後一個參數時，可以寫在括號外：

```swift
let doubled = numbers.map() { $0 * 2 }

// 如果閉包是唯一參數，可以省略括號
let tripled = numbers.map { $0 * 3 }
```

這是 Swift 常見的寫法，比 Java 的 Lambda 更簡潔。

## 閉包實際應用

### 陣列操作

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

// Filter：過濾偶數
let evenNumbers = numbers.filter { $0 % 2 == 0 }
// [2, 4, 6, 8, 10]

// Map：轉換
let squared = numbers.map { $0 * $0 }
// [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

// Reduce：聚合
let sum = numbers.reduce(0) { $0 + $1 }
// 55

// 串接操作
let result = numbers
    .filter { $0 % 2 == 0 }
    .map { $0 * $0 }
    .reduce(0, +)
// 220
```

這些操作類似 Java 8 的 Stream API。

### Sorted

```swift
let names = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]

// 降冪排序
let reversed = names.sorted { $0 > $1 }
// ["Ewa", "Daniella", "Chris", "Barry", "Alex"]

// 依字串長度排序
let byLength = names.sorted { $0.count < $1.count }
// ["Ewa", "Alex", "Chris", "Barry", "Daniella"]
```

## 捕獲值 (Capturing Values)

閉包可以捕獲外部變數：

```swift
func makeCounter() -> () -> Int {
    var count = 0
    let counter = {
        count += 1
        return count
    }
    return counter
}

let counter1 = makeCounter()
print(counter1())  // 1
print(counter1())  // 2
print(counter1())  // 3

let counter2 = makeCounter()
print(counter2())  // 1
```

每個閉包實例維護自己的 count 副本，這是閉包的重要特性。

## Escaping Closure

當閉包在函式返回後才執行時，需要標記為 `@escaping`：

```swift
var completionHandlers: [() -> Void] = []

func registerHandler(handler: @escaping () -> Void) {
    completionHandlers.append(handler)
}

func processData(completion: @escaping (String) -> Void) {
    // 模擬非同步操作
    DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
        completion("Data processed")
    }
}
```

這在處理非同步操作時很常見，類似 Java 的 Callback。

## Autoclosure

`@autoclosure` 自動將表達式包裝成閉包：

```swift
func logIfTrue(_ condition: @autoclosure () -> Bool, message: String) {
    if condition() {
        print(message)
    }
}

let value = 10
logIfTrue(value > 5, message: "Value is greater than 5")
```

不需要明確寫閉包語法，編譯器會自動處理。

## 與 Java 的比較

1. **語法簡潔度**：Swift 閉包的簡寫形式比 Java Lambda 更精簡
2. **參數標籤**：Swift 的外部參數名稱提升程式碼可讀性
3. **Trailing Closure**：Swift 獨有的語法糖，讓 DSL 風格的 API 更優雅
4. **捕獲值**：兩者都支援，但 Swift 的語法更直觀
5. **Escaping**：Swift 明確標註閉包的生命週期，Java 透過介面語意處理

## 小結

Swift 的函式和閉包設計展現了函數式程式設計的理念。相較於 Java，Swift 提供更簡潔的語法和更強大的功能。Trailing closure 語法特別適合建構 DSL，在 iOS 開發中會頻繁使用。

下週將學習物件導向程式設計，包含類別、結構、列舉等型別的定義與使用。
