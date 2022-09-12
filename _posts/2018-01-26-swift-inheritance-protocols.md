---
layout: post
title: "Swift 繼承與協定"
date: 2018-01-26
categories: [Swift, iOS]
tags: [Swift, Inheritance, Protocol]
---

第四週學習物件導向的核心概念：繼承和協定（Protocol）。

在 Java 中，我們用 `extends` 實現繼承，用 `interface` 定義介面。Swift 的概念類似，但語法和設計哲學有些不同。

特別值得注意的是，Swift 的 **Protocol** 比 Java 的 interface 更強大。它不只是定義方法簽名，還能提供預設實作、要求屬性、甚至用在泛型約束上。這讓 Swift 的程式設計更靈活，也是 Apple 推廣「Protocol-Oriented Programming」的基礎。

## 繼承 (Inheritance)

只有類別支援繼承，結構和列舉不支援。這是因為結構是值型別，繼承的語義在值型別上不太合理。

### 基本繼承

```swift
class Vehicle {
    var speed: Double = 0.0
    
    func description() -> String {
        return "Moving at \(speed) km/h"
    }
}

class Car: Vehicle {
    var gear: Int = 1
    
    func changeGear(to newGear: Int) {
        gear = newGear
        print("Gear changed to \(gear)")
    }
}

let car = Car()
car.speed = 60.0
car.gear = 3
print(car.description())
```

語法與 Java 類似，但使用冒號而非 `extends` 關鍵字。

### 覆寫方法

```swift
class Vehicle {
    var speed: Double = 0.0
    
    func description() -> String {
        return "Moving at \(speed) km/h"
    }
    
    func makeNoise() {
        print("Some noise")
    }
}

class Car: Vehicle {
    override func description() -> String {
        return "Car is \(super.description())"
    }
    
    override func makeNoise() {
        print("Vroom vroom")
    }
}

let car = Car()
car.speed = 100
print(car.description())  // Car is Moving at 100.0 km/h
car.makeNoise()           // Vroom vroom
```

必須使用 `override` 關鍵字，編譯器會檢查是否真的有覆寫父類別的方法。

### 覆寫屬性

```swift
class Vehicle {
    var speed: Double = 0.0
    var description: String {
        return "Moving at \(speed) km/h"
    }
}

class Car: Vehicle {
    var gear: Int = 1
    
    override var description: String {
        return "Car in gear \(gear), \(super.description)"
    }
    
    override var speed: Double {
        didSet {
            print("Speed changed to \(speed)")
        }
    }
}

let car = Car()
car.speed = 60  // Speed changed to 60.0
```

可以覆寫計算屬性，也可以為繼承的屬性新增屬性觀察器。

### 防止覆寫

```swift
class Vehicle {
    final func startEngine() {
        print("Engine started")
    }
    
    final var maxSpeed: Double {
        return 200.0
    }
}

// final class FinalVehicle {}  // 整個類別不可被繼承
```

使用 `final` 關鍵字防止子類別覆寫，類似 Java 的 `final`。

## 協定 (Protocol)

協定定義了方法、屬性和其他需求的藍圖，類似 Java 的 interface。

### 定義協定

```swift
protocol Drawable {
    var color: String { get set }
    var lineWidth: Double { get }
    
    func draw()
    func erase()
}
```

屬性需求必須指定是唯讀 `{ get }` 還是可讀寫 `{ get set }`。

### 實作協定

```swift
class Circle: Drawable {
    var color: String = "black"
    var lineWidth: Double = 1.0
    var radius: Double = 10.0
    
    func draw() {
        print("Drawing a \(color) circle with radius \(radius)")
    }
    
    func erase() {
        print("Erasing circle")
    }
}

struct Rectangle: Drawable {
    var color: String
    var lineWidth: Double
    var width: Double
    var height: Double
    
    func draw() {
        print("Drawing a \(color) rectangle \(width)x\(height)")
    }
    
    func erase() {
        print("Erasing rectangle")
    }
}
```

類別和結構都可以實作協定。

### 協定作為型別

```swift
func render(shape: Drawable) {
    shape.draw()
}

let circle = Circle()
let rectangle = Rectangle(color: "red", lineWidth: 2.0, width: 10, height: 20)

render(shape: circle)
render(shape: rectangle)
```

協定可以作為參數型別，實現多型。

### 多個協定

```swift
protocol Named {
    var name: String { get }
}

protocol Aged {
    var age: Int { get }
}

class Person: Named, Aged {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}
```

使用逗號分隔多個協定，如同 Java 可以實作多個 interface。

### 繼承與協定

```swift
class Animal {
    var name: String
    
    init(name: String) {
        self.name = name
    }
}

protocol CanFly {
    func fly()
}

class Bird: Animal, CanFly {
    func fly() {
        print("\(name) is flying")
    }
}
```

父類別寫在前，協定寫在後。

## 協定的方法需求

### 實例方法需求

```swift
protocol RandomNumberGenerator {
    func random() -> Double
}

class LinearCongruentialGenerator: RandomNumberGenerator {
    var lastRandom = 42.0
    let m = 139968.0
    let a = 3877.0
    let c = 29573.0
    
    func random() -> Double {
        lastRandom = (lastRandom * a + c).truncatingRemainder(dividingBy: m)
        return lastRandom / m
    }
}
```

### Mutating 方法需求

