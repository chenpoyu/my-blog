---
layout: post
title: "iOS 非同步程式設計：GCD 到 Operation Queue"
date: 2018-08-03
categories: [iOS, Concurrency]
tags: [GCD, Operation Queue, Threading, Async]
---

這週深入研究 iOS 的非同步程式設計，從 Grand Central Dispatch (GCD) 到 Operation Queue，理解如何有效管理多執行緒任務。這和 Java 的 ExecutorService、CompletableFuture 有相似之處，但也有不同的設計哲學。

## GCD：低階但強大的並行 API

**基本的 Dispatch Queue**：

```swift
// 主執行緒（Main Queue）
DispatchQueue.main.async {
    // 更新 UI 必須在主執行緒
    self.label.text = "Updated"
}

// 背景執行緒（Global Queue）
DispatchQueue.global().async {
    // 耗時操作
    let result = performHeavyComputation()
    
    // 回到主執行緒更新 UI
    DispatchQueue.main.async {
        self.updateUI(result)
    }
}
```

**QoS（Quality of Service）優先級**：

```swift
// 使用者互動（最高優先級）
DispatchQueue.global(qos: .userInteractive).async {
    // 立即需要的任務，如動畫
}

// 使用者初始化
DispatchQueue.global(qos: .userInitiated).async {
    // 使用者等待的任務，如載入資料
}

// 實用工具
DispatchQueue.global(qos: .utility).async {
    // 長時間執行的任務，如下載
}

// 背景
DispatchQueue.global(qos: .background).async {
    // 不緊急的任務，如同步
}
```

這類似 Java 的 ThreadPoolExecutor 優先級設定。

**自訂 Queue**：

```swift
// 序列佇列（Serial Queue）：任務依序執行
let serialQueue = DispatchQueue(label: "com.example.serial")
serialQueue.async {
    print("Task 1")
}
serialQueue.async {
    print("Task 2")  // 等 Task 1 完成後執行
}

// 並行佇列（Concurrent Queue）：任務同時執行
let concurrentQueue = DispatchQueue(label: "com.example.concurrent", attributes: .concurrent)
concurrentQueue.async {
    print("Task 1")
}
concurrentQueue.async {
    print("Task 2")  // 可能和 Task 1 同時執行
}
```

## DispatchGroup：等待多個任務完成

當需要等待多個非同步任務都完成時，用 DispatchGroup：

```swift
let group = DispatchGroup()

// 方法 1：enter/leave
group.enter()
fetchUserData { user in
    print("User: \(user)")
    group.leave()
}

group.enter()
fetchPosts { posts in
    print("Posts: \(posts)")
    group.leave()
}

// 等待所有任務完成
group.notify(queue: .main) {
    print("All tasks completed")
    self.updateUI()
}

// 或同步等待（會阻塞執行緒）
group.wait()  // 不建議在主執行緒使用
```

**方法 2：使用 async 的 group 參數**：

```swift
let group = DispatchGroup()
let queue = DispatchQueue.global()

queue.async(group: group) {
    // Task 1
    sleep(1)
    print("Task 1 done")
}

queue.async(group: group) {
    // Task 2
    sleep(2)
    print("Task 2 done")
}

group.notify(queue: .main) {
    print("All done")
}
```

這類似 Java 的 `CompletableFuture.allOf()` 或 `CountDownLatch`。

## DispatchSemaphore：控制並行數量

限制同時執行的任務數量：

```swift
let semaphore = DispatchSemaphore(value: 3)  // 最多 3 個並行

for i in 1...10 {
    DispatchQueue.global().async {
        semaphore.wait()  // 等待信號量
        
        print("Task \(i) started")
        sleep(2)  // 模擬耗時操作
        print("Task \(i) finished")
        
        semaphore.signal()  // 釋放信號量
    }
}
```

這類似 Java 的 `Semaphore`。

## DispatchWorkItem：可取消的任務

```swift
let workItem = DispatchWorkItem {
    for i in 1...10 {
        guard !Thread.current.isCancelled else {
            print("Task cancelled")
            return
        }
        print("Working... \(i)")
        sleep(1)
    }
}

DispatchQueue.global().async(execute: workItem)

// 5 秒後取消
DispatchQueue.global().asyncAfter(deadline: .now() + 5) {
    workItem.cancel()
}
```

## Dispatch Barrier：讀寫鎖

在並行佇列中實現執行緒安全的讀寫：

```swift
class SafeDictionary<Key: Hashable, Value> {
    private var dictionary: [Key: Value] = [:]
    private let queue = DispatchQueue(label: "SafeDictionary", attributes: .concurrent)
    
    // 讀取（並行）
    func value(forKey key: Key) -> Value? {
        var result: Value?
        queue.sync {
            result = dictionary[key]
        }
        return result
    }
    
    // 寫入（串行，用 barrier 阻擋其他讀寫）
    func setValue(_ value: Value?, forKey key: Key) {
        queue.async(flags: .barrier) {
            self.dictionary[key] = value
        }
    }
}
```

Barrier 確保寫入時沒有其他讀取或寫入在進行，類似讀寫鎖。

## Operation 和 OperationQueue：高階抽象

GCD 很強大但較低階，Operation 提供更高階的抽象：

**基本使用**：

```swift
let queue = OperationQueue()

// BlockOperation
let operation1 = BlockOperation {
    print("Operation 1")
    sleep(1)
}

let operation2 = BlockOperation {
    print("Operation 2")
    sleep(1)
}

// 設定依賴關係
operation2.addDependency(operation1)  // operation2 等 operation1 完成

queue.addOperation(operation1)
queue.addOperation(operation2)
```

