---
layout: post
title: "iOS 推送通知基礎"
date: 2018-05-04
categories: [iOS, Swift]
tags: [Swift, iOS, Push Notification, UserNotifications, APNs]
---

這週開始研究推送通知（Push Notification）。在 Java Web 開發中，我們用 WebSocket 或輪詢來推送訊息給使用者。但在行動裝置上，推送通知是更節能也更直接的方式，因為它可以在 app 沒有執行時也能觸達使用者。

## 推送通知的運作原理

一開始我以為推送通知就是「app 跑在背景時接收訊息」，但實際架構更複雜。iOS 的推送系統涉及三方：

1. **你的 App**：請求權限、接收通知
2. **APNs（Apple Push Notification service）**：Apple 的推送伺服器
3. **你的後端伺服器**：決定要推送什麼內容給誰

流程是這樣的：
- App 向系統註冊推送功能，取得唯一的 **Device Token**
- App 把 Device Token 傳給你的後端伺服器
- 當後端想推送訊息時，把訊息和 Device Token 發給 APNs
- APNs 負責把訊息傳送到使用者的裝置

這個架構的好處是**節省電力**。如果每個 app 都自己維持長連線到各自的伺服器，電池會很快耗盡。透過 APNs 統一處理，系統只需要一條連線就能接收所有 app 的推送。

這讓我想到 Java 微服務架構中的 API Gateway，所有請求透過統一入口，避免每個服務都暴露端點。

## 本地通知 vs 遠端通知

iOS 的通知分兩種：

**本地通知（Local Notification）**：由 app 本身排程，不需要伺服器。比如提醒 app 可以設定「每天早上 8 點提醒吃藥」，這就是本地通知。

**遠端通知（Remote Notification）**：由伺服器透過 APNs 推送。比如社群 app 告訴你「有人按讚你的貼文」，這需要遠端通知。

這週先專注於本地通知，因為它不需要設定伺服器和憑證，更容易理解基礎概念。

## 請求使用者權限

iOS 對隱私保護很嚴格，要使用通知功能**必須先取得使用者同意**。這和 Android 的直接推送不同，是 Apple 生態系的特色。

```swift
import UserNotifications

class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        
        // 請求通知權限
        let center = UNUserNotificationCenter.current()
        center.requestAuthorization(options: [.alert, .sound, .badge]) { granted, error in
            if granted {
                print("使用者同意推送通知")
            } else {
                print("使用者拒絕推送通知")
            }
        }
        
        return true
    }
}
```

`options` 參數指定你需要哪些權限：
- **alert**：顯示彈窗或橫幅
- **sound**：播放提示音
- **badge**：在 app 圖示上顯示數字徽章

使用者可以分別控制這三種權限。有些使用者可能同意顯示通知但不要聲音，這在深夜使用時很貼心。

這個權限請求**只會顯示一次**。如果使用者拒絕，下次再呼叫不會彈窗，必須引導使用者去「設定」手動開啟。這和 Java 桌面應用很不同，桌面應用可以一直煩使用者，但 iOS 保護使用者的選擇。

## 建立本地通知

取得權限後，就能排程通知了：

```swift
func scheduleLocalNotification() {
    // 建立通知內容
    let content = UNMutableNotificationContent()
    content.title = "記得喝水"
    content.body = "已經過了一小時，該補充水分了！"
    content.sound = .default
    content.badge = 1  // 圖示上的數字
    
    // 設定觸發條件：5 秒後觸發
    let trigger = UNTimeIntervalNotificationTrigger(
        timeInterval: 5,
        repeats: false
    )
    
    // 建立請求
    let request = UNNotificationRequest(
        identifier: "water-reminder",  // 唯一識別碼
        content: content,
        trigger: trigger
    )
    
    // 加入排程
    UNUserNotificationCenter.current().add(request) { error in
        if let error = error {
            print("排程通知失敗：\(error)")
        } else {
            print("通知已排程")
        }
    }
}
```

