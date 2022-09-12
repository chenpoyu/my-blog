---
layout: post
title: "iOS 多執行緒與 GCD"
date: 2018-04-06
categories: [Swift, iOS]
tags: [iOS, GCD, Concurrency, Threading]
---

這週深入學習 iOS 的多執行緒程式設計。在 Java 開發中，我們用 Thread、ExecutorService、synchronized 等來處理並行。iOS 的做法不太一樣，但核心概念相通。

iOS 的多執行緒解決方案主要有兩個：
- **Grand Central Dispatch (GCD)**：Apple 推薦的方式，使用 dispatch queue
- **Operation Queue**：高階封裝，使用 NSOperation

兩者的關係類似 Java 的 Thread 和 ExecutorService：GCD 是底層、輕量級，Operation Queue 是高階、功能豐富。

最大的差異是 iOS 強制要求 **UI 更新必須在主執行緒**。Java Swing 也有 EDT (Event Dispatch Thread) 的概念，但沒有這麼嚴格。iOS 如果在背景執行緒更新 UI，會造成不可預期的行為甚至 crash。

## 多執行緒基礎

iOS 應用的主執行緒 (Main Thread) 負責 UI 更新和使用者互動。耗時操作應放在背景執行緒執行，避免阻塞 UI。

### 執行緒與佇列

```
Main Thread (主執行緒):
- 處理 UI 更新
- 回應使用者互動
- 必須保持流暢

Background Thread (背景執行緒):
- 網路請求
- 檔案 I/O
- 資料處理
- 複雜計算
```

## Grand Central Dispatch (GCD)

GCD 是 Apple 的多執行緒解決方案，比直接使用執行緒更簡單且高效。

### Dispatch Queue

```swift
// 主佇列 - 在主執行緒執行
DispatchQueue.main.async {
    // 更新 UI
    self.label.text = "Updated"
}

// 全域佇列 - 在背景執行緒執行
DispatchQueue.global().async {
    // 耗時操作
    let data = self.processData()
    
    // 回到主執行緒更新 UI
    DispatchQueue.main.async {
        self.updateUI(with: data)
    }
}
```

### QoS (Quality of Service)

指定任務優先級：

```swift
// User Interactive - 最高優先級，立即執行
DispatchQueue.global(qos: .userInteractive).async {
    // UI 動畫、即時互動
}

// User Initiated - 使用者發起的任務
DispatchQueue.global(qos: .userInitiated).async {
    // 使用者點擊載入資料
}

// Default - 預設優先級
DispatchQueue.global(qos: .default).async {
    // 一般任務
}

// Utility - 長時間任務，使用者可見進度
DispatchQueue.global(qos: .utility).async {
    // 下載、匯入資料
}

// Background - 最低優先級，使用者不可見
DispatchQueue.global(qos: .background).async {
    // 清理快取、資料同步
}
```

### 同步 vs 非同步

```swift
// sync - 同步執行，會阻塞當前執行緒
DispatchQueue.global().sync {
    print("Task 1")
}
print("Task 2")  // 等 Task 1 完成才執行

// async - 非同步執行，不阻塞當前執行緒
DispatchQueue.global().async {
    print("Task 1")
}
print("Task 2")  // 立即執行，不等 Task 1
```

### 自訂佇列

```swift
// Serial Queue - 任務依序執行
let serialQueue = DispatchQueue(label: "com.example.serialQueue")
serialQueue.async {
    print("Task 1")
}
serialQueue.async {
    print("Task 2")  // 等 Task 1 完成
}

// Concurrent Queue - 任務並行執行
let concurrentQueue = DispatchQueue(label: "com.example.concurrentQueue",
                                   attributes: .concurrent)
concurrentQueue.async {
    print("Task 1")
}
concurrentQueue.async {
    print("Task 2")  // 可能與 Task 1 同時執行
}
```

## 實際應用場景

### 網路請求