**自訂 Operation**：

```swift
class ImageDownloadOperation: Operation {
    let url: URL
    var image: UIImage?
    
    init(url: URL) {
        self.url = url
    }
    
    override func main() {
        if isCancelled { return }
        
        guard let data = try? Data(contentsOf: url) else { return }
        
        if isCancelled { return }
        
        self.image = UIImage(data: data)
    }
}

// 使用
let operation = ImageDownloadOperation(url: imageURL)
operation.completionBlock = {
    if let image = operation.image {
        DispatchQueue.main.async {
            self.imageView.image = image
        }
    }
}

let queue = OperationQueue()
queue.addOperation(operation)

// 可以隨時取消
operation.cancel()
```

**Operation 的優點**：
- 可以取消
- 有依賴關係
- 有 KVO 可觀察狀態
- 可以設定優先級
- 可以暫停和繼續

**GCD vs Operation**：

| 特性 | GCD | Operation |
|------|-----|-----------|
| 效能 | 更快（低階） | 稍慢（高階封裝） |
| 取消 | 需要手動實作 | 內建支援 |
| 依賴 | 需用 DispatchGroup | 內建支援 |
| KVO | 不支援 | 支援 |
| 重用 | 困難 | 容易（繼承 Operation） |

## 實戰案例：圖片下載管理器

結合 Operation 和 Cache 實作圖片下載管理：

```swift
class ImageDownloadManager {
    static let shared = ImageDownloadManager()
    
    private let queue: OperationQueue = {
        let queue = OperationQueue()
        queue.maxConcurrentOperationCount = 4  // 最多同時下載 4 張
        return queue
    }()
    
    private let cache = NSCache<NSString, UIImage>()
    private var operations: [URL: ImageDownloadOperation] = [:]
    
    func downloadImage(from url: URL, completion: @escaping (UIImage?) -> Void) {
        // 檢查快取
        if let cachedImage = cache.object(forKey: url.absoluteString as NSString) {
            completion(cachedImage)
            return
        }
        
        // 檢查是否已在下載
        if let existingOperation = operations[url] {
            existingOperation.completionBlock = {
                completion(existingOperation.image)
            }
            return
        }
        
        // 建立新的下載操作
        let operation = ImageDownloadOperation(url: url)
        operation.completionBlock = { [weak self] in
            if let image = operation.image {
                self?.cache.setObject(image, forKey: url.absoluteString as NSString)
            }
            completion(operation.image)
            self?.operations.removeValue(forKey: url)
        }
        
        operations[url] = operation
        queue.addOperation(operation)
    }
    
    func cancelDownload(for url: URL) {
        operations[url]?.cancel()
        operations.removeValue(forKey: url)
    }
}

// 使用
ImageDownloadManager.shared.downloadImage(from: imageURL) { image in
    DispatchQueue.main.async {
        self.imageView.image = image
    }
}
```

## 常見陷阱和最佳實踐

### 1. 主執行緒死鎖

```swift
// 錯誤：在主執行緒同步呼叫主執行緒
DispatchQueue.main.sync {  // 死鎖！
    print("Never executed")
}

// 正確：用 async
DispatchQueue.main.async {
    print("Executed")
}
```

### 2. 強參照循環

```swift
// 錯誤：閉包捕獲 self
DispatchQueue.global().async {
    self.updateData()  // 可能造成循環參照
}

// 正確：用 [weak self]
DispatchQueue.global().async { [weak self] in
    self?.updateData()
}
```

### 3. UI 更新必須在主執行緒

```swift
URLSession.shared.dataTask(with: url) { data, _, error in
    guard let data = data else { return }
    let image = UIImage(data: data)
    
    // 錯誤：在背景執行緒更新 UI
    self.imageView.image = image  // 可能崩潰或行為異常
    
    // 正確：回到主執行緒
    DispatchQueue.main.async {
        self.imageView.image = image
    }
}.resume()
```

### 4. 避免過度並行

```swift
// 錯誤：建立太多執行緒
for i in 1...10000 {
    DispatchQueue.global().async {
        // 10000 個任務同時執行
    }
}

// 正確：限制並行數量
let operationQueue = OperationQueue()
operationQueue.maxConcurrentOperationCount = 4

for i in 1...10000 {
    operationQueue.addOperation {
        // 最多 4 個任務同時執行
    }
}
```

## 與 Java 的對比

**Java ExecutorService**：

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

executor.submit(() -> {
    // Task
});

executor.shutdown();
```

**Swift OperationQueue**：

```swift
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 4

queue.addOperation {
    // Task
}

// 不需要手動 shutdown，ARC 會處理
```

**Java CompletableFuture**：

```java
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> processData(data))
    .thenAcceptAsync(result -> updateUI(result), mainThreadExecutor);
```

**Swift GCD**：

```swift
DispatchQueue.global().async {
    let data = fetchData()
    let result = processData(data)
    
    DispatchQueue.main.async {
        updateUI(result)
    }
}
```

Swift 的語法更簡潔，但 Java 的 CompletableFuture 鏈式呼叫更優雅。

## 學習心得

iOS 的並行程式設計相對簡單，GCD 提供直觀的 API，不用自己管理執行緒。但還是要注意：
- 主執行緒死鎖
- UI 更新必須在主執行緒
- 避免過度並行
- 記憶體管理（[weak self]）

與 Java 相比，iOS 的並行 API 更輕量，適合行動裝置的特性。OperationQueue 的依賴關係和取消功能在複雜場景下很有用。

下週繼續研究 Core Data 的進階應用，看看如何處理複雜的資料關聯和查詢優化。