`identifier` 很重要，它用來識別通知。如果用相同的 identifier 新增通知，會取代舊的通知。這可以用來更新提醒內容而不產生重複。

這個概念類似資料庫的 Primary Key，用來確保唯一性。

## 不同的觸發條件

除了時間間隔，還有其他觸發方式：

### 指定日期和時間

```swift
var dateComponents = DateComponents()
dateComponents.hour = 8
dateComponents.minute = 0

let trigger = UNCalendarNotificationTrigger(
    dateMatching: dateComponents,
    repeats: true  // 每天重複
)
```

這會在每天早上 8:00 觸發通知，適合規律的提醒。注意 `repeats: true` 會讓通知持續觸發，直到你手動移除。

### 位置觸發

```swift
let center = CLLocationCoordinate2D(latitude: 25.0330, longitude: 121.5654)
let region = CLCircularRegion(
    center: center,
    radius: 100,  // 半徑 100 公尺
    identifier: "office"
)
region.notifyOnEntry = true  // 進入區域時通知

let trigger = UNLocationNotificationTrigger(
    region: region,
    repeats: false
)
```

這可以實作「到達辦公室附近時提醒打卡」這類功能。位置觸發需要額外的定位權限，這週還沒深入研究 Core Location，之後會再探討。

## 處理使用者點擊通知

當使用者點擊通知時，app 會被啟動或回到前景。我們需要處理這個事件：

```swift
class AppDelegate: UIResponder, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        
        UNUserNotificationCenter.current().delegate = self
        // ...請求權限的程式碼
        
        return true
    }
    
    // 當 app 在前景時收到通知
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void
    ) {
        // 即使 app 在前景也顯示通知
        completionHandler([.banner, .sound, .badge])
    }
    
    // 使用者點擊通知時
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void
    ) {
        let identifier = response.notification.request.identifier
        
        print("使用者點擊了通知：\(identifier)")
        
        // 根據不同的通知導航到不同畫面
        if identifier == "water-reminder" {
            // 導航到喝水記錄頁面
            navigateToWaterLog()
        }
        
        completionHandler()
    }
}
```

`willPresent` 方法很特別：預設情況下，**app 在前景時收到通知不會顯示**，因為使用者已經在用 app 了。但如果你希望顯示，就在 completionHandler 中指定選項。

這個設計很合理。想像你正在用社群 app，此時收到「有新訊息」的推送，沒必要彈出通知打斷你，因為你已經在看訊息了。

## 通知內容的豐富化

除了基本的標題和內文，通知還能包含圖片、影片、附件：

```swift
let content = UNMutableNotificationContent()
content.title = "今日天氣"
content.body = "多雲時晴，氣溫 25°C"

// 加入圖片附件
if let imageURL = Bundle.main.url(forResource: "weather", withExtension: "jpg") {
    if let attachment = try? UNNotificationAttachment(
        identifier: "weather-image",
        url: imageURL,
        options: nil
    ) {
        content.attachments = [attachment]
    }
}
```

通知支援的附件類型包括圖片、音訊、影片。但有大小限制（圖片 10MB、影片 50MB），而且必須是本地檔案，不能直接用網路 URL。

如果要用網路圖片，需要先下載到本地再建立附件。這確保通知能離線顯示，不會因為網路問題而載入失敗。

## 可操作的通知

iOS 允許在通知上加入**動作按鈕**，讓使用者不用打開 app 就能快速回應：

```swift
// 定義動作
let acceptAction = UNNotificationAction(
    identifier: "accept",
    title: "接受",
    options: .foreground  // 點擊後開啟 app
)

let declineAction = UNNotificationAction(
    identifier: "decline",
    title: "拒絕",
    options: .destructive  // 紅色按鈕
)

// 定義分類
let category = UNNotificationCategory(
    identifier: "meeting-invitation",
    actions: [acceptAction, declineAction],
    intentIdentifiers: [],
    options: []
)

// 註冊分類
UNUserNotificationCenter.current().setNotificationCategories([category])

// 使用分類
let content = UNMutableNotificationContent()
content.title = "會議邀請"
content.body = "下午 3 點的 Sprint Review"
content.categoryIdentifier = "meeting-invitation"
```

