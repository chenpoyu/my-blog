---
layout: post
title: "iOS 遠端推送與 APNs 整合"
date: 2018-05-11
categories: [iOS, Swift]
tags: [Swift, iOS, Remote Push, APNs, Backend Integration]
---

這週深入研究遠端推送通知。上週學的本地通知只能處理 app 本身的提醒，但真正有價值的是**從伺服器推送即時資訊**給使用者。這涉及憑證設定、後端整合、以及各種推送策略。

## 遠端推送的完整流程

理解遠端推送之前，必須先搞清楚整個生態系統的角色分工。這不像 Java Web 開發那樣單純的 Client-Server 模型，而是**三方協作**：

**階段一：註冊 Device Token**
1. App 啟動時向 iOS 系統請求推送權限
2. 系統向 APNs（Apple 推送伺服器）註冊這個裝置
3. APNs 產生一個唯一的 Device Token 並回傳給 app
4. App 把 Device Token 傳給你的後端伺服器儲存

**階段二：發送推送**
1. 後端決定要推送訊息給某個使用者
2. 後端找出該使用者的 Device Token
3. 後端透過 APNs API 發送推送請求（包含 Token 和訊息內容）
4. APNs 負責把訊息傳送到對應的裝置

這個架構最關鍵的設計是 **Apple 掌握所有裝置的連線**。你的後端不直接連接使用者裝置，而是透過 APNs 中介。這有幾個好處：

- **統一的連線管理**：裝置只需要維持一條連線給 APNs，不是每個 app 都要自己連
- **安全性**：Apple 驗證推送來源，防止惡意推送
- **電源效率**：系統級的推送服務比個別 app 省電太多

這讓我想到微服務架構中的 Message Broker（像 RabbitMQ 或 Kafka），所有服務透過中央訊息系統通訊，而不是點對點連接。

## 設定 APNs 憑證

要使用遠端推送，首先要在 Apple Developer 網站建立憑證。這個過程相當繁瑣，但不做不行。

### 建立 App ID 並啟用推送

1. 登入 Apple Developer Console
2. Certificates, Identifiers & Profiles → Identifiers
3. 建立或編輯你的 App ID
4. 勾選 **Push Notifications** 能力
5. 儲存並生成憑證

### 產生推送憑證

Apple 提供兩種推送憑證：

**開發環境憑證（Development）**
- 用於測試階段
- 只能推送給用開發者憑證安裝的 app
- APNs 端點：`api.development.push.apple.com`

**正式環境憑證（Production）**
- 用於 App Store 發布的 app
- 可推送給所有使用者
- APNs 端點：`api.push.apple.com`

這兩個環境是**完全分離**的。開發時取得的 Device Token 無法用於正式環境推送，反之亦然。我一開始沒搞清楚這點，花了很多時間 debug 為什麼測試推送總是失敗。

這個概念類似 Spring 的 Profile（dev / prod），不同環境使用不同的配置和端點。

### 轉換憑證格式

下載的憑證是 `.cer` 格式，但大部分後端工具需要 `.p12` 或 `.pem` 格式。轉換步驟：

```bash
# 匯入憑證到鑰匙圈，然後匯出為 .p12
# 或用指令轉換

# .cer 轉 .pem
openssl x509 -in aps_development.cer -inform der -out aps_development.pem

# .p12 轉 .pem（如果後端需要）
openssl pkcs12 -in aps_development.p12 -out aps_development.pem -nodes -clcerts
```

這些憑證要妥善保管，**絕對不能 commit 到 git**。我在公司的 Java 專案中見過把資料庫密碼 commit 到 repo 的慘案，推送憑證更敏感，洩漏的話任何人都能冒充你的伺服器推送訊息。

## iOS 端：註冊遠端推送

在 app 中註冊遠端推送：

```swift
import UIKit
import UserNotifications

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        
        // 請求通知權限
        UNUserNotificationCenter.current().requestAuthorization(
            options: [.alert, .sound, .badge]
        ) { granted, error in
            if granted {
                print("使用者同意推送")
                // 在主執行緒註冊
                DispatchQueue.main.async {
                    application.registerForRemoteNotifications()
                }
            } else {
                print("使用者拒絕推送：\(error?.localizedDescription ?? "")")
            }
        }
        
        return true
    }
    
    // 成功註冊遠端推送
    func application(
        _ application: UIApplication,
        didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
    ) {
        // 將 Data 轉換成字串格式
        let tokenString = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
        print("Device Token: \(tokenString)")
        
        // 傳送給後端伺服器
        sendTokenToServer(tokenString)
    }
    
    // 註冊失敗
    func application(
        _ application: UIApplication,
        didFailToRegisterForRemoteNotificationsWithError error: Error
    ) {
        print("註冊失敗：\(error.localizedDescription)")
        // 模擬器會走到這裡，因為模擬器不支援推送
    }
}
```

