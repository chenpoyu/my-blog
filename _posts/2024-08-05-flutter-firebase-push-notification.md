---
layout: post
title: "Flutter Firebase 推播通知：內推、外推與 Deep Link"
date: 2024-08-05 09:30:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Firebase, Push Notification, Deep Link]
---

這週處理 Firebase Cloud Messaging（FCM）推播通知。充電站 App 需要各種通知：充電完成、優惠活動、系統維護等。

推播分成兩種：**內推**（In-App Notification，App 在前景）和**外推**（Remote Notification，App 在背景或關閉），處理方式不同。還要實作 Deep Link，點通知能直接跳到特定頁面。

## Firebase 設定

### Android 設定

1. Firebase Console 建立專案
2. 新增 Android App，填入 package name
3. 下載 `google-services.json` 放到 `android/app/`
4. 修改 `android/build.gradle`：

```gradle
dependencies {
    classpath 'com.google.gms:google-services:4.3.15'
}
```

5. 修改 `android/app/build.gradle`：

```gradle
apply plugin: 'com.google.gms.google-services'

dependencies {
    implementation platform('com.google.firebase:firebase-bom:32.0.0')
}
```

### iOS 設定

1. Firebase Console 新增 iOS App
2. 下載 `GoogleService-Info.plist` 放到 `ios/Runner/`
3. 在 Xcode 開啟專案，`Runner` → `Signing & Capabilities` → 新增 `Push Notifications`
4. 新增 `Background Modes`，勾選 `Remote notifications`
5. 到 Apple Developer 建立 APNs Key，上傳到 Firebase Console

## 套件安裝

```yaml
dependencies:
  firebase_core: ^2.15.0
  firebase_messaging: ^14.6.5
  flutter_local_notifications: ^15.1.0  # 本地通知（內推）
```

## Firebase 初始化

```dart
// main.dart
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';  // 自動產生

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  await setupDependencies();
  await _initializeNotifications();
  
  runApp(MyApp());
}

Future<void> _initializeNotifications() async {
  final messaging = FirebaseMessaging.instance;
  
  // 請求權限（iOS）
  final settings = await messaging.requestPermission(
    alert: true,
    badge: true,
    sound: true,
    provisional: false,
  );
  
  if (settings.authorizationStatus == AuthorizationStatus.authorized) {
    print('User granted permission');
  } else {
    print('User declined or has not accepted permission');
  }
  
  // 取得 FCM Token
  final token = await messaging.getToken();
  print('FCM Token: $token');
  
  // 上傳到後端，綁定使用者
  await _uploadTokenToServer(token);
  
  // 監聽 Token 變化
  messaging.onTokenRefresh.listen(_uploadTokenToServer);
}
```

## 推播處理架構

建立一個 Notification Service 統一處理：

```dart
// core/services/notification_service.dart
class NotificationService {
  static final instance = NotificationService._();
  NotificationService._();
  
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;
  final FlutterLocalNotificationsPlugin _localNotifications = 
      FlutterLocalNotificationsPlugin();
  
  Future<void> initialize() async {
    // 設定本地通知（內推）
    await _initializeLocalNotifications();
    
    // 處理前景通知（App 開啟中）
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);
    
    // 處理背景通知點擊（App 在背景）
    FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);
    
    // 處理 App 從關閉狀態被通知喚醒
    final initialMessage = await _messaging.getInitialMessage();
    if (initialMessage != null) {
      _handleNotificationTap(initialMessage);
    }
  }
  
  Future<void> _initializeLocalNotifications() async {
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: false,
      requestBadgePermission: false,
      requestSoundPermission: false,
    );
    
    const initSettings = InitializationSettings(
      android: androidSettings,
      iOS: iosSettings,
    );
    
    await _localNotifications.initialize(
      initSettings,
      onDidReceiveNotificationResponse: _onLocalNotificationTap,
    );
  }
  
  // 前景通知（內推）
  void _handleForegroundMessage(RemoteMessage message) {
    print('Foreground message: ${message.notification?.title}');
    
    // 顯示本地通知（因為前景時 FCM 不會自動顯示）
    _showLocalNotification(message);
  }
  
  // 通知點擊（外推）
  void _handleNotificationTap(RemoteMessage message) {
    print('Notification tapped: ${message.data}');
    
    // 處理 Deep Link
    _handleDeepLink(message.data);
  }
  
  // 本地通知點擊
  void _onLocalNotificationTap(NotificationResponse response) {
    print('Local notification tapped: ${response.payload}');
    
    if (response.payload != null) {
      final data = jsonDecode(response.payload!);
      _handleDeepLink(data);
    }
  }
  
  Future<void> _showLocalNotification(RemoteMessage message) async {
    final notification = message.notification;
    if (notification == null) return;
    
    const androidDetails = AndroidNotificationDetails(
      'charging_channel',
      'Charging Notifications',
      channelDescription: 'Notifications for charging status',
      importance: Importance.high,
      priority: Priority.high,
      showWhen: true,
    );
    
    const iosDetails = DarwinNotificationDetails(
      presentAlert: true,
      presentBadge: true,
      presentSound: true,
    );
    
    const details = NotificationDetails(
      android: androidDetails,
      iOS: iosDetails,
    );
    
    await _localNotifications.show(
      message.hashCode,
      notification.title,
      notification.body,
      details,
      payload: jsonEncode(message.data),
    );
  }
  
  void _handleDeepLink(Map<String, dynamic> data) {
    final type = data['type'];
    
    switch (type) {
      case 'charging_complete':
        _navigateToChargingHistory(data['session_id']);
        break;
      case 'promotion':
        _navigateToPromotion(data['promo_id']);
        break;
      case 'station_update':
        _navigateToStation(data['station_id']);
        break;
      default:
        _navigateToHome();
    }
  }
  
  void _navigateToChargingHistory(String sessionId) {
    // 使用 Navigator 跳轉（需要 NavigatorKey）
    navigatorKey.currentState?.pushNamed(
      '/charging-history',
      arguments: sessionId,
    );
  }
  
  // 其他導航方法...
}
```