```swift
class DataLoader {
    func loadData(completion: @escaping (String) -> Void) {
        DispatchQueue.global(qos: .userInitiated).async {
            // 模擬網路請求
            Thread.sleep(forTimeInterval: 2)
            let data = "Loaded data"
            
            DispatchQueue.main.async {
                completion(data)
            }
        }
    }
}

class ViewController: UIViewController {
    let label = UILabel()
    let loader = DataLoader()
    
    func loadData() {
        label.text = "Loading..."
        
        loader.loadData { data in
            self.label.text = data
        }
    }
}
```

### 圖片處理

```swift
func processImage(_ image: UIImage, completion: @escaping (UIImage?) -> Void) {
    DispatchQueue.global(qos: .userInitiated).async {
        // 耗時的圖片處理
        guard let ciImage = CIImage(image: image) else {
            DispatchQueue.main.async {
                completion(nil)
            }
            return
        }
        
        let filter = CIFilter(name: "CISepiaTone")
        filter?.setValue(ciImage, forKey: kCIInputImageKey)
        filter?.setValue(0.8, forKey: kCIInputIntensityKey)
        
        guard let outputImage = filter?.outputImage,
              let cgImage = CIContext().createCGImage(outputImage, from: outputImage.extent) else {
            DispatchQueue.main.async {
                completion(nil)
            }
            return
        }
        
        let processedImage = UIImage(cgImage: cgImage)
        
        DispatchQueue.main.async {
            completion(processedImage)
        }
    }
}
```

### 資料庫操作

```swift
class DatabaseManager {
    private let queue = DispatchQueue(label: "com.example.database", qos: .utility)
    
    func saveData(_ data: String, completion: @escaping (Bool) -> Void) {
        queue.async {
            // 模擬資料庫寫入
            Thread.sleep(forTimeInterval: 1)
            let success = true
            
            DispatchQueue.main.async {
                completion(success)
            }
        }
    }
    
    func fetchData(completion: @escaping ([String]) -> Void) {
        queue.async {
            // 模擬資料庫讀取
            Thread.sleep(forTimeInterval: 0.5)
            let data = ["Item 1", "Item 2", "Item 3"]
            
            DispatchQueue.main.async {
                completion(data)
            }
        }
    }
}
```

## Dispatch Group

管理多個非同步任務，等待全部完成。

```swift
func loadMultipleResources(completion: @escaping () -> Void) {
    let group = DispatchGroup()
    
    // 任務 1
    group.enter()
    loadUserData {
        print("User data loaded")
        group.leave()
    }
    
    // 任務 2
    group.enter()
    loadSettings {
        print("Settings loaded")
        group.leave()
    }
    
    // 任務 3
    group.enter()
    loadNotifications {
        print("Notifications loaded")
        group.leave()
    }
    
    // 所有任務完成後執行
    group.notify(queue: .main) {
        print("All resources loaded")
        completion()
    }
}

// 或使用 wait (會阻塞)
func loadAndWait() {
    let group = DispatchGroup()
    
    group.enter()
    loadData {
        group.leave()
    }
    
    group.wait()  // 阻塞當前執行緒，等待完成
    print("Data loaded")
}
```

### Dispatch Group 批次處理

```swift
func downloadImages(urls: [String], completion: @escaping ([UIImage]) -> Void) {
    var images: [UIImage] = []
    let group = DispatchGroup()
    let queue = DispatchQueue(label: "com.example.imageDownload", attributes: .concurrent)
    
    for url in urls {
        group.enter()
        queue.async {
            if let image = self.downloadImage(from: url) {
                images.append(image)
            }
            group.leave()
        }
    }
    
    group.notify(queue: .main) {
        completion(images)
    }
}
```

## Dispatch Semaphore

控制並行數量：

```swift
class ConcurrentDownloader {
    private let semaphore = DispatchSemaphore(value: 3)  // 最多 3 個並行
    
    func downloadFiles(_ urls: [String]) {
        for url in urls {
            DispatchQueue.global().async {
                self.semaphore.wait()  // 等待可用名額
                
                self.downloadFile(url) {
                    self.semaphore.signal()  // 釋放名額
                }
            }
        }
    }
    
    func downloadFile(_ url: String, completion: @escaping () -> Void) {
        print("Downloading: \(url)")
        Thread.sleep(forTimeInterval: 2)
        print("Downloaded: \(url)")
        completion()
    }
}
```