`deviceToken` 是 `Data` 類型，需要轉換成 hex 字串才能傳給後端。這個 Token 就像使用者的**推送地址**，後端要推送訊息必須知道這個地址。

**重要細節**：Device Token 不是永久不變的。Apple 會在某些情況下更新 Token，比如：
- 使用者重新安裝 app
- 使用者從備份還原裝置
- 作業系統升級

所以最佳實踐是**每次 app 啟動都檢查並更新 Token**。後端要設計好處理 Token 更新的邏輯。

## 後端實作：發送推送

後端有多種方式發送推送，這週我用 Java 實作了一個簡單的推送服務。

### 使用 Pushy 函式庫

```java
import com.turo.pushy.apns.*;
import com.turo.pushy.apns.util.*;

public class APNsService {
    private ApnsClient apnsClient;
    
    public APNsService(String certificatePath, String password) throws Exception {
        // 載入憑證
        this.apnsClient = new ApnsClientBuilder()
            .setApnsServer(ApnsClientBuilder.DEVELOPMENT_APNS_HOST)
            .setClientCredentials(
                new File(certificatePath), 
                password
            )
            .build();
    }
    
    public void sendPush(String deviceToken, String title, String message) {
        try {
            // 建立推送內容
            String payload = new SimpleApnsPushNotification(
                deviceToken,
                "com.yourcompany.yourapp",  // Bundle ID
                String.format(
                    "{\"aps\":{\"alert\":{\"title\":\"%s\",\"body\":\"%s\"},\"sound\":\"default\"}}",
                    title,
                    message
                )
            );
            
            // 發送推送
            PushNotificationResponse<SimpleApnsPushNotification> response = 
                apnsClient.sendNotification(payload).get();
            
            if (response.isAccepted()) {
                System.out.println("推送成功");
            } else {
                System.out.println("推送失敗：" + response.getRejectionReason());
            }
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    public void shutdown() throws Exception {
        apnsClient.close().get();
    }
}
```

推送的內容是 **JSON 格式**，必須包含 `aps` 物件。這是 Apple 定義的標準結構：

```json
{
  "aps": {
    "alert": {
      "title": "新訊息",
      "body": "小明傳了一張照片給你"
    },
    "sound": "default",
    "badge": 1
  },
  "custom_data": {
    "message_id": 12345,
    "sender": "ming"
  }
}
```

`aps` 以外的欄位是**自訂資料**，可以放任何 JSON 資料。這些資料會傳到 app，但不會顯示在通知上。

這個設計很靈活。你可以在推送中夾帶業務邏輯需要的資料，app 接收後直接處理，不用再發 API 請求。

## 接收遠端推送

在 app 中處理收到的推送：

```swift
func application(
    _ application: UIApplication,
    didReceiveRemoteNotification userInfo: [AnyHashable : Any],
    fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void
) {
    print("收到遠端推送：\(userInfo)")
    
    // 解析自訂資料
    if let messageId = userInfo["message_id"] as? Int {
        print("訊息 ID：\(messageId)")
        // 根據 ID 取得完整訊息內容
        fetchMessageDetail(messageId) { success in
            if success {
                completionHandler(.newData)
            } else {
                completionHandler(.failed)
            }
        }
    } else {
        completionHandler(.noData)
    }
}
```

`fetchCompletionHandler` 很重要，它告訴系統這個推送有沒有帶來新資料。iOS 會根據你的回報調整推送策略：

- **.newData**：有新資料，推送很有用
- **.noData**：沒新資料，可能是重複推送
- **.failed**：處理失敗，可能網路有問題

如果你總是回報 `.noData`，系統可能降低你 app 的推送優先權。這是 iOS 對電源和頻寬的智慧管理。

## 靜默推送（Silent Push）

有時你不需要顯示通知，只是想叫醒 app 去更新資料。這就是**靜默推送**：

```json
{
  "aps": {
    "content-available": 1
  },
  "update_type": "new_messages"
}
```

注意沒有 `alert`、`sound`、`badge`，只有 `content-available: 1`。這個推送不會顯示任何 UI，但會觸發 app 在背景執行。

使用場景：
- 聊天 app 預先下載新訊息，使用者打開時立刻顯示
- 新聞 app 更新頭條，使用者打開時內容已經是最新的
- 同步使用者在其他裝置的操作

靜默推送有**頻率限制**。如果濫用，iOS 會節流（throttle）你的推送。Apple 建議每小時不超過 2-3 個靜默推送。

