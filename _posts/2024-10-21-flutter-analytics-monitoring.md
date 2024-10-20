---
layout: post
title: "Flutter 分析與監控：Firebase Analytics 與 Crashlytics"
date: 2024-10-21 15:25:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Firebase, Analytics, Crashlytics, Monitoring]
---

App 上架後，要追蹤使用者行為和 App 健康度。這週整合 Firebase Analytics（使用者分析）和 Crashlytics（Crash 監控）。

## Firebase Analytics

追蹤使用者行為：哪些功能最常用、使用流程、留存率等。

### 安裝

```yaml
dependencies:
  firebase_core: ^2.24.0
  firebase_analytics: ^10.7.0
```

### 初始化

```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_analytics/firebase_analytics.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  static FirebaseAnalytics analytics = FirebaseAnalytics.instance;
  static FirebaseAnalyticsObserver observer =
      FirebaseAnalyticsObserver(analytics: analytics);
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      navigatorObservers: [observer], // 自動追蹤頁面瀏覽
      home: MapScreen(),
    );
  }
}
```

### 記錄事件

```dart
class AnalyticsService {
  final FirebaseAnalytics _analytics = FirebaseAnalytics.instance;
  
  // 搜尋充電站
  Future<void> logStationSearch({
    required String searchTerm,
    required int resultCount,
  }) async {
    await _analytics.logEvent(
      name: 'station_search',
      parameters: {
        'search_term': searchTerm,
        'result_count': resultCount,
        'timestamp': DateTime.now().toIso8601String(),
      },
    );
  }
  
  // 查看充電站詳情
  Future<void> logStationView(String stationId) async {
    await _analytics.logEvent(
      name: 'station_view',
      parameters: {
        'station_id': stationId,
      },
    );
  }
  
  // 開始充電
  Future<void> logChargingStart({
    required String stationId,
    required String chargerId,
  }) async {
    await _analytics.logEvent(
      name: 'charging_start',
      parameters: {
        'station_id': stationId,
        'charger_id': chargerId,
      },
    );
  }
  
  // 充電完成
  Future<void> logChargingComplete({
    required String sessionId,
    required double energyKwh,
    required int durationMinutes,
    required int costNtd,
  }) async {
    await _analytics.logEvent(
      name: 'charging_complete',
      parameters: {
        'session_id': sessionId,
        'energy_kwh': energyKwh,
        'duration_minutes': durationMinutes,
        'cost_ntd': costNtd,
      },
    );
  }
  
  // 分享充電站
  Future<void> logStationShare(String stationId, String method) async {
    await _analytics.logEvent(
      name: ShareAnalytics.share,
      parameters: {
        'content_type': 'station',
        'item_id': stationId,
        'method': method, // 'line', 'facebook', 'copy_link'
      },
    );
  }
  
  // 設定使用者屬性
  Future<void> setUserProperties({
    required String userId,
    String? preferredLanguage,
    bool? notificationsEnabled,
  }) async {
    await _analytics.setUserId(id: userId);
    
    if (preferredLanguage != null) {
      await _analytics.setUserProperty(
        name: 'preferred_language',
        value: preferredLanguage,
      );
    }
    
    if (notificationsEnabled != null) {
      await _analytics.setUserProperty(
        name: 'notifications_enabled',
        value: notificationsEnabled.toString(),
      );
    }
  }
}
```

### 預設事件

Firebase 有預設事件，建議使用：

```dart
// 應用程式開啟
_analytics.logAppOpen();

// 登入
_analytics.logLogin(loginMethod: 'email');

// 註冊
_analytics.logSignUp(signUpMethod: 'email');

// 購買（如果有付費功能）
_analytics.logPurchase(
  value: 100.0,
  currency: 'TWD',
  items: [
    AnalyticsEventItem(
      itemName: 'Premium Plan',
      itemId: 'premium_monthly',
    ),
  ],
);

// 教學完成
_analytics.logTutorialComplete();
```

### Screen View

頁面瀏覽追蹤：

```dart
class MapScreen extends StatefulWidget {
  @override
  _MapScreenState createState() => _MapScreenState();
}

class _MapScreenState extends State<MapScreen> {
  @override
  void initState() {
    super.initState();
    FirebaseAnalytics.instance.logScreenView(
      screenName: 'MapScreen',
      screenClass: 'MapScreen',
    );
  }
  
  @override
  Widget build(BuildContext context) {
    // ...
  }
}
```

### 與 Riverpod 整合