## Dispatch Barrier

在並行佇列中建立同步點：

```swift
class ThreadSafeArray {
    private var array: [Int] = []
    private let queue = DispatchQueue(label: "com.example.array", attributes: .concurrent)
    
    func append(_ value: Int) {
        queue.async(flags: .barrier) {
            self.array.append(value)
        }
    }
    
    func read(at index: Int) -> Int? {
        var result: Int?
        queue.sync {
            if index < array.count {
                result = array[index]
            }
        }
        return result
    }
    
    func getAll() -> [Int] {
        var result: [Int] = []
        queue.sync {
            result = array
        }
        return result
    }
}

let safeArray = ThreadSafeArray()

// 多個執行緒可以同時讀取
DispatchQueue.global().async {
    print(safeArray.getAll())
}

// 寫入時會等待所有讀取完成，並阻止新的讀取
safeArray.append(1)
```

## DispatchWorkItem

封裝可取消的任務：

```swift
class SearchViewController: UIViewController {
    var searchWorkItem: DispatchWorkItem?
    
    func searchTextDidChange(_ text: String) {
        // 取消先前的搜尋
        searchWorkItem?.cancel()
        
        // 建立新的搜尋任務
        let workItem = DispatchWorkItem { [weak self] in
            self?.performSearch(text)
        }
        
        searchWorkItem = workItem
        
        // 延遲 0.5 秒執行
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5, execute: workItem)
    }
    
    func performSearch(_ text: String) {
        print("Searching for: \(text)")
        // 執行搜尋
    }
}
```

## DispatchSource

監控系統事件：

```swift
class FileMonitor {
    private var source: DispatchSourceFileSystemObject?
    
    func monitorFile(at url: URL) {
        let fileDescriptor = open(url.path, O_EVTONLY)
        
        source = DispatchSource.makeFileSystemObjectSource(
            fileDescriptor: fileDescriptor,
            eventMask: .write,
            queue: DispatchQueue.global()
        )
        
        source?.setEventHandler { [weak self] in
            print("File was modified")
            self?.handleFileChange()
        }
        
        source?.setCancelHandler {
            close(fileDescriptor)
        }
        
        source?.resume()
    }
    
    func stopMonitoring() {
        source?.cancel()
    }
    
    func handleFileChange() {
        // 處理檔案變更
    }
}
```

## Operation 和 OperationQueue

更高階的並行 API，支援依賴關係和取消。

### 基本使用

```swift
import Foundation

let queue = OperationQueue()
queue.maxConcurrentOperationCount = 2

// BlockOperation
let operation1 = BlockOperation {
    print("Operation 1")
    Thread.sleep(forTimeInterval: 1)
}

let operation2 = BlockOperation {
    print("Operation 2")
    Thread.sleep(forTimeInterval: 1)
}

queue.addOperations([operation1, operation2], waitUntilFinished: false)
```

### 自訂 Operation

```swift
class DownloadOperation: Operation {
    let url: String
    
    init(url: String) {
        self.url = url
        super.init()
    }
    
    override func main() {
        if isCancelled { return }
        
        print("Downloading: \(url)")
        Thread.sleep(forTimeInterval: 2)
        
        if isCancelled { return }
        
        print("Downloaded: \(url)")
    }
}

let downloadQueue = OperationQueue()
let operation = DownloadOperation(url: "https://example.com/file")
downloadQueue.addOperation(operation)

// 取消操作
operation.cancel()
```

### Operation 依賴

```swift
let operation1 = BlockOperation {
    print("Download image")
    Thread.sleep(forTimeInterval: 1)
}

let operation2 = BlockOperation {
    print("Apply filter")
    Thread.sleep(forTimeInterval: 1)
}

let operation3 = BlockOperation {
    print("Save image")
}

// operation2 依賴 operation1
operation2.addDependency(operation1)

// operation3 依賴 operation2
operation3.addDependency(operation2)

let queue = OperationQueue()
queue.addOperations([operation1, operation2, operation3], waitUntilFinished: false)

// 執行順序: operation1 -> operation2 -> operation3
```