這讓我想到 Java 中的 rate limiting。後端也要實作推送頻率控制，避免觸發 Apple 的保護機制。

## 推送的最佳實踐

經過這週的研究和測試，我整理出幾個關鍵原則：

### 1. 個人化推送內容

不要發「親愛的用戶您好」這種通用訊息。推送應該包含具體、相關的資訊：

糟糕：「您有新訊息」  
良好：「小明：晚上要不要一起吃飯？」

### 2. 控制推送頻率

太頻繁的推送會讓使用者關閉通知或解除安裝 app。後端要實作智慧推送邏輯：

```java
public class PushStrategy {
    private static final long MIN_INTERVAL = 3600000; // 1 小時
    
    public boolean shouldSendPush(String userId) {
        // 檢查上次推送時間
        Long lastPush = getLastPushTime(userId);
        if (lastPush != null) {
            long elapsed = System.currentTimeMillis() - lastPush;
            if (elapsed < MIN_INTERVAL) {
                return false;  // 太頻繁，跳過
            }
        }
        
        // 檢查使用者的偏好時間
        if (isUserSleepTime(userId)) {
            return false;  // 深夜不打擾
        }
        
        return true;
    }
}
```

### 3. 合併多個通知

如果短時間內有多個相同類型的推送，應該合併成一個：

糟糕：
- 「小明按讚了你的貼文」
- 「小華按讚了你的貼文」
- 「小李按讚了你的貼文」

良好：
- 「小明和其他 2 人按讚了你的貼文」

iOS 提供 **Thread ID** 機制來分組通知：

```json
{
  "aps": {
    "alert": {
      "title": "3 個新按讚",
      "body": "小明、小華、小李按讚了你的貼文"
    },
    "thread-id": "post-likes-12345"
  }
}
```

相同 thread-id 的通知會被分組顯示。

### 4. 提供關閉選項

在 app 中讓使用者細緻控制推送類型：

```swift
struct NotificationSettings {
    var enableNewMessage = true
    var enableLikes = true
    var enableComments = true
    var enableFollowers = false
    var quietHours = (start: 22, end: 8)  // 22:00 到 8:00 安靜
}
```

後端根據這些設定決定是否推送。使用者感受到被尊重，比較不會完全關閉推送。

## 除錯遠端推送

遠端推送的除錯比本地通知困難，因為涉及多個環節：

### 使用 Pusher 工具測試

我用了 [Pusher](https://github.com/noodlewerk/NWPusher) 這個 macOS 工具，可以直接發送測試推送：

1. 載入 `.p12` 憑證
2. 輸入 Device Token
3. 撰寫推送內容
4. 點擊發送

如果推送能成功，代表憑證和 Token 都正確。如果失敗，檢查：
- 憑證是否過期
- Device Token 是否正確（注意開發/正式環境的差異）
- Bundle ID 是否匹配

### 後端日誌

後端要詳細記錄推送結果：

```java
logger.info("發送推送給 User: {}, Token: {}", userId, deviceToken);
PushNotificationResponse response = apnsClient.sendNotification(payload).get();

if (response.isAccepted()) {
    logger.info("推送成功");
} else {
    logger.error("推送失敗：{}, Token 可能無效", response.getRejectionReason());
    // 如果 Token 無效，從資料庫中移除
    if ("BadDeviceToken".equals(response.getRejectionReason())) {
        invalidateToken(deviceToken);
    }
}
```

APNs 會回報失敗原因，像是 `BadDeviceToken`、`DeviceTokenNotForTopic` 等。根據錯誤類型採取對應處理。

## 小結

這週研究遠端推送，最大的感受是這是一個**端到端的系統工程**。不只是寫 iOS 程式碼，還要：

1. 理解 Apple 的推送架構
2. 設定複雜的憑證和權限
3. 實作後端推送服務
4. 設計推送策略和頻率控制
5. 處理 Token 失效和更新

這比我做 Java Web 開發時的 WebSocket 或 Server-Sent Events 複雜，但好處是 **Apple 幫你處理了裝置管理和電源優化**。你不用擔心長連線掉線、重連邏輯、電量消耗等問題。

關鍵要點：
1. **Device Token 會改變**：每次啟動都要檢查並更新
2. **開發和正式環境完全分離**：憑證和端點都不同
3. **尊重使用者**：控制頻率、提供細緻的開關、避免深夜推送
4. **監控推送品質**：根據 APNs 的回報調整策略

下週計劃研究 Core Location，包括定位服務、地圖整合、地理圍欄等。這能和推送結合，實作「到達某地時提醒」的功能。
