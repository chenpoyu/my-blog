---
layout: post
title: "Swift Protocol 與 Extensions：協定導向編程"
date: 2018-07-06
categories: [iOS, Swift]
tags: [Swift, Protocol, Extensions, POP]
---

這週深入研究 Swift 的 Protocol 和 Extensions，發現這是 Swift 最強大的特性之一。Apple 稱之為 Protocol-Oriented Programming (POP)，是不同於 Java 物件導向編程的思維方式。

## 為什麼需要 Protocol-Oriented Programming

在 Java 中，我們習慣用繼承來重用程式碼：

```java
public abstract class Animal {
    public void eat() {
        System.out.println("Eating...");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println("Woof!");
    }
}
```

但繼承有幾個問題：
1. **單一繼承限制**：一個類別只能繼承一個父類別
2. **繼承層級過深**：難以維護和理解
3. **強耦合**：子類別依賴父類別的實作細節

Swift 的 Protocol + Extensions 提供更靈活的解決方案。

## Protocol 的基本使用

Protocol 類似 Java 的 interface，定義一組方法和屬性：

```swift
protocol Flyable {
    var altitude: Double { get set }
    func fly()
    func land()
}

class Bird: Flyable {
    var altitude: Double = 0
    
    func fly() {
        altitude = 100
        print("Bird flying at \(altitude)m")
    }
    
    func land() {
        altitude = 0
        print("Bird landed")
    }
}
```

但 Swift 的 Protocol 比 Java interface 更強大：

**1. 可以定義屬性**

```swift
protocol Identifiable {
    var id: String { get }  // 唯讀屬性
    var name: String { get set }  // 可讀寫屬性
}
```

Java 的 interface 只能定義方法，不能定義屬性（Java 8 後可以定義 default method，但還是不如 Swift 靈活）。

**2. 可以用於值型別**

```swift
struct Airplane: Flyable {  // struct 也能遵循 protocol
    var altitude: Double = 0
    
    func fly() {
        print("Airplane flying")
    }
    
    func land() {
        print("Airplane landed")
    }
}
```

Java 的 interface 只能用於類別，struct 和 enum 不能實作 interface。

## Protocol Extensions：帶有預設實作的 Protocol

這是 Swift 最強大的特性之一。可以為 Protocol 提供預設實作：

```swift
protocol Drivable {
    var speed: Double { get set }
    func drive()
    func stop()
}

extension Drivable {
    // 提供預設實作
    func drive() {
        speed = 60
        print("Driving at \(speed) km/h")
    }
    
    func stop() {
        speed = 0
        print("Stopped")
    }
}

struct Car: Drivable {
    var speed: Double = 0
    // 不需要實作 drive() 和 stop()，會使用預設實作
}

let car = Car()
car.drive()  // 使用預設實作
```

這類似 Java 8 的 default method，但更徹底：

```java
// Java 8 default method
public interface Drivable {
    default void drive() {
        System.out.println("Driving");
    }
}
```

**關鍵差異**：
- Swift 的 extension 可以在 protocol 定義之外加入
- Swift 的 extension 可以加入計算屬性
- Swift 的 extension 可以有條件約束（後面會講）

## Protocol 組合

一個型別可以遵循多個 protocol，解決單一繼承的限制：

```swift
protocol Walkable {
    func walk()
}

protocol Swimmable {
    func swim()
}

protocol Flyable {
    func fly()
}

// 鴨子可以走、游、飛
struct Duck: Walkable, Swimmable, Flyable {
    func walk() { print("Duck walking") }
    func swim() { print("Duck swimming") }
    func fly() { print("Duck flying") }
}

// 企鵝只能走和游
struct Penguin: Walkable, Swimmable {
    func walk() { print("Penguin walking") }
    func swim() { print("Penguin swimming") }
}
```

在 Java 中，如果 Walkable、Swimmable、Flyable 是抽象類別，Duck 只能選擇繼承其中一個。用 interface 可以多重實作，但沒有程式碼重用。

Swift 的 Protocol Extensions 可以做到**多重實作 + 程式碼重用**。

## Protocol 的條件擴展

可以為特定條件的型別提供擴展：

```swift
protocol Container {
    associatedtype Item
    var items: [Item] { get set }
    mutating func add(_ item: Item)
}

// 為所有元素是 Equatable 的 Container 加入 contains 方法
extension Container where Item: Equatable {
    func contains(_ item: Item) -> Bool {
        return items.contains(item)
    }
}

struct NumberContainer: Container {
    var items: [Int] = []
    
    mutating func add(_ item: Int) {
        items.append(item)
    }
}

var container = NumberContainer()
container.add(5)
print(container.contains(5))  // true，因為 Int 遵循 Equatable
```

這個功能在 Java 中很難實現，需要用泛型加上很複雜的約束。

## 實戰案例：網路請求的抽象化

之前寫網路請求時，每個 API 都要重複寫類似的程式碼。用 Protocol + Extensions 可以大幅簡化：

