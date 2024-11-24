---
layout: post
title: "Flutter 效能監控與 APM 工具"
date: 2024-11-25 14:30:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Performance, APM, Monitoring]
---

App 上線後，效能問題才真正浮現。這週研究如何監控線上 App 效能，及時發現和解決問題。

## Firebase Performance Monitoring

最常用的免費 APM 工具。

### 基本設定

```yaml
dependencies:
  firebase_core: ^2.24.0
  firebase_performance: ^0.9.3
```

```dart
// main.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_performance/firebase_performance.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  
  // 啟用效能監控
  FirebasePerformance.instance.setPerformanceCollectionEnabled(true);
  
  runApp(MyApp());
}
```

### 追蹤網路請求

```dart
class ApiClient {
  final Dio _dio;
  final FirebasePerformance _performance = FirebasePerformance.instance;
  
  ApiClient(this._dio) {
    _dio.interceptors.add(
      InterceptorsWrapper(
        onRequest: (options, handler) async {
          // 開始追蹤
          final metric = _performance.newHttpMetric(
            options.uri.toString(),
            HttpMethod.values.firstWhere(
              (e) => e.toString().split('.').last.toUpperCase() == 
                     options.method.toUpperCase(),
            ),
          );
          
          options.extra['metric'] = metric;
          await metric.start();
          
          handler.next(options);
        },
        onResponse: (response, handler) async {
          // 記錄成功
          final metric = response.requestOptions.extra['metric'] 
              as HttpMetric?;
          
          if (metric != null) {
            metric.responseContentType = response.headers['content-type']?.first;
            metric.httpResponseCode = response.statusCode;
            metric.responsePayloadSize = 
                response.data.toString().length;
            await metric.stop();
          }
          
          handler.next(response);
        },
        onError: (error, handler) async {
          // 記錄錯誤
          final metric = error.requestOptions.extra['metric'] 
              as HttpMetric?;
          
          if (metric != null) {
            metric.httpResponseCode = error.response?.statusCode;
            await metric.stop();
          }
          
          handler.next(error);
        },
      ),
    );
  }
}
```

### 自訂追蹤指標

追蹤關鍵業務流程。

```dart
class ChargingService {
  final FirebasePerformance _performance = FirebasePerformance.instance;
  
  Future<void> startCharging(String stationId) async {
    // 建立自訂 Trace
    final trace = _performance.newTrace('start_charging');
    
    // 加入自訂屬性
    trace.putAttribute('station_id', stationId);
    trace.putAttribute('user_type', 'premium');
    
    await trace.start();
    
    try {
      // 執行充電流程
      await _connectToStation(stationId);
      
      // 記錄自訂指標
      trace.incrementMetric('connection_attempts', 1);
      
      await _authenticateUser();
      await _startChargingSession();
      
      trace.putAttribute('result', 'success');
    } catch (e) {
      trace.putAttribute('result', 'failure');
      trace.putAttribute('error', e.toString());
      rethrow;
    } finally {
      await trace.stop();
    }
  }
  
  Future<List<Station>> searchStations(String keyword) async {
    final trace = _performance.newTrace('search_stations');
    trace.putAttribute('keyword', keyword);
    
    await trace.start();
    
    final result = await _repository.search(keyword);
    
    // 記錄結果數量
    trace.incrementMetric('result_count', result.length);
    
    await trace.stop();
    
    return result;
  }
}
```

### 監控畫面載入時間

