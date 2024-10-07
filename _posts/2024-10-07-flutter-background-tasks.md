---
layout: post
title: "Flutter 背景任務：WorkManager 與定時任務"
date: 2024-10-07 16:30:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Background Task, WorkManager]
---

這週處理背景任務。充電站 App 需要定期同步資料、檢查充電狀態、發送提醒等，這些都要在背景執行。

Mobile 的背景任務有限制，不像後端可以無限制執行。要遵守系統規則，否則會被終止。

## Flutter 背景執行的限制

**iOS**:
- App 進入背景後，最多執行 30 秒
- 之後會被凍結（suspended）
- 特定情況可延長：音樂播放、位置更新、VoIP、新聞下載

**Android**:
- Android 8.0+ 限制背景服務
- Doze 模式會限制背景活動
- 需要用 WorkManager 處理

## 背景任務類型

1. **短期任務**（幾秒內）：用 `compute()` 或 Isolate
2. **中期任務**（幾分鐘）：用 `workmanager`
3. **長期任務**（持續執行）：用 Foreground Service（Android）
4. **定時任務**：用 `workmanager` 的 periodic task

## WorkManager 套件

跨平台的背景任務管理。

```yaml
dependencies:
  workmanager: ^0.5.0
```

### 初始化

```dart
import 'package:workmanager/workmanager.dart';

// 背景任務 callback（必須是 top-level 函式）
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    try {
      switch (task) {
        case 'syncStations':
          await _syncStations();
          break;
        case 'checkChargingStatus':
          await _checkChargingStatus(inputData);
          break;
        case 'cleanupCache':
          await _cleanupCache();
          break;
        default:
          print('Unknown task: $task');
      }
      return Future.value(true);
    } catch (e) {
      print('Task failed: $e');
      return Future.value(false);
    }
  });
}

Future<void> _syncStations() async {
  // 初始化必要服務
  await Hive.initFlutter();
  
  // 同步充電站資料
  final api = ChargingApi();
  final stations = await api.fetchStations();
  
  // 儲存到本地資料庫
  final db = await ChargingDatabase().database;
  for (final station in stations) {
    await db.insert('stations', station.toJson(),
        conflictAlgorithm: ConflictAlgorithm.replace);
  }
  
  print('Synced ${stations.length} stations');
}

Future<void> _checkChargingStatus(Map<String, dynamic>? inputData) async {
  final sessionId = inputData?['session_id'];
  if (sessionId == null) return;
  
  await Hive.initFlutter();
  
  // 檢查充電狀態
  final api = ChargingApi();
  final session = await api.getChargingSession(sessionId);
  
  if (session.status == 'completed') {
    // 發送通知
    await _showNotification(
      '充電完成',
      '您的車輛已充電完成，共充了 ${session.energyKwh} kWh',
    );
  }
}

Future<void> _cleanupCache() async {
  await Hive.initFlutter();
  
  // 清除過期快取
  final db = await ChargingDatabase().database;
  final expiredTime = DateTime.now().subtract(Duration(days: 7));
  
  await db.delete(
    'cache',
    where: 'timestamp < ?',
    whereArgs: [expiredTime.millisecondsSinceEpoch],
  );
  
  print('Cache cleaned up');
}

Future<void> _showNotification(String title, String body) async {
  final flutterLocalNotificationsPlugin = FlutterLocalNotificationsPlugin();
  
  const androidDetails = AndroidNotificationDetails(
    'charging_channel',
    'Charging Notifications',
    importance: Importance.high,
    priority: Priority.high,
  );
  
  const iosDetails = DarwinNotificationDetails();
  
  const details = NotificationDetails(
    android: androidDetails,
    iOS: iosDetails,
  );
  
  await flutterLocalNotificationsPlugin.show(
    0,
    title,
    body,
    details,
  );
}

// 在 main 初始化
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  
  Workmanager().initialize(
    callbackDispatcher,
    isInDebugMode: true, // Release 時改為 false
  );
  
  runApp(MyApp());
}
```

### 註冊一次性任務

```dart
// 立即執行（但允許延遲）
await Workmanager().registerOneOffTask(
  'sync-task-1',           // 唯一 ID
  'syncStations',          // task 名稱（對應 switch case）
  initialDelay: Duration(seconds: 10),
);

// 帶參數
await Workmanager().registerOneOffTask(
  'check-charging-1',
  'checkChargingStatus',
  inputData: {
    'session_id': '12345',
  },
  initialDelay: Duration(minutes: 5),
);
```

