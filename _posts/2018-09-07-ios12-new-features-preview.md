---
layout: post
title: "iOS 12 新特性與 SwiftUI 預覽"
date: 2018-09-07
categories: [iOS, New Features]
tags: [iOS 12, Siri Shortcuts, ARKit, Performance]
---

iOS 12 在今年 WWDC 2018 發布，這週研究新特性，看看能如何應用在 app 開發中。iOS 12 主要聚焦在效能提升和既有功能的改進，而非大幅改版。

## 效能提升

Apple 強調 iOS 12 大幅提升舊裝置效能：

- App 啟動速度快 40%
- 鍵盤顯示快 50%
- Camera 啟動快 70%

對開發者的意義：**效能最佳化是持續的承諾**。使用者更新到 iOS 12 後，對 app 效能的期待也提高。

## Siri Shortcuts：讓 App 更智慧

**Siri Shortcuts 允許使用者用語音觸發 app 功能**。

### NSUserActivity Integration

最簡單的方式是透過 NSUserActivity：

```swift
import Intents

func donateActivity() {
    let activity = NSUserActivity(activityType: "com.example.app.orderCoffee")
    activity.title = "Order Coffee"
    activity.userInfo = ["drink": "Latte", "size": "Grande"]
    activity.isEligibleForSearch = true
    activity.isEligibleForPrediction = true  // 重點：讓 Siri 學習
    
    // 設定建議的語音指令
    activity.suggestedInvocationPhrase = "Order my usual coffee"
    
    self.userActivity = activity
    activity.becomeCurrent()
}

// 當 Siri 觸發時
func application(_ application: UIApplication,
                 continue userActivity: NSUserActivity,
                 restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
    
    if userActivity.activityType == "com.example.app.orderCoffee" {
        if let drink = userActivity.userInfo?["drink"] as? String,
           let size = userActivity.userInfo?["size"] as? String {
            orderCoffee(drink: drink, size: size)
            return true
        }
    }
    
    return false
}
```

**Siri 會學習使用者習慣**：經常在早上 8 點打開 app 點咖啡，Siri 會在 8 點建議這個動作。

### Custom Intents

更進階的做法是定義自訂 Intent：

1. File → New → File → SiriKit Intent Definition File
2. 定義 Intent（如 "Order Coffee"）
3. 設定參數（drink type, size, location）

```swift
import Intents

class OrderCoffeeIntentHandler: NSObject, OrderCoffeeIntentHandling {
    func handle(intent: OrderCoffeeIntent, completion: @escaping (OrderCoffeeIntentResponse) -> Void) {
        guard let drink = intent.drink,
              let size = intent.size else {
            completion(OrderCoffeeIntentResponse(code: .failure, userActivity: nil))
            return
        }
        
        // 執行訂購邏輯
        CoffeeService.shared.order(drink: drink, size: size) { success in
            if success {
                let response = OrderCoffeeIntentResponse(code: .success, userActivity: nil)
                response.orderNumber = "A123"
                completion(response)
            } else {
                completion(OrderCoffeeIntentResponse(code: .failure, userActivity: nil))
            }
        }
    }
}
```

**使用者可以在 Settings → Siri & Search 錄製自訂語音指令**，比如「我要咖啡」觸發訂購。

## ARKit 2：更強大的擴增實境

**多人 AR 體驗**：

```swift
import ARKit

class ARViewController: UIViewController, ARSessionDelegate {
    let sceneView = ARSCNView()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let configuration = ARWorldTrackingConfiguration()
        configuration.isCollaborationEnabled = true  // iOS 12 新增
        
        sceneView.session.run(configuration)
        sceneView.session.delegate = self
    }
    
    // 接收其他裝置的 AR 資料
    func session(_ session: ARSession, didOutputCollaborationData data: ARSession.CollaborationData) {
        // 透過網路傳送給其他玩家
        sendToOtherPlayers(data)
    }
    
    // 接收網路資料
    func receiveDataFromNetwork(_ data: Data) {
        do {
            let collaborationData = try ARSession.CollaborationData(data: data)
            sceneView.session.update(with: collaborationData)
        } catch {
            print("Failed to update with collaboration data")
        }
    }
}
```

**Persistent AR**：儲存和載入 AR 場景

```swift
// 儲存 AR 世界地圖
sceneView.session.getCurrentWorldMap { worldMap, error in
    guard let map = worldMap else { return }
    
    do {
        let data = try NSKeyedArchiver.archivedData(withRootObject: map, requiringSecureCoding: true)
        try data.write(to: self.mapSaveURL)
    } catch {
        print("Failed to save map")
    }
}

// 載入 AR 世界地圖
func loadWorldMap() {
    guard let data = try? Data(contentsOf: mapSaveURL),
          let worldMap = try? NSKeyedUnarchiver.unarchivedObject(ofClass: ARWorldMap.self, from: data) else {
        return
    }
    
    let configuration = ARWorldTrackingConfiguration()
    configuration.initialWorldMap = worldMap
    sceneView.session.run(configuration, options: [.resetTracking, .removeExistingAnchors])
}
```

這讓使用者可以在同一個位置放置虛擬物件，離開後再回來還能看到。