```dart
final analyticsProvider = Provider((ref) => AnalyticsService());

// 在 ViewModel 使用
class StationViewModel extends StateNotifier<StationState> {
  final AnalyticsService _analytics;
  
  StationViewModel(this._analytics) : super(StationState.initial());
  
  Future<void> viewStation(String stationId) async {
    await _analytics.logStationView(stationId);
    
    // 載入充電站資料
    final station = await repository.getStation(stationId);
    state = state.copyWith(currentStation: station);
  }
}
```

## Firebase Crashlytics

追蹤 Crash 和非致命錯誤。

### 安裝

```yaml
dependencies:
  firebase_crashlytics: ^3.4.0
```

### 初始化

```dart
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  
  // Crashlytics 初始化
  FlutterError.onError = (errorDetails) {
    FirebaseCrashlytics.instance.recordFlutterFatalError(errorDetails);
  };
  
  // PlatformDispatcher 錯誤
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };
  
  runApp(MyApp());
}
```

### 記錄錯誤

```dart
class CrashReportingService {
  final FirebaseCrashlytics _crashlytics = FirebaseCrashlytics.instance;
  
  // 記錄非致命錯誤
  Future<void> recordError(
    dynamic exception,
    StackTrace? stack, {
    String? reason,
    bool fatal = false,
  }) async {
    await _crashlytics.recordError(
      exception,
      stack,
      reason: reason,
      fatal: fatal,
    );
  }
  
  // 記錄自訂訊息
  Future<void> log(String message) async {
    await _crashlytics.log(message);
  }
  
  // 設定使用者 ID
  Future<void> setUserId(String userId) async {
    await _crashlytics.setUserId(userId);
  }
  
  // 設定自訂 Key
  Future<void> setCustomKey(String key, Object value) async {
    await _crashlytics.setCustomKey(key, value);
  }
  
  // 測試 Crash（僅用於開發）
  void testCrash() {
    _crashlytics.crash();
  }
}
```

### 在 Repository 記錄錯誤

```dart
class ChargingStationRepository {
  final CrashReportingService _crashReporting;
  
  Future<Either<Failure, List<ChargingStation>>> getNearbyStations(
    LatLng location,
    double radius,
  ) async {
    try {
      await _crashReporting.log('Fetching nearby stations');
      await _crashReporting.setCustomKey('location', location.toString());
      await _crashReporting.setCustomKey('radius', radius);
      
      final stations = await api.fetchStations(location, radius);
      return Right(stations);
    } on DioException catch (e, stack) {
      await _crashReporting.recordError(
        e,
        stack,
        reason: 'Failed to fetch stations',
      );
      return Left(NetworkFailure(e.message ?? 'Network error'));
    } catch (e, stack) {
      await _crashReporting.recordError(
        e,
        stack,
        reason: 'Unexpected error in getNearbyStations',
        fatal: false,
      );
      return Left(UnknownFailure(e.toString()));
    }
  }
}
```

### Breadcrumbs

記錄使用者操作流程，幫助重現 Crash：

```dart
class BreadcrumbService {
  final CrashReportingService _crashReporting;
  
  Future<void> recordUserAction(String action, [Map<String, dynamic>? data]) async {
    final message = data != null
        ? '$action: ${jsonEncode(data)}'
        : action;
    
    await _crashReporting.log(message);
  }
}

// 使用
class ChargingViewModel extends StateNotifier<ChargingState> {
  final BreadcrumbService _breadcrumb;
  
  Future<void> startCharging(String stationId) async {
    await _breadcrumb.recordUserAction('start_charging', {
      'station_id': stationId,
      'timestamp': DateTime.now().toIso8601String(),
    });
    
    try {
      // 開始充電邏輯
      await chargingRepository.startCharging(stationId);
    } catch (e, stack) {
      await _crashReporting.recordError(e, stack);
    }
  }
}
```

## Performance Monitoring

追蹤 App 效能。

```yaml
dependencies:
  firebase_performance: ^0.9.0
```

### 追蹤網路請求

```dart
import 'package:firebase_performance/firebase_performance.dart';

class PerformanceInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final httpMetric = FirebasePerformance.instance.newHttpMetric(
      options.uri.toString(),
      HttpMethod.values.firstWhere(
        (m) => m.toString().split('.').last == options.method.toLowerCase(),
      ),
    );
    
    options.extra['http_metric'] = httpMetric;
    httpMetric.start();
    
    handler.next(options);
  }
  
  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    final httpMetric = response.requestOptions.extra['http_metric'] as HttpMetric?;
    
    httpMetric?.responseCode = response.statusCode;
    httpMetric?.responsePayloadSize = response.data.toString().length;
    httpMetric?.stop();
    
    handler.next(response);
  }
  
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final httpMetric = err.requestOptions.extra['http_metric'] as HttpMetric?;
    
    httpMetric?.responseCode = err.response?.statusCode ?? 0;
    httpMetric?.stop();
    
    handler.next(err);
  }
}
```