```swift
protocol Toggleable {
    mutating func toggle()
}

struct Switch: Toggleable {
    var isOn: Bool = false
    
    mutating func toggle() {
        isOn = !isOn
    }
}

var lightSwitch = Switch()
lightSwitch.toggle()
print(lightSwitch.isOn)  // true
```

協定中的 `mutating` 需求允許結構和列舉修改自身。類別實作時不需要 `mutating` 關鍵字。

### 初始化方法需求

```swift
protocol Initializable {
    init(value: Int)
}

class Container: Initializable {
    var value: Int
    
    required init(value: Int) {
        self.value = value
    }
}
```

類別實作協定的初始化方法需要使用 `required` 關鍵字，確保子類別也實作此初始化方法。

## 協定的繼承

```swift
protocol Named {
    var name: String { get }
}

protocol Identifiable: Named {
    var id: String { get }
}

class User: Identifiable {
    var name: String
    var id: String
    
    init(name: String, id: String) {
        self.name = name
        self.id = id
    }
}
```

協定可以繼承其他協定，類似 Java interface 的繼承。

### 類別專屬協定

```swift
protocol SomeClassProtocol: AnyObject {
    func doSomething()
}

class MyClass: SomeClassProtocol {
    func doSomething() {
        print("Doing something")
    }
}

// struct MyStruct: SomeClassProtocol {}  // 編譯錯誤
```

使用 `AnyObject` 限制協定只能被類別實作。

## 協定組合

```swift
protocol Named {
    var name: String { get }
}

protocol Aged {
    var age: Int { get }
}

func wishHappyBirthday(to person: Named & Aged) {
    print("Happy birthday, \(person.name), you are now \(person.age)!")
}

class Person: Named, Aged {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

let person = Person(name: "John", age: 30)
wishHappyBirthday(to: person)
```

使用 `&` 符號組合多個協定需求。

## 協定擴展

```swift
protocol Describable {
    var description: String { get }
}

extension Describable {
    var uppercaseDescription: String {
        return description.uppercased()
    }
    
    func printDescription() {
        print(description)
    }
}

struct Product: Describable {
    var name: String
    var description: String {
        return "Product: \(name)"
    }
}

let product = Product(name: "iPhone")
product.printDescription()           // Product: iPhone
print(product.uppercaseDescription) // PRODUCT: IPHONE
```

協定擴展可以為協定提供預設實作，這是 Swift 的強大特性。類似 Java 8 的 default method，但更靈活。

### 條件性擴展

```swift
extension Collection where Element: Equatable {
    func allEqual() -> Bool {
        guard let first = self.first else { return true }
        return self.allSatisfy { $0 == first }
    }
}

let numbers = [1, 1, 1, 1]
print(numbers.allEqual())  // true

let mixed = [1, 2, 3]
print(mixed.allEqual())    // false
```

可以對符合特定條件的型別提供擴展。

## 型別檢查與轉換

### 型別檢查

```swift
protocol Animal {
    func makeSound()
}

class Dog: Animal {
    func makeSound() {
        print("Woof")
    }
    
    func fetch() {
        print("Fetching")
    }
}

class Cat: Animal {
    func makeSound() {
        print("Meow")
    }
}

let animals: [Animal] = [Dog(), Cat(), Dog()]

for animal in animals {
    if animal is Dog {
        print("This is a dog")
    } else if animal is Cat {
        print("This is a cat")
    }
}
```

使用 `is` 運算子檢查型別，類似 Java 的 `instanceof`。

### 型別轉換

```swift
for animal in animals {
    if let dog = animal as? Dog {
        dog.fetch()
    }
    animal.makeSound()
}
```

使用 `as?` 進行安全的向下轉型，失敗時回傳 nil。`as!` 是強制轉型，失敗會發生執行時期錯誤。

## 委託模式 (Delegation)

委託是 iOS 開發中常見的設計模式：

```swift
protocol DataSourceDelegate: AnyObject {
    func numberOfItems() -> Int
    func itemAt(index: Int) -> String
}

class ListView {
    weak var delegate: DataSourceDelegate?
    
    func display() {
        guard let delegate = delegate else { return }
        
        let count = delegate.numberOfItems()
        for i in 0..<count {
            let item = delegate.itemAt(index: i)
            print("Item \(i): \(item)")
        }
    }
}

class DataSource: DataSourceDelegate {
    let items = ["Apple", "Banana", "Orange"]
    
    func numberOfItems() -> Int {
        return items.count
    }
    
    func itemAt(index: Int) -> String {
        return items[index]
    }
}

let listView = ListView()
let dataSource = DataSource()
listView.delegate = dataSource
listView.display()
```

使用 `weak` 避免循環參考，這是記憶體管理的重要技巧。

## 與 Java 的比較

1. **協定 vs Interface**：Swift 協定功能更強大，支援屬性需求和擴展
2. **值型別實作協定**：Swift 的結構可以實作協定，Java 只有類別能實作 interface
3. **協定擴展**：Swift 可以為協定提供預設實作，比 Java 8 的 default method 更靈活
4. **型別限制**：Swift 可以限制協定只能被類別實作
5. **協定組合**：Swift 支援臨時組合多個協定

## 小結

Swift 的協定是面向協定程式設計的核心。相較於 Java 的 interface，Swift 協定提供更多功能，特別是協定擴展機制，可以為協定提供預設實作，減少重複程式碼。委託模式在 iOS 開發中無所不在，理解協定對後續學習至關重要。

下週將學習列舉、泛型以及錯誤處理機制。