## Grouped Notifications：通知群組

iOS 12 自動將同一個 app 的通知群組：

```swift
// 設定 thread identifier 控制群組
let content = UNMutableNotificationContent()
content.title = "New Message"
content.body = "Alice: How are you?"
content.threadIdentifier = "chat_conversation_123"  // 同一對話的訊息會群組

let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: nil)
UNUserNotificationCenter.current().add(request)
```

**Summary 格式**：

```swift
content.summaryArgument = "Alice"  // 顯示 "3 messages from Alice"
content.summaryArgumentCount = 1
```

## Screen Time API：監控 App 使用時間

雖然主要是系統功能，但開發者可以取得資訊優化體驗：

```swift
// 檢查是否在「停用時間」
if #available(iOS 12.0, *) {
    // Screen Time 相關 API 主要是系統層級
    // 開發者無法直接存取，但可以配合 Screen Time 設計功能
}
```

## Automatic Strong Passwords：自動強密碼

**Password AutoFill 改進**：

```swift
// UITextField 設定 content type
usernameTextField.textContentType = .username
passwordTextField.textContentType = .newPassword  // 新密碼會自動產生強密碼

// 在 iOS 12，會自動提供強密碼建議
```

**Security Code AutoFill**：自動填入簡訊驗證碼

```swift
// 設定 content type
verificationCodeTextField.textContentType = .oneTimeCode

// iOS 會自動偵測簡訊中的驗證碼並建議填入
```

這讓使用者體驗大幅提升，不用手動複製貼上驗證碼。

## CarPlay Audio Apps

iOS 12 開放第三方音訊 app 支援 CarPlay：

```swift
import MediaPlayer

// 實作 CarPlay 介面
class CarPlaySceneDelegate: UIResponder, CPTemplateApplicationSceneDelegate {
    func templateApplicationScene(_ templateApplicationScene: CPTemplateApplicationScene,
                                  didConnect interfaceController: CPInterfaceController) {
        let item1 = CPListItem(text: "Playlist 1", detailText: "20 songs")
        let item2 = CPListItem(text: "Playlist 2", detailText: "15 songs")
        
        let section = CPListSection(items: [item1, item2])
        let listTemplate = CPListTemplate(title: "My Music", sections: [section])
        
        interfaceController.setRootTemplate(listTemplate, animated: true)
    }
}
```

這對音樂、Podcast、有聲書 app 很重要。

## Critical Alerts：重要通知

繞過「勿擾模式」的通知（需要 Apple 批准）：

```swift
// 在 Entitlements 加入 Critical Alerts
// 請求權限
UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .criticalAlert]) { granted, error in
    // ...
}

// 發送 critical alert
let content = UNMutableNotificationContent()
content.title = "Emergency"
content.body = "Urgent message"
content.sound = UNNotificationSound.defaultCriticalSound(withAudioVolume: 1.0)

let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: nil)
UNUserNotificationCenter.current().add(request)
```

適用於醫療、緊急服務類 app。

## Network.framework：新的網路 API

取代 BSD sockets，提供更現代的 API：

```swift
import Network

let connection = NWConnection(host: "example.com", port: 80, using: .tcp)

connection.stateUpdateHandler = { state in
    switch state {
    case .ready:
        print("Connected")
    case .failed(let error):
        print("Failed: \(error)")
    default:
        break
    }
}

connection.start(queue: .global())

// 發送資料
let data = "GET / HTTP/1.1\r\n\r\n".data(using: .utf8)!
connection.send(content: data, completion: .contentProcessed { error in
    if let error = error {
        print("Send error: \(error)")
    }
})

// 接收資料
connection.receive(minimumIncompleteLength: 1, maximumLength: 65536) { data, context, isComplete, error in
    if let data = data, !data.isEmpty {
        print("Received: \(String(data: data, encoding: .utf8) ?? "")")
    }
}
```

這個 API 更適合現代網路應用，支援 IPv4/IPv6、TLS、HTTP/2 等。

## 展望未來：SwiftUI 的傳聞

雖然 SwiftUI 要到 2019 年的 iOS 13 才正式發布，但已經有傳聞 Apple 在開發宣告式 UI 框架：

```swift
// 這是想像的語法（2018 年還不存在）
struct ContentView: View {
    var body: some View {
        VStack {
            Text("Hello, World!")
            Button("Tap me") {
                print("Tapped")
            }
        }
    }
}
```

如果成真，將徹底改變 iOS UI 開發方式，從 imperative（命令式）轉向 declarative（宣告式），類似 React 或 Flutter。

## 學習心得

iOS 12 是務實的更新，專注在效能和使用者體驗，而非大量新功能。對開發者來說，重點是：

**Siri Shortcuts**：讓 app 更智慧，但需要思考哪些功能適合語音觸發
**ARKit 2**：多人 AR 和持久化開啟更多應用場景
**Automatic Strong Passwords**：大幅改善使用者註冊和登入體驗
**Network.framework**：未來網路程式的標準 API

同時也要注意效能優化，因為 iOS 12 提升了基準，使用者對速度的期待更高。期待明年 iOS 13 會有更多驚喜（特別是傳聞中的 Dark Mode 和新 UI 框架）。
