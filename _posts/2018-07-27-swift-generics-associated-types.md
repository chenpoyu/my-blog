---
layout: post
title: "Swift 泛型與關聯型別：型別安全的抽象化"
date: 2018-07-27
categories: [iOS, Swift]
tags: [Swift, Generics, Associated Types, Type Safety]
---

這週研究 Swift 的泛型系統，發現比 Java 的泛型更強大且靈活。特別是關聯型別（Associated Types）和泛型約束，能實現高度抽象化同時保持型別安全。

## Swift 泛型 vs Java 泛型

先看基本的泛型函式比較：

**Java**：

```java
public <T> T findFirst(List<T> list) {
    if (list.isEmpty()) {
        return null;
    }
    return list.get(0);
}
```

**Swift**：

```swift
func findFirst<T>(_ array: [T]) -> T? {
    return array.first
}
```

語法類似，但 Swift 的 Optional 讓回傳值的語意更清楚：`T?` 明確表示可能沒有值。

## 泛型型別：Stack 的實作

**基本的泛型 struct**：

```swift
struct Stack<Element> {
    private var items: [Element] = []
    
    mutating func push(_ item: Element) {
        items.append(item)
    }
    
    mutating func pop() -> Element? {
        return items.popLast()
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

// 使用
var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)
print(intStack.pop())  // Optional(2)

var stringStack = Stack<String>()
stringStack.push("Hello")
stringStack.push("World")
```

這和 Java 的 `Stack<T>` 類似，但 Swift 的 struct 是值型別，複製時不會共享狀態。

## 泛型約束：限制型別

有時需要限制泛型參數必須符合特定條件：

```swift
// 只接受遵循 Equatable 的型別
func findIndex<T: Equatable>(of valueToFind: T, in array: [T]) -> Int? {
    for (index, value) in array.enumerated() {
        if value == valueToFind {  // 需要 Equatable 才能用 ==
            return index
        }
    }
    return nil
}

let numbers = [1, 2, 3, 4, 5]
print(findIndex(of: 3, in: numbers))  // Optional(2)

let names = ["Alice", "Bob", "Charlie"]
print(findIndex(of: "Bob", in: names))  // Optional(1)
```

**多重約束**：

```swift
func compare<T: Comparable & CustomStringConvertible>(_ a: T, _ b: T) -> String {
    if a < b {
        return "\(a.description) is less than \(b.description)"
    } else if a > b {
        return "\(a.description) is greater than \(b.description)"
    } else {
        return "\(a.description) equals \(b.description)"
    }
}

print(compare(5, 10))  // "5 is less than 10"
```

**where 子句**：

```swift
func allEqual<T: Equatable>(_ array: [T]) -> Bool {
    guard let first = array.first else { return true }
    return array.allSatisfy { $0 == first }
}

// 或用 where 子句
func allEqual<T>(_ array: [T]) -> Bool where T: Equatable {
    guard let first = array.first else { return true }
    return array.allSatisfy { $0 == first }
}
```

這比 Java 的 `<T extends Comparable<T>>` 更簡潔易讀。

## 關聯型別（Associated Types）：Protocol 中的泛型

這是 Swift 獨有的特性，Java 沒有對應的概念。

**問題場景**：想定義一個 Container protocol，但不知道元素型別

```swift
// 錯誤的做法：protocol 不能直接用泛型參數
protocol Container<Element> {  // 這樣不行！
    var items: [Element] { get }
}
```

**正確的做法：用 Associated Type**

```swift
protocol Container {
    associatedtype Item  // 關聯型別
    var items: [Item] { get }
    mutating func append(_ item: Item)
}

struct IntContainer: Container {
    typealias Item = Int  // 明確指定（也可以讓編譯器推斷）
    var items: [Int] = []
    
    mutating func append(_ item: Int) {
        items.append(item)
    }
}

struct StringContainer: Container {
    // 讓編譯器推斷 Item = String
    var items: [String] = []
    
    mutating func append(_ item: String) {
        items.append(item)
    }
}
```

**為什麼需要 Associated Types？**

因為 protocol 定義的是「行為的契約」，但具體的型別由遵循 protocol 的型別決定。這讓 protocol 更靈活。

## 泛型 + Associated Types 的實戰

**案例：可搜尋的資料源**

```swift
protocol Searchable {
    associatedtype Element
    var elements: [Element] { get }
    func search(matching: (Element) -> Bool) -> [Element]
}

extension Searchable {
    // 提供預設實作
    func search(matching predicate: (Element) -> Bool) -> [Element] {
        return elements.filter(predicate)
    }
}

struct UserDatabase: Searchable {
    var elements: [User]
    
    // 不需要實作 search，會使用預設實作
}

struct ProductCatalog: Searchable {
    var elements: [Product]
    
    // 也可以客製化實作
    func search(matching predicate: (Product) -> Bool) -> [Product] {
        // 自己的搜尋邏輯，比如加上快取
        return elements.filter(predicate)
    }
}

struct User {
    let id: Int
    let name: String
}

struct Product {
    let id: Int
    let name: String
    let price: Double
}

// 使用
var userDB = UserDatabase(elements: [
    User(id: 1, name: "Alice"),
    User(id: 2, name: "Bob")
])

let result = userDB.search { $0.name.contains("A") }
print(result)  // [User(id: 1, name: "Alice")]
```

