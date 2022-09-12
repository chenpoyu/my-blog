---
layout: post
title: "iOS 效能調校：Instruments 深度分析與優化技巧"
date: 2018-08-24
categories: [iOS, Performance]
tags: [Performance, Instruments, Optimization, Profiling]
---

這週專注在 app 效能優化，深入使用 Xcode 的 Instruments 工具。效能問題在開發時不明顯，但使用者裝置上可能會卡頓、耗電、當機。Instruments 提供強大的分析工具，幫助找出效能瓶頸。

## Instruments 工具概覽

Instruments 包含多種分析工具：

- **Time Profiler**：CPU 使用率分析
- **Allocations**：記憶體分配追蹤
- **Leaks**：記憶體洩漏檢測
- **Network**：網路請求監控
- **Energy Log**：電池消耗分析
- **System Trace**：系統層級追蹤
- **Core Animation**：UI 渲染效能

## Time Profiler：找出 CPU 瓶頸

**啟動 Time Profiler**：
1. Product → Profile（⌘I）
2. 選擇 Time Profiler
3. 按下紅色錄製按鈕
4. 操作 app，特別是會卡頓的地方
5. 停止錄製並分析

**案例：優化圖片處理**

發現某個畫面載入很慢，用 Time Profiler 分析：

```swift
// 原始程式碼（慢）
func loadImages() {
    for i in 0..<100 {
        let image = UIImage(named: "image_\(i)")
        let resized = resizeImage(image!, to: CGSize(width: 100, height: 100))
        imageCache[i] = resized
    }
}

func resizeImage(_ image: UIImage, to size: CGSize) -> UIImage {
    UIGraphicsBeginImageContextWithOptions(size, false, 0.0)
    image.draw(in: CGRect(origin: .zero, size: size))
    let resizedImage = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext()
    return resizedImage ?? image
}
```

**Time Profiler 顯示**：
- `loadImages()` 佔用 85% CPU 時間
- `resizeImage()` 被呼叫 100 次，每次 30ms
- 在主執行緒執行，造成 UI 卡頓

**優化策略**：

```swift
// 1. 移到背景執行緒
func loadImages() {
    DispatchQueue.global(qos: .userInitiated).async {
        for i in 0..<100 {
            let image = UIImage(named: "image_\(i)")
            let resized = self.resizeImage(image!, to: CGSize(width: 100, height: 100))
            
            DispatchQueue.main.async {
                self.imageCache[i] = resized
            }
        }
    }
}

// 2. 用更快的縮放方法
func resizeImage(_ image: UIImage, to size: CGSize) -> UIImage {
    let renderer = UIGraphicsImageRenderer(size: size)
    return renderer.image { context in
        image.draw(in: CGRect(origin: .zero, size: size))
    }
}

// 3. 只載入需要的圖片（lazy loading）
func loadImage(at index: Int) {
    if imageCache[index] != nil { return }
    
    DispatchQueue.global(qos: .userInitiated).async {
        let image = UIImage(named: "image_\(index)")
        let resized = self.resizeImage(image!, to: CGSize(width: 100, height: 100))
        
        DispatchQueue.main.async {
            self.imageCache[index] = resized
            self.updateUI(at: index)
        }
    }
}
```

**結果**：載入時間從 3 秒降到 0.3 秒，UI 不再卡頓。

## Allocations：記憶體使用分析

**使用 Allocations 工具**：

1. 錄製一段操作
2. 查看記憶體使用趨勢
3. 用 Mark Generation 標記特定操作
4. 查看新增的物件

**案例：TableView 記憶體飆升**

發現滑動 TableView 時記憶體持續增加：

```swift
// 問題程式碼
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = UITableViewCell()  // 每次都建立新 cell！
    cell.textLabel?.text = items[indexPath.row]
    return cell
}
```

**Allocations 顯示**：
- UITableViewCell 物件持續增加
- 記憶體從 50MB 增加到 200MB

**修正**：

```swift
// 正確做法：重用 cell
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
    cell.textLabel?.text = items[indexPath.row]
    return cell
}
```

**進階分析：找出哪個物件佔用最多記憶體**

在 Allocations 中：
1. 點選 "Display Settings" → 勾選 "Created & Persistent"
2. 按照 "Persistent Bytes" 排序
3. 找出佔用記憶體最多的物件類別

```swift
// 發現大量 UIImage 佔用記憶體
class ImageCache {
    private var cache: [String: UIImage] = [:]
    
    // 問題：快取沒有上限
    func setImage(_ image: UIImage, forKey key: String) {
        cache[key] = image
    }
}

// 解決方案：用 NSCache（有自動清理機制）
class ImageCache {
    private let cache = NSCache<NSString, UIImage>()
    
    init() {
        cache.countLimit = 100  // 最多 100 個物件
        cache.totalCostLimit = 50 * 1024 * 1024  // 最多 50MB
    }
    
    func setImage(_ image: UIImage, forKey key: String) {
        cache.setObject(image, forKey: key as NSString)
    }
    
    func image(forKey key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
}
```

## Leaks：記憶體洩漏檢測

**常見的記憶體洩漏**：

1. **閉包循環參照**

```swift
// 洩漏
class ViewController: UIViewController {
    var closure: (() -> Void)?
    
    func setupClosure() {
        closure = {
            self.view.backgroundColor = .red  // self 被捕獲
        }
    }
}

// 修正
closure = { [weak self] in
    self?.view.backgroundColor = .red
}
```

2. **Delegate 沒用 weak**