## 通知類型區分

後端發送推播時，要包含不同的 `data` 欄位來區分類型。

### 充電完成通知

```json
{
  "notification": {
    "title": "充電完成",
    "body": "您的車輛已充電完成，共充了 45.2 kWh"
  },
  "data": {
    "type": "charging_complete",
    "session_id": "session_12345",
    "energy": "45.2",
    "cost": "360"
  }
}
```

### 優惠活動通知

```json
{
  "notification": {
    "title": "限時優惠！",
    "body": "本週末充電享 8 折優惠"
  },
  "data": {
    "type": "promotion",
    "promo_id": "promo_2024_08",
    "discount": "0.8"
  }
}
```

### 系統通知

```json
{
  "notification": {
    "title": "充電站維護通知",
    "body": "台北站 A3 充電樁將於今晚 10 點進行維護"
  },
  "data": {
    "type": "station_update",
    "station_id": "station_taipei_001",
    "charger_id": "A3",
    "status": "maintenance"
  }
}
```

## Deep Link 進階處理

使用 **go_router** 統一管理路由和 Deep Link：

```dart
// core/router/app_router.dart
final navigatorKey = GlobalKey<NavigatorState>();

final router = GoRouter(
  navigatorKey: navigatorKey,
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => MapScreen(),
    ),
    GoRoute(
      path: '/station/:id',
      builder: (context, state) {
        final stationId = state.pathParameters['id']!;
        return StationDetailScreen(stationId: stationId);
      },
    ),
    GoRoute(
      path: '/charging-history/:sessionId',
      builder: (context, state) {
        final sessionId = state.pathParameters['sessionId']!;
        return ChargingHistoryScreen(sessionId: sessionId);
      },
    ),
    GoRoute(
      path: '/promotion/:promoId',
      builder: (context, state) {
        final promoId = state.pathParameters['promoId']!;
        return PromotionDetailScreen(promoId: promoId);
      },
    ),
  ],
);

// 在 MyApp 使用
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: router,
      // ...
    );
  }
}
```

修改 NotificationService 的 Deep Link 處理：

```dart
void _handleDeepLink(Map<String, dynamic> data) {
  final type = data['type'];
  
  switch (type) {
    case 'charging_complete':
      router.go('/charging-history/${data['session_id']}');
      break;
    case 'promotion':
      router.go('/promotion/${data['promo_id']}');
      break;
    case 'station_update':
      router.go('/station/${data['station_id']}');
      break;
    default:
      router.go('/');
  }
}
```

## 通知頻道（Android）

Android 8.0+ 需要建立通知頻道：

```dart
Future<void> _createNotificationChannels() async {
  if (Platform.isAndroid) {
    // 充電相關通知（高優先度）
    const chargingChannel = AndroidNotificationChannel(
      'charging_channel',
      'Charging Notifications',
      description: 'Notifications for charging status',
      importance: Importance.high,
      enableVibration: true,
      playSound: true,
    );
    
    // 促銷通知（一般優先度）
    const promotionChannel = AndroidNotificationChannel(
      'promotion_channel',
      'Promotions',
      description: 'Promotional offers and discounts',
      importance: Importance.defaultImportance,
    );
    
    // 系統通知（低優先度）
    const systemChannel = AndroidNotificationChannel(
      'system_channel',
      'System Updates',
      description: 'System maintenance and updates',
      importance: Importance.low,
    );
    
    await _localNotifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(chargingChannel);
    
    await _localNotifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(promotionChannel);
    
    await _localNotifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(systemChannel);
  }
}
```

根據通知類型選擇頻道：

```dart
Future<void> _showLocalNotification(RemoteMessage message) async {
  final type = message.data['type'];
  String channelId;
  
  switch (type) {
    case 'charging_complete':
    case 'charging_start':
      channelId = 'charging_channel';
      break;
    case 'promotion':
      channelId = 'promotion_channel';
      break;
    default:
      channelId = 'system_channel';
  }
  
  final androidDetails = AndroidNotificationDetails(
    channelId,
    'Notifications',
    importance: Importance.high,
    priority: Priority.high,
  );
  
  // ...
}
```