### 註冊週期性任務

```dart
// 每 15 分鐘執行一次（最短間隔）
await Workmanager().registerPeriodicTask(
  'sync-periodic',
  'syncStations',
  frequency: Duration(minutes: 15),
  constraints: Constraints(
    networkType: NetworkType.connected,     // 需要網路
    requiresBatteryNotLow: true,           // 電池不能太低
    requiresCharging: false,               // 不需要充電中
    requiresDeviceIdle: false,             // 不需要裝置閒置
  ),
);

// 每日清理快取
await Workmanager().registerPeriodicTask(
  'cleanup-daily',
  'cleanupCache',
  frequency: Duration(hours: 24),
  initialDelay: Duration(hours: 1),
);
```

### 取消任務

```dart
// 取消特定任務
await Workmanager().cancelByUniqueName('sync-task-1');

// 取消所有任務
await Workmanager().cancelAll();
```

## 實戰：充電狀態輪詢

使用者開始充電後，定期檢查充電進度。

```dart
class ChargingMonitorService {
  Future<void> startMonitoring(String sessionId) async {
    await Workmanager().registerPeriodicTask(
      'monitor-$sessionId',
      'checkChargingStatus',
      frequency: Duration(minutes: 15),
      inputData: {
        'session_id': sessionId,
      },
      constraints: Constraints(
        networkType: NetworkType.connected,
      ),
    );
  }
  
  Future<void> stopMonitoring(String sessionId) async {
    await Workmanager().cancelByUniqueName('monitor-$sessionId');
  }
}

// 使用
class ChargingViewModel extends StateNotifier<ChargingState> {
  final ChargingMonitorService _monitorService;
  
  Future<void> startCharging(String stationId, String chargerId) async {
    // 開始充電
    final session = await chargingRepository.startCharging(
      stationId: stationId,
      chargerId: chargerId,
    );
    
    // 啟動背景監控
    await _monitorService.startMonitoring(session.id);
    
    state = state.copyWith(
      currentSession: session,
      isCharging: true,
    );
  }
  
  Future<void> stopCharging() async {
    if (state.currentSession != null) {
      // 停止充電
      await chargingRepository.stopCharging(state.currentSession!.id);
      
      // 停止背景監控
      await _monitorService.stopMonitoring(state.currentSession!.id);
      
      state = state.copyWith(
        currentSession: null,
        isCharging: false,
      );
    }
  }
}
```

## 本地通知

背景任務完成後常需要通知使用者。

```yaml
dependencies:
  flutter_local_notifications: ^16.0.0
```

### 初始化

```dart
class NotificationService {
  final FlutterLocalNotificationsPlugin _plugin =
      FlutterLocalNotificationsPlugin();
  
  Future<void> init() async {
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: true,
      requestBadgePermission: true,
      requestSoundPermission: true,
    );
    
    const settings = InitializationSettings(
      android: androidSettings,
      iOS: iosSettings,
    );
    
    await _plugin.initialize(
      settings,
      onDidReceiveNotificationResponse: _onNotificationTap,
    );
    
    // 建立 Android 通知頻道
    if (Platform.isAndroid) {
      await _createNotificationChannels();
    }
  }
  
  Future<void> _createNotificationChannels() async {
    const channel = AndroidNotificationChannel(
      'charging_channel',
      'Charging Notifications',
      description: 'Notifications for charging status',
      importance: Importance.high,
    );
    
    await _plugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(channel);
  }
  
  void _onNotificationTap(NotificationResponse response) {
    // 處理點擊
    final payload = response.payload;
    if (payload != null) {
      final data = jsonDecode(payload);
      _handleNotificationAction(data);
    }
  }
  
  Future<void> showNotification({
    required String title,
    required String body,
    Map<String, dynamic>? data,
  }) async {
    const androidDetails = AndroidNotificationDetails(
      'charging_channel',
      'Charging Notifications',
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
    
    await _plugin.show(
      DateTime.now().millisecond,
      title,
      body,
      details,
      payload: data != null ? jsonEncode(data) : null,
    );
  }
  
  // 排程通知
  Future<void> scheduleNotification({
    required String title,
    required String body,
    required DateTime scheduledDate,
  }) async {
    await _plugin.zonedSchedule(
      DateTime.now().millisecond,
      title,
      body,
      tz.TZDateTime.from(scheduledDate, tz.local),
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'scheduled_channel',
          'Scheduled Notifications',
        ),
        iOS: DarwinNotificationDetails(),
      ),
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }
}
```