**案例：資料轉換管線**

```swift
protocol Transformer {
    associatedtype Input
    associatedtype Output
    func transform(_ input: Input) -> Output
}

struct StringToIntTransformer: Transformer {
    func transform(_ input: String) -> Int {
        return Int(input) ?? 0
    }
}

struct IntToStringTransformer: Transformer {
    func transform(_ input: Int) -> String {
        return "\(input)"
    }
}

// 組合多個 Transformer
func chain<T1: Transformer, T2: Transformer>(_ t1: T1, _ t2: T2) -> (T1.Input) -> T2.Output
    where T1.Output == T2.Input {
    return { input in
        let intermediate = t1.transform(input)
        return t2.transform(intermediate)
    }
}

let stringToInt = StringToIntTransformer()
let intToString = IntToStringTransformer()
let roundTrip = chain(stringToInt, intToString)

print(roundTrip("42"))  // "42"
```

這個 `where T1.Output == T2.Input` 約束確保兩個 transformer 能串接，型別完全安全。

## 泛型的效能考量

Swift 的泛型是**編譯時期特化**（compile-time specialization），不像 Java 的泛型有型別擦除（type erasure）。

**Java 的型別擦除**：

```java
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();
// 執行時期都變成 List<Object>，型別資訊遺失
```

**Swift 的特化**：

```swift
func printValue<T>(_ value: T) {
    print(value)
}

printValue("Hello")  // 編譯器產生 printValue_String 版本
printValue(42)  // 編譯器產生 printValue_Int 版本
```

**優點**：
- 執行時期效能好，沒有 boxing/unboxing
- 保留完整的型別資訊

**缺點**：
- 編譯時間更長
- 二進位檔案更大（每個特化版本都有程式碼）

**最佳化**：可以用 `@_specialize` 提示編譯器特化特定型別

```swift
@_specialize(exported: true, where T == Int)
@_specialize(exported: true, where T == String)
func process<T>(_ value: T) {
    // ...
}
```

## 泛型的實戰應用

### 1. 網路層抽象化

```swift
protocol APIRequest {
    associatedtype Response: Decodable
    var path: String { get }
}

class APIClient {
    func send<T: APIRequest>(_ request: T, completion: @escaping (Result<T.Response, Error>) -> Void) {
        let url = URL(string: "https://api.example.com\(request.path)")!
        
        URLSession.shared.dataTask(with: url) { data, _, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NSError(domain: "No data", code: 0)))
                return
            }
            
            do {
                let response = try JSONDecoder().decode(T.Response.self, from: data)
                completion(.success(response))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}

struct GetUserRequest: APIRequest {
    typealias Response = User
    var path: String { "/users/\(userId)" }
    let userId: Int
}

struct User: Decodable {
    let id: Int
    let name: String
}

// 使用
let client = APIClient()
let request = GetUserRequest(userId: 123)
client.send(request) { result in
    switch result {
    case .success(let user):
        print(user.name)
    case .failure(let error):
        print(error)
    }
}
```

這個設計型別完全安全，編譯時期就知道回應型別。

### 2. 資料持久化層

```swift
protocol Repository {
    associatedtype Entity
    func save(_ entity: Entity) throws
    func fetch(id: String) throws -> Entity?
    func fetchAll() throws -> [Entity]
    func delete(id: String) throws
}

class UserDefaultsRepository<T: Codable>: Repository {
    typealias Entity = T
    
    private let key: String
    
    init(key: String) {
        self.key = key
    }
    
    func save(_ entity: T) throws {
        let data = try JSONEncoder().encode(entity)
        UserDefaults.standard.set(data, forKey: key)
    }
    
    func fetch(id: String) throws -> T? {
        guard let data = UserDefaults.standard.data(forKey: "\(key)_\(id)") else {
            return nil
        }
        return try JSONDecoder().decode(T.self, from: data)
    }
    
    func fetchAll() throws -> [T] {
        // 實作略
        return []
    }
    
    func delete(id: String) throws {
        UserDefaults.standard.removeObject(forKey: "\(key)_\(id)")
    }
}

// 使用
let userRepo = UserDefaultsRepository<User>(key: "users")
try userRepo.save(User(id: 1, name: "Alice"))
let user = try userRepo.fetch(id: "1")
```

同一套 Repository 介面可以用於不同的儲存後端（UserDefaults、Core Data、Realm 等）。

## 學習心得

Swift 的泛型比 Java 更強大，主要是因為：

1. **Associated Types**：讓 protocol 能定義泛型行為
2. **Protocol Extensions**：提供預設實作，減少重複程式碼
3. **編譯時期特化**：效能好且保留型別資訊
4. **where 子句**：表達複雜的型別約束

一開始覺得 Associated Types 很難理解，和 Java 的思維不同。但理解後發現這是 Swift POP（Protocol-Oriented Programming）的核心，能寫出非常優雅且型別安全的抽象化程式碼。

重點是**多用泛型來減少型別轉換和重複程式碼**，同時保持編譯時期的型別安全。這是 Swift 比 Objective-C 進步很多的地方。