## 背景處理

有些通知需要在背景處理（例如更新本地資料庫），不只是顯示通知。

```dart
// 在 main.dart 最外層定義（不能是 class 的 method）
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  print('Background message: ${message.notification?.title}');
  
  // 處理背景邏輯
  final type = message.data['type'];
  
  if (type == 'charging_complete') {
    // 更新本地資料庫
    final sessionId = message.data['session_id'];
    await _updateChargingSession(sessionId);
  } else if (type == 'station_update') {
    // 更新充電站狀態
    final stationId = message.data['station_id'];
    await _updateStationStatus(stationId);
  }
}

Future<void> _updateChargingSession(String sessionId) async {
  // 更新本地資料庫
  final db = await ChargingDatabase().database;
  await db.update(
    'charging_sessions',
    {'status': 'completed', 'end_time': DateTime.now().millisecondsSinceEpoch},
    where: 'id = ?',
    whereArgs: [sessionId],
  );
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // 註冊背景處理
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
  
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // ...
}
```

## 通知權限管理

要優雅地請求通知權限，不要一開啟 App 就跳權限請求。

```dart
class NotificationPermissionDialog extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text('開啟通知'),
      content: Text('開啟通知以即時收到充電完成、優惠活動等訊息'),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: Text('稍後再說'),
        ),
        ElevatedButton(
          onPressed: () => Navigator.pop(context, true),
          child: Text('開啟'),
        ),
      ],
    );
  }
}

// 在適當時機請求
Future<void> _requestNotificationPermission() async {
  final shouldRequest = await showDialog<bool>(
    context: context,
    builder: (context) => NotificationPermissionDialog(),
  );
  
  if (shouldRequest == true) {
    final messaging = FirebaseMessaging.instance;
    final settings = await messaging.requestPermission();
    
    if (settings.authorizationStatus == AuthorizationStatus.authorized) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('通知已開啟')),
      );
    }
  }
}
```

## 主題訂閱（Topic）

群發通知可以用 Topic，不用維護所有 Token。

```dart
Future<void> subscribeToTopics() async {
  final messaging = FirebaseMessaging.instance;
  
  // 訂閱全體通知
  await messaging.subscribeToTopic('all_users');
  
  // 根據使用者偏好訂閱
  final prefs = await SharedPreferences.getInstance();
  
  if (prefs.getBool('notify_promotions') ?? true) {
    await messaging.subscribeToTopic('promotions');
  }
  
  if (prefs.getBool('notify_maintenance') ?? true) {
    await messaging.subscribeToTopic('maintenance');
  }
  
  // 根據地區訂閱
  final region = prefs.getString('region') ?? 'taipei';
  await messaging.subscribeToTopic('region_$region');
}

Future<void> unsubscribeFromTopic(String topic) async {
  await FirebaseMessaging.instance.unsubscribeFromTopic(topic);
}
```

後端發送時指定 Topic：

```json
{
  "topic": "promotions",
  "notification": {
    "title": "限時優惠！",
    "body": "本週末充電享 8 折優惠"
  }
}
```

## 通知統計

記錄通知的開啟率、點擊率：

```dart
void _handleNotificationTap(RemoteMessage message) {
  // 記錄點擊事件
  _logNotificationEvent('notification_opened', message);
  
  _handleDeepLink(message.data);
}

Future<void> _logNotificationEvent(
  String eventName,
  RemoteMessage message,
) async {
  await FirebaseAnalytics.instance.logEvent(
    name: eventName,
    parameters: {
      'notification_type': message.data['type'],
      'campaign_id': message.data['campaign_id'] ?? 'none',
    },
  );
  
  // 也可以上傳到自己的分析系統
  await _uploadToAnalytics(eventName, message.data);
}
```

## 測試推播

開發時用 Firebase Console 手動發送測試：

1. Firebase Console → Cloud Messaging
2. Send test message
3. 輸入 FCM Token（從 console log 複製）
4. 填寫標題、內容、Data payload
5. 發送

也可以用 Postman 測試：

```
POST https://fcm.googleapis.com/fcm/send
Headers:
  Authorization: key=YOUR_SERVER_KEY
  Content-Type: application/json

Body:
{
  "to": "FCM_TOKEN",
  "notification": {
    "title": "Test",
    "body": "This is a test"
  },
  "data": {
    "type": "test",
    "key": "value"
  }
}
```

## 實務心得

推播通知看似簡單，實際上細節很多。內推和外推的處理邏輯不同，Deep Link 要測試各種場景（App 開啟、背景、關閉）。

通知頻道很重要，讓使用者可以選擇接收哪些類型的通知。不要所有通知都用同一個頻道，會被使用者關掉。

背景處理要特別注意，不能做太耗時的操作，會被系統終止。如果有複雜邏輯，應該喚醒 App 或用 WorkManager。

下週要來研究 App 的執行流程控制，什麼時候該用主線程、什麼時候該用背景線程，還有 API 錯誤訊息的處理策略。