## 執行緒安全

### 競爭條件 (Race Condition)

```swift
// 不安全的程式碼
class Counter {
    var count = 0
    
    func increment() {
        count += 1  // 多執行緒同時執行可能出錯
    }
}

// 使用佇列保證執行緒安全
class SafeCounter {
    private var count = 0
    private let queue = DispatchQueue(label: "com.example.counter")
    
    func increment() {
        queue.async {
            self.count += 1
        }
    }
    
    func getCount() -> Int {
        return queue.sync {
            return count
        }
    }
}
```

### 使用鎖

```swift
import Foundation

class LockedCounter {
    private var count = 0
    private let lock = NSLock()
    
    func increment() {
        lock.lock()
        count += 1
        lock.unlock()
    }
    
    func getCount() -> Int {
        lock.lock()
        defer { lock.unlock() }
        return count
    }
}
```

## 實作圖片下載器

```swift
class ImageDownloader {
    static let shared = ImageDownloader()
    
    private let cache = NSCache<NSString, UIImage>()
    private let downloadQueue = DispatchQueue(label: "com.example.imageDownload", attributes: .concurrent)
    private var activeDownloads: [String: DispatchWorkItem] = [:]
    
    func downloadImage(from urlString: String, completion: @escaping (UIImage?) -> Void) {
        // 檢查快取
        if let cachedImage = cache.object(forKey: urlString as NSString) {
            DispatchQueue.main.async {
                completion(cachedImage)
            }
            return
        }
        
        // 檢查是否正在下載
        if activeDownloads[urlString] != nil {
            return
        }
        
        guard let url = URL(string: urlString) else {
            DispatchQueue.main.async {
                completion(nil)
            }
            return
        }
        
        let workItem = DispatchWorkItem { [weak self] in
            guard let self = self else { return }
            
            if let data = try? Data(contentsOf: url),
               let image = UIImage(data: data) {
                // 儲存到快取
                self.cache.setObject(image, forKey: urlString as NSString)
                
                DispatchQueue.main.async {
                    completion(image)
                }
            } else {
                DispatchQueue.main.async {
                    completion(nil)
                }
            }
            
            // 移除活動下載
            self.activeDownloads.removeValue(forKey: urlString)
        }
        
        activeDownloads[urlString] = workItem
        downloadQueue.async(execute: workItem)
    }
    
    func cancelDownload(for urlString: String) {
        activeDownloads[urlString]?.cancel()
        activeDownloads.removeValue(forKey: urlString)
    }
}
```

## 最佳實踐

```swift
// 1. 永遠在主執行緒更新 UI
DispatchQueue.main.async {
    self.label.text = "Updated"
}

// 2. 避免在主執行緒執行耗時操作
DispatchQueue.global().async {
    let data = self.loadData()
    DispatchQueue.main.async {
        self.updateUI(with: data)
    }
}

// 3. 使用合適的 QoS
DispatchQueue.global(qos: .userInitiated).async {
    // 使用者發起的任務
}

// 4. 避免過度巢狀
// 不好的做法
DispatchQueue.global().async {
    DispatchQueue.main.async {
        DispatchQueue.global().async {
            // 太多層巢狀
        }
    }
}

// 5. 小心 [weak self]
DispatchQueue.global().async { [weak self] in
    guard let self = self else { return }
    // 使用 self
}
```

## 小結

GCD 提供強大的多執行緒能力，讓開發者專注於任務而非執行緒管理。主執行緒處理 UI，背景執行緒處理耗時操作是基本原則。Dispatch Group 管理多任務，Semaphore 控制並行數，Barrier 保證執行緒安全。理解並正確使用這些工具能大幅提升應用效能和使用者體驗。

下週將學習動畫和轉場效果，讓應用更生動。