```dart
class StationDetailScreen extends StatefulWidget {
  final String stationId;
  
  const StationDetailScreen({required this.stationId});
  
  @override
  _StationDetailScreenState createState() => _StationDetailScreenState();
}

class _StationDetailScreenState extends State<StationDetailScreen> {
  final _performance = FirebasePerformance.instance;
  Trace? _screenTrace;
  
  @override
  void initState() {
    super.initState();
    _startScreenTrace();
    _loadData();
  }
  
  void _startScreenTrace() async {
    _screenTrace = _performance.newTrace('station_detail_screen');
    _screenTrace?.putAttribute('station_id', widget.stationId);
    await _screenTrace?.start();
  }
  
  Future<void> _loadData() async {
    final dataTrace = _performance.newTrace('load_station_data');
    await dataTrace.start();
    
    try {
      // 載入資料
      await Future.wait([
        _loadStationInfo(),
        _loadAvailability(),
        _loadReviews(),
      ]);
      
      dataTrace.putAttribute('result', 'success');
    } catch (e) {
      dataTrace.putAttribute('result', 'failure');
    } finally {
      await dataTrace.stop();
      await _screenTrace?.stop();
    }
  }
  
  @override
  void dispose() {
    _screenTrace?.stop();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(/* UI */);
  }
}
```

## Sentry 整合

更強大的錯誤追蹤和效能監控。

### 設定

```yaml
dependencies:
  sentry_flutter: ^7.14.0
```

```dart
import 'package:sentry_flutter/sentry_flutter.dart';

Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = 'your_sentry_dsn';
      options.tracesSampleRate = 1.0; // 100% 追蹤（生產環境建議降低）
      options.profilesSampleRate = 1.0; // 效能分析
      
      // 自動追蹤
      options.enableAutoPerformanceTracing = true;
      options.enableUserInteractionTracing = true;
      
      // 環境設定
      options.environment = kReleaseMode ? 'production' : 'development';
      
      // 篩選敏感資訊
      options.beforeSend = (event, hint) {
        // 移除敏感資料
        event = event.copyWith(
          user: event.user?.copyWith(
            email: null, // 不要傳 email
            ipAddress: null,
          ),
        );
        return event;
      };
    },
    appRunner: () => runApp(MyApp()),
  );
}
```

### 手動追蹤交易

```dart
class ChargingService {
  Future<void> startCharging(String stationId) async {
    final transaction = Sentry.startTransaction(
      'start_charging',
      'charging',
    );
    
    try {
      // 連接充電站
      final connectSpan = transaction.startChild('connect_station');
      connectSpan.setData('station_id', stationId);
      await _connectToStation(stationId);
      await connectSpan.finish();
      
      // 認證
      final authSpan = transaction.startChild('authenticate');
      await _authenticateUser();
      await authSpan.finish();
      
      // 開始充電
      final startSpan = transaction.startChild('start_session');
      await _startChargingSession();
      await startSpan.finish();
      
      transaction.status = SpanStatus.ok();
    } catch (e, stackTrace) {
      transaction.throwable = e;
      transaction.status = SpanStatus.internalError();
      
      // 回報錯誤
      await Sentry.captureException(
        e,
        stackTrace: stackTrace,
        hint: Hint.withMap({'station_id': stationId}),
      );
      
      rethrow;
    } finally {
      await transaction.finish();
    }
  }
}
```

### 監控網路請求

```dart
import 'package:sentry_dio/sentry_dio.dart';

class ApiClient {
  late Dio _dio;
  
  ApiClient() {
    _dio = Dio();
    
    // 加入 Sentry Interceptor
    _dio.addSentry(
      captureFailedRequests: true,
      maxRequestBodySize: MaxRequestBodySize.always,
    );
  }
}
```

### 效能麵包屑

記錄使用者操作路徑。

```dart
class NavigationObserver extends RouteObserver<PageRoute<dynamic>> {
  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) {
    super.didPush(route, previousRoute);
    
    if (route is PageRoute) {
      Sentry.addBreadcrumb(
        Breadcrumb(
          message: 'Navigation',
          category: 'navigation',
          data: {
            'to': route.settings.name,
            'from': previousRoute?.settings.name,
          },
          level: SentryLevel.info,
        ),
      );
    }
  }
}

// 在 MaterialApp 使用
MaterialApp(
  navigatorObservers: [
    SentryNavigatorObserver(),
  ],
);
```

## 自訂效能監控

建立自己的 APM 系統。