```swift
protocol APIRequest {
    associatedtype Response: Decodable
    var endpoint: String { get }
    var method: String { get }
}

extension APIRequest {
    // 預設實作 GET 請求
    var method: String { return "GET" }
    
    func send(completion: @escaping (Result<Response, Error>) -> Void) {
        guard let url = URL(string: "https://api.example.com\(endpoint)") else {
            completion(.failure(NSError(domain: "Invalid URL", code: 0)))
            return
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = method
        
        URLSession.shared.dataTask(with: request) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NSError(domain: "No data", code: 0)))
                return
            }
            
            do {
                let decoded = try JSONDecoder().decode(Response.self, from: data)
                completion(.success(decoded))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}

// 定義具體的 API 請求
struct GetUserRequest: APIRequest {
    typealias Response = User
    var endpoint: String { return "/users/\(userId)" }
    let userId: Int
}

struct User: Decodable {
    let id: Int
    let name: String
}

// 使用
let request = GetUserRequest(userId: 123)
request.send { result in
    switch result {
    case .success(let user):
        print("User: \(user.name)")
    case .failure(let error):
        print("Error: \(error)")
    }
}
```

這個設計的好處：
- 每個 API 請求只需要定義 `endpoint` 和 `Response` 型別
- 網路請求的邏輯都在 extension 中，不用重複寫
- 型別安全，Response 是編譯時期確定的
- 易於測試，可以 mock APIRequest protocol

在 Java 中，類似的設計需要用抽象類別 + 泛型，程式碼會更冗長。

## Protocol 與 Delegate Pattern

iOS 開發中 Delegate 模式無所不在，其實就是 Protocol 的應用：

```swift
protocol DataFetcherDelegate: AnyObject {  // AnyObject 讓它只能用於 class
    func didFetchData(_ data: String)
    func didFailWithError(_ error: Error)
}

class DataFetcher {
    weak var delegate: DataFetcherDelegate?  // weak 避免循環參照
    
    func fetchData() {
        // 模擬網路請求
        DispatchQueue.global().asyncAfter(deadline: .now() + 1) {
            let success = Bool.random()
            
            DispatchQueue.main.async {
                if success {
                    self.delegate?.didFetchData("Some data")
                } else {
                    let error = NSError(domain: "Network error", code: 0)
                    self.delegate?.didFailWithError(error)
                }
            }
        }
    }
}

class ViewController: UIViewController, DataFetcherDelegate {
    let fetcher = DataFetcher()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        fetcher.delegate = self
        fetcher.fetchData()
    }
    
    func didFetchData(_ data: String) {
        print("Received: \(data)")
    }
    
    func didFailWithError(_ error: Error) {
        print("Error: \(error)")
    }
}
```

**為什麼用 Delegate 而不是閉包？**

閉包適合簡單的回呼：

```swift
fetcher.onComplete = { data in
    print(data)
}
```

Delegate 適合多個方法的情況：
- 方法多時，閉包會變成很多個屬性，不好管理
- Delegate 可以用 weak 避免循環參照
- Delegate 語意清楚，表達「誰負責處理這些事件」

## Protocol 與泛型的結合

Protocol 配合泛型能寫出更通用的程式碼：

```swift
protocol Stack {
    associatedtype Element
    mutating func push(_ element: Element)
    mutating func pop() -> Element?
    func peek() -> Element?
}

struct ArrayStack<T>: Stack {
    private var items: [T] = []
    
    mutating func push(_ element: T) {
        items.append(element)
    }
    
    mutating func pop() -> T? {
        return items.popLast()
    }
    
    func peek() -> T? {
        return items.last
    }
}

// 使用
var stack = ArrayStack<Int>()
stack.push(1)
stack.push(2)
print(stack.pop())  // Optional(2)
```

`associatedtype` 讓 Protocol 能定義泛型，這是 Java interface 做不到的（Java 的泛型在 interface 層級定義，不在方法層級）。

## POP vs OOP：何時用哪個？

**用繼承（OOP）的情況**：
- 有明確的 "is-a" 關係（UIViewController 是一種 UIResponder）
- 需要 override 父類別的方法
- 共享狀態很重要

**用 Protocol（POP）的情況**：
- 需要多重實作
- 值型別（struct）需要共享行為
- 程式碼重用但不想建立繼承關係
- 想要更鬆耦合的設計

實務上，我現在傾向**優先用 Protocol**，除非真的需要繼承。因為：
- Protocol 更靈活
- 測試更容易（可以 mock protocol）
- 不會有繼承層級過深的問題

## 學習心得

從 Java 的 OOP 轉到 Swift 的 POP，一開始不習慣。Java 的思維是「建立類別繼承體系」，Swift 的思維是「組合多個 protocol」。

現在回頭看 Java 專案，發現很多繼承其實不必要，用 interface + composition 會更好。Swift 的 POP 強化了這個觀念。

Protocol Extensions 是 Swift 的殺手級特性，讓程式碼重用變得更簡單。下次寫程式時，可以多想想「這個功能能不能用 protocol extension 解決」，往往能寫出更簡潔優雅的程式碼。