處理動作回應：

```swift
func userNotificationCenter(
    _ center: UNUserNotificationCenter,
    didReceive response: UNNotificationResponse,
    withCompletionHandler completionHandler: @escaping () -> Void
) {
    switch response.actionIdentifier {
    case "accept":
        print("使用者接受會議")
        acceptMeeting(response.notification)
        
    case "decline":
        print("使用者拒絕會議")
        declineMeeting(response.notification)
        
    default:
        // 預設動作：點擊通知本身
        print("打開 app")
    }
    
    completionHandler()
}
```

這個功能大幅提升使用者體驗。使用者不需要「打開 app → 找到通知內容 → 執行動作」，而是直接在通知上完成，節省時間。

## 管理待處理的通知

可以查詢、更新、移除已排程的通知：

```swift
// 查詢所有待處理的通知
UNUserNotificationCenter.current().getPendingNotificationRequests { requests in
    print("目前有 \(requests.count) 個待處理的通知")
    for request in requests {
        print("- \(request.identifier): \(request.content.title)")
    }
}

// 移除特定通知
UNUserNotificationCenter.current().removePendingNotificationRequests(
    withIdentifiers: ["water-reminder"]
)

// 移除所有待處理的通知
UNUserNotificationCenter.current().removeAllPendingNotificationRequests()

// 移除已送達但還沒被點擊的通知
UNUserNotificationCenter.current().removeAllDeliveredNotifications()
```

注意「待處理（Pending）」和「已送達（Delivered）」的差異：
- **Pending**：已排程但還沒送出
- **Delivered**：已經顯示在通知中心，但使用者還沒點擊

這在實務上很重要。比如使用者設定了 10 個提醒，後來改變心意，你要能正確清除對應的通知。

## Badge 數字管理

App 圖示上的數字徽章需要手動管理：

```swift
// 設定徽章數字
UIApplication.shared.applicationIconBadgeNumber = 5

// 清除徽章
UIApplication.shared.applicationIconBadgeNumber = 0
```

**最佳實踐**：當使用者打開 app 並查看了內容後，應該清除徽章。沒有比看到一個永遠顯示數字但內容已讀的 app 更惱人的了。

我在 Java Web 開發中從來不需要考慮這些細節，但在 iOS 上，這些小地方會大幅影響使用者觀感。

## 除錯技巧

開發通知功能時常遇到的問題：

1. **通知沒顯示**：檢查是否取得權限、app 是否在背景
2. **時間不對**：注意時區問題，用 `DateComponents` 而不是絕對時間戳
3. **重複通知**：確認 `identifier` 是否重複、`repeats` 是否設定正確

模擬器可以測試本地通知，但要測試遠端推送需要實體裝置。這點和 Java Web 開發不同，Web 開發可以完全在本地測試，但 iOS 開發有些功能必須用真機。

## 小結

這週學習本地通知的過程中，最深刻的體會是 **iOS 對使用者體驗的重視**。從權限請求、通知顯示時機、到動作按鈕設計，每個細節都考慮到使用者的感受。

關鍵要點：
1. **尊重使用者權限**：一定要先請求授權
2. **合理使用通知**：不要濫發，否則使用者會關閉權限
3. **提供快速動作**：讓使用者不用打開 app 就能回應
4. **管理徽章數字**：及時清除，避免資訊過載

下週打算研究遠端推送，包括如何設定 APNs 憑證、後端如何發送推送、以及靜默推送（Silent Push）的應用場景。這需要更多的後端整合知識，剛好可以結合我的 Java 經驗。