```dart
class PerformanceMonitor {
  static final Map<String, Stopwatch> _timers = {};
  static final List<PerformanceMetric> _metrics = [];
  
  // 開始計時
  static void startTimer(String name) {
    _timers[name] = Stopwatch()..start();
  }
  
  // 結束計時
  static void stopTimer(String name, {Map<String, dynamic>? attributes}) {
    final timer = _timers[name];
    if (timer == null) return;
    
    timer.stop();
    
    final metric = PerformanceMetric(
      name: name,
      duration: timer.elapsedMilliseconds,
      timestamp: DateTime.now(),
      attributes: attributes ?? {},
    );
    
    _metrics.add(metric);
    _timers.remove(name);
    
    // 超過閾值就警告
    if (timer.elapsedMilliseconds > 1000) {
      print('⚠️ Slow operation: $name took ${timer.elapsedMilliseconds}ms');
    }
    
    // 定期上傳指標
    if (_metrics.length >= 100) {
      _uploadMetrics();
    }
  }
  
  // 上傳指標到後端
  static Future<void> _uploadMetrics() async {
    if (_metrics.isEmpty) return;
    
    try {
      final metricsToUpload = List<PerformanceMetric>.from(_metrics);
      _metrics.clear();
      
      // 上傳到你的 APM 後端
      await _api.uploadMetrics(metricsToUpload);
    } catch (e) {
      print('Failed to upload metrics: $e');
    }
  }
  
  // 記錄記憶體使用
  static Future<void> logMemoryUsage() async {
    final info = await DeviceInfoPlugin().androidInfo;
    // 記錄記憶體資訊
  }
}

class PerformanceMetric {
  final String name;
  final int duration;
  final DateTime timestamp;
  final Map<String, dynamic> attributes;
  
  PerformanceMetric({
    required this.name,
    required this.duration,
    required this.timestamp,
    required this.attributes,
  });
  
  Map<String, dynamic> toJson() => {
    'name': name,
    'duration': duration,
    'timestamp': timestamp.toIso8601String(),
    'attributes': attributes,
  };
}

// 使用
Future<void> loadStations() async {
  PerformanceMonitor.startTimer('load_stations');
  
  try {
    final stations = await _repository.getStations();
    
    PerformanceMonitor.stopTimer('load_stations', attributes: {
      'count': stations.length,
      'success': true,
    });
  } catch (e) {
    PerformanceMonitor.stopTimer('load_stations', attributes: {
      'success': false,
      'error': e.toString(),
    });
  }
}
```

## 監控關鍵指標

定義和追蹤 KPI。

```dart
class AppMetrics {
  static final FirebaseAnalytics _analytics = FirebaseAnalytics.instance;
  
  // App 啟動時間
  static void logAppStartTime(Duration duration) {
    _analytics.logEvent(
      name: 'app_start_time',
      parameters: {
        'duration_ms': duration.inMilliseconds,
      },
    );
  }
  
  // 充電成功率
  static void logChargingResult(bool success, String reason) {
    _analytics.logEvent(
      name: 'charging_result',
      parameters: {
        'success': success,
        'reason': reason,
      },
    );
  }
  
  // 頁面載入時間
  static void logScreenLoadTime(String screenName, Duration duration) {
    _analytics.logEvent(
      name: 'screen_load_time',
      parameters: {
        'screen_name': screenName,
        'duration_ms': duration.inMilliseconds,
      },
    );
  }
  
  // API 回應時間
  static void logApiResponseTime(String endpoint, Duration duration) {
    _analytics.logEvent(
      name: 'api_response_time',
      parameters: {
        'endpoint': endpoint,
        'duration_ms': duration.inMilliseconds,
      },
    );
  }
}
```

## 實務心得

APM 工具要在開發早期就整合，不要等上線才裝。

Firebase Performance 免費且易用，適合快速起步。

Sentry 功能更強大，錯誤追蹤和效能監控都很完整。

自訂指標要有意義，追蹤對業務重要的流程。

注意取樣率，100% 追蹤會增加網路和儲存成本。

定期檢視監控數據，發現異常趨勢及早處理。

敏感資訊不要上傳到第三方平台。

效能問題通常在真實使用情境才會出現，模擬器測不準。

下週做個 Flutter 專案的年終回顧，整理這幾個月的學習心得。