## 前景服務（Android）

長時間執行的任務（如導航）需要前景服務。

```yaml
dependencies:
  flutter_foreground_task: ^6.0.0
```

### 設定

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
  <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
  <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
  
  <application>
    <service
      android:name="com.pravera.flutter_foreground_task.service.ForegroundService"
      android:foregroundServiceType="location"
      android:exported="false" />
  </application>
</manifest>
```

### 使用

```dart
class NavigationService {
  Future<void> startNavigation(LatLng destination) async {
    // 初始化前景任務
    FlutterForegroundTask.init(
      androidNotificationOptions: AndroidNotificationOptions(
        channelId: 'navigation_channel',
        channelName: 'Navigation',
        channelDescription: 'Navigation to charging station',
        channelImportance: NotificationChannelImportance.LOW,
        priority: NotificationPriority.LOW,
        iconData: const NotificationIconData(
          resType: ResourceType.mipmap,
          resPrefix: ResourcePrefix.ic,
          name: 'launcher',
        ),
      ),
      iosNotificationOptions: const IOSNotificationOptions(
        showNotification: true,
        playSound: false,
      ),
      foregroundTaskOptions: const ForegroundTaskOptions(
        interval: 5000, // 5 秒更新一次
        autoRunOnBoot: false,
        allowWakeLock: true,
        allowWifiLock: true,
      ),
    );
    
    // 啟動前景服務
    await FlutterForegroundTask.startService(
      notificationTitle: '導航中',
      notificationText: '正在前往充電站',
      callback: _navigationCallback,
    );
  }
  
  Future<void> stopNavigation() async {
    await FlutterForegroundTask.stopService();
  }
}

// 背景 callback
@pragma('vm:entry-point')
void _navigationCallback() {
  FlutterForegroundTask.setTaskHandler(NavigationTaskHandler());
}

class NavigationTaskHandler extends TaskHandler {
  @override
  Future<void> onStart(DateTime timestamp, SendPort? sendPort) async {
    print('Navigation started');
  }
  
  @override
  Future<void> onEvent(DateTime timestamp, SendPort? sendPort) async {
    // 每 5 秒執行
    final location = await _getCurrentLocation();
    final distance = _calculateDistance(location, destination);
    
    // 更新通知
    FlutterForegroundTask.updateService(
      notificationTitle: '導航中',
      notificationText: '距離目的地 ${distance.toStringAsFixed(1)} km',
    );
    
    // 傳送資料到 UI
    sendPort?.send(location);
  }
  
  @override
  Future<void> onDestroy(DateTime timestamp, SendPort? sendPort) async {
    print('Navigation stopped');
  }
}
```

## 位置追蹤

持續追蹤使用者位置（例如導航時）。

```yaml
dependencies:
  geolocator: ^10.0.0
```

```dart
class LocationTrackingService {
  StreamSubscription<Position>? _positionStream;
  
  Future<void> startTracking() async {
    // 檢查權限
    final permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.denied) {
      final result = await Geolocator.requestPermission();
      if (result != LocationPermission.whileInUse &&
          result != LocationPermission.always) {
        return;
      }
    }
    
    // 開始追蹤
    const locationSettings = LocationSettings(
      accuracy: LocationAccuracy.high,
      distanceFilter: 10, // 移動 10 公尺才更新
    );
    
    _positionStream = Geolocator.getPositionStream(
      locationSettings: locationSettings,
    ).listen((Position position) {
      _onLocationUpdate(position);
    });
  }
  
  void _onLocationUpdate(Position position) {
    print('Location: ${position.latitude}, ${position.longitude}');
    
    // 更新 UI 或儲存位置
    locationController.add(LatLng(
      position.latitude,
      position.longitude,
    ));
  }
  
  Future<void> stopTracking() async {
    await _positionStream?.cancel();
    _positionStream = null;
  }
}
```

## 實務心得

背景任務要謹慎使用，不要濫用系統資源。

iOS 的背景限制很嚴格，很多事情做不了。Android 較寬鬆但也有限制。

WorkManager 是跨平台方案，但不保證精確時間。系統會批次執行任務來省電。

前景服務會顯示通知，使用者可見。適合導航、音樂播放等長時間任務。

測試背景任務要用實機，模擬器行為不準確。

記得處理任務失敗的情況，設定重試邏輯。

使用者可能會關閉背景權限或通知權限，要有 fallback 機制。

下週來研究 Flutter 的 App 上架流程，包括商店截圖、描述、審核等。