### 追蹤自訂操作

```dart
Future<void> loadStations() async {
  final trace = FirebasePerformance.instance.newTrace('load_stations');
  await trace.start();
  
  try {
    final stations = await api.fetchStations();
    trace.setMetric('station_count', stations.length);
    
    await trace.stop();
  } catch (e) {
    trace.setMetric('failed', 1);
    await trace.stop();
    rethrow;
  }
}
```

## 遠端設定 (Remote Config)

動態調整 App 行為，不用發布新版本。

```yaml
dependencies:
  firebase_remote_config: ^4.3.0
```

```dart
class RemoteConfigService {
  final FirebaseRemoteConfig _remoteConfig = FirebaseRemoteConfig.instance;
  
  Future<void> init() async {
    await _remoteConfig.setConfigSettings(RemoteConfigSettings(
      fetchTimeout: Duration(minutes: 1),
      minimumFetchInterval: Duration(hours: 1), // Production: 12 hours
    ));
    
    // 設定預設值
    await _remoteConfig.setDefaults({
      'show_promotion_banner': false,
      'min_app_version': '1.0.0',
      'maintenance_mode': false,
      'api_base_url': 'https://api.example.com',
    });
    
    // Fetch and activate
    await _remoteConfig.fetchAndActivate();
  }
  
  bool getShowPromotionBanner() {
    return _remoteConfig.getBool('show_promotion_banner');
  }
  
  String getMinAppVersion() {
    return _remoteConfig.getString('min_app_version');
  }
  
  bool isMaintenanceMode() {
    return _remoteConfig.getBool('maintenance_mode');
  }
  
  String getApiBaseUrl() {
    return _remoteConfig.getString('api_base_url');
  }
}

// 檢查版本
Future<void> checkAppVersion() async {
  final remoteConfig = RemoteConfigService();
  final minVersion = remoteConfig.getMinAppVersion();
  final currentVersion = '1.0.0'; // 從 package_info_plus 取得
  
  if (_isVersionOlder(currentVersion, minVersion)) {
    // 顯示強制更新對話框
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (context) => AlertDialog(
        title: Text('需要更新'),
        content: Text('請更新到最新版本以繼續使用'),
        actions: [
          ElevatedButton(
            onPressed: () => _openAppStore(),
            child: Text('前往更新'),
          ),
        ],
      ),
    );
  }
}
```

## A/B Testing

測試不同版本的 UI 或功能。

```dart
Future<void> setupABTest() async {
  final remoteConfig = FirebaseRemoteConfig.instance;
  
  await remoteConfig.setDefaults({
    'button_color': 'green',
    'show_tutorial': true,
  });
  
  await remoteConfig.fetchAndActivate();
  
  // 根據 Remote Config 顯示不同版本
  final buttonColor = remoteConfig.getString('button_color');
  final showTutorial = remoteConfig.getBool('show_tutorial');
  
  return MaterialApp(
    theme: ThemeData(
      primaryColor: buttonColor == 'green' ? Colors.green : Colors.blue,
    ),
    home: showTutorial ? TutorialScreen() : MapScreen(),
  );
}
```

## Dashboard 追蹤

Firebase Console 提供豐富的分析報表：

**Analytics**:
- 使用者數量（DAU, MAU）
- 留存率
- 事件統計
- 使用者旅程

**Crashlytics**:
- Crash-free 使用者比例
- Crash 排名
- Stack trace
- 受影響裝置

**Performance**:
- App 啟動時間
- 畫面渲染時間
- 網路請求效能
- 自訂 Trace

## 實務心得

分析和監控是持續改善 App 的基礎。沒有數據就是瞎子摸象。

Event 命名要一致，建議建立 constants 檔案統一管理。

不要追蹤過多事件，聚焦在關鍵行為。太多事件反而難以分析。

Crashlytics 的 Breadcrumbs 很重要，幫助重現複雜的 Crash 情境。

Remote Config 可以做功能開關，漸進式發布新功能。

A/B Testing 要有假設和目標，不是亂測試。

隱私很重要，追蹤前要告知使用者並取得同意。

下週要來研究 Flutter 的安全性，包括程式碼混淆、資料加密等。