```swift
// 洩漏
protocol DataFetcherDelegate {
    func didFetch(data: String)
}

class DataFetcher {
    var delegate: DataFetcherDelegate?  // 應該用 weak
}

// 修正
protocol DataFetcherDelegate: AnyObject {  // 加上 AnyObject
    func didFetch(data: String)
}

class DataFetcher {
    weak var delegate: DataFetcherDelegate?  // 用 weak
}
```

3. **Timer 沒有 invalidate**

```swift
// 洩漏
class TimerViewController: UIViewController {
    var timer: Timer?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            self?.updateTime()
        }
    }
    // ViewController 不會被釋放，因為 Timer 持有它
}

// 修正
deinit {
    timer?.invalidate()
    timer = nil
}
```

**用 Memory Graph 視覺化洩漏**：

1. 在 Xcode debug bar 點選記憶體圖示（第二個）
2. 查看物件的參照關係
3. 找出循環參照的鏈條

## UI 渲染效能優化

### 1. 避免在主執行緒做耗時操作

```swift
// 錯誤：在主執行緒載入大圖
imageView.image = UIImage(contentsOfFile: largeImagePath)

// 正確：背景載入
DispatchQueue.global(qos: .userInitiated).async {
    let image = UIImage(contentsOfFile: largeImagePath)
    DispatchQueue.main.async {
        self.imageView.image = image
    }
}
```

### 2. TableView / CollectionView 優化

```swift
// Cell 預載入（prefetching）
class MyViewController: UIViewController, UITableViewDataSourcePrefetching {
    func tableView(_ tableView: UITableView, prefetchRowsAt indexPaths: [IndexPath]) {
        for indexPath in indexPaths {
            // 預載入資料（如圖片）
            loadData(at: indexPath.row)
        }
    }
}

// 設定預估高度，避免計算所有 cell 高度
tableView.estimatedRowHeight = 100
tableView.rowHeight = UITableView.automaticDimension
```

### 3. 圖層優化

```swift
// 避免離屏渲染（Offscreen Rendering）
// 不好
view.layer.cornerRadius = 10
view.layer.masksToBounds = true  // 觸發離屏渲染

// 更好：直接用圓角圖片，或用 bezierPath
let maskPath = UIBezierPath(
    roundedRect: view.bounds,
    cornerRadius: 10
)
let maskLayer = CAShapeLayer()
maskLayer.path = maskPath.cgPath
view.layer.mask = maskLayer
```

### 4. 減少 View 層級

```swift
// 不好：巢狀太深
view
  └─ containerView
       └─ stackView
            └─ subView1
                 └─ label1
            └─ subView2
                 └─ label2

// 更好：扁平化
view
  └─ label1
  └─ label2
```

## 網路效能優化

### 1. 批次請求

```swift
// 不好：多個獨立請求
for id in userIds {
    fetchUser(id: id) { user in
        // ...
    }
}

// 更好：批次請求
fetchUsers(ids: userIds) { users in
    // ...
}
```

### 2. 快取策略

```swift
// HTTP 快取
let configuration = URLSessionConfiguration.default
configuration.requestCachePolicy = .returnCacheDataElseLoad
configuration.urlCache = URLCache(
    memoryCapacity: 20 * 1024 * 1024,  // 20MB 記憶體快取
    diskCapacity: 100 * 1024 * 1024,   // 100MB 磁碟快取
    diskPath: "URLCache"
)

let session = URLSession(configuration: configuration)
```

### 3. 壓縮回應

```swift
// 伺服器啟用 gzip 壓縮
// iOS 自動解壓縮

// 請求時指定接受壓縮
var request = URLRequest(url: url)
request.setValue("gzip, deflate", forHTTPHeaderField: "Accept-Encoding")
```

## 電池消耗優化

**Energy Log 工具分析**：

1. 執行 Energy Log
2. 查看 CPU、網路、GPS 使用情況
3. 找出耗電熱點

**優化策略**：

```swift
// 1. 降低定位精度
locationManager.desiredAccuracy = kCLLocationAccuracyHundredMeters  // 而非 Best

// 2. 減少定位頻率
locationManager.distanceFilter = 100  // 移動 100 公尺才更新

// 3. 背景任務用適當的 QoS
DispatchQueue.global(qos: .background).async {
    // 不緊急的任務
}

// 4. 批次處理，而非持續執行
// 不好：持續監聽
timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { _ in
    checkSomething()
}

// 更好：需要時才檢查
func checkWhenNeeded() {
    checkSomething()
}
```

## 效能測試的最佳實踐

1. **在真實裝置上測試**：模擬器效能不代表實際裝置
2. **測試低階裝置**：如 iPhone 6、iPhone SE
3. **測試差網路環境**：用 Network Link Conditioner 模擬慢網路
4. **測試低電量狀態**：iOS 會啟動低電量模式
5. **長時間測試**：記憶體洩漏可能要一段時間才顯現

## 與 Java 的效能工具對比

**Java**：
- JProfiler、VisualVM：效能分析
- JMeter：負載測試
- GC 日誌：記憶體分析

**iOS**：
- Instruments：效能分析
- XCTest：單元測試和 UI 測試
- Xcode Debugger：記憶體圖譜

Instruments 整合更緊密，但 Java 的工具生態系更成熟。

## 學習心得

效能優化是持續的過程，不是一次性的任務。Instruments 提供強大的工具，但關鍵是**理解效能問題的根源**。

常見誤區：
- 過早優化：先讓程式運作，再優化瓶頸
- 忽略測試：在模擬器上看起來快，不代表裝置上快
- 只優化 CPU：電池消耗、網路延遲也很重要

重點是**測量、分析、優化、再測量**，用數據驅動決策，而非憑感覺。
