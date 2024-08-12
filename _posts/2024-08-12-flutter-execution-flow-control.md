---
layout: post
title: "Flutter 執行流程控制：主線程、背景任務與錯誤處理"
date: 2024-08-12 16:20:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Async, Error Handling, 效能優化]
---

這週深入研究 Flutter 的執行流程控制。Mobile App 跟後端不同，UI 必須流暢，不能卡頓。什麼操作該在主線程、什麼該放背景，怎麼避免 API 錯誤訊息重複顯示，這些都要仔細設計。

## Flutter 的執行模型

首先要理解 Flutter 的線程模型。

### Main Isolate（UI 線程）

Flutter 的 UI 運行在 Main Isolate（主隔離區），類似其他平台的「主線程」或「UI 線程」。

所有 UI 繪製、事件處理、build widget 都在這裡執行。如果 Main Isolate 被阻塞（執行耗時操作），UI 就會卡頓。

**60 FPS 的要求**：每一幀要在 16ms 內完成，超過就會掉幀。

### 不要在 Main Isolate 做什麼

**耗時計算**：大量資料處理、複雜演算法、加密解密。

**同步 I/O**：讀寫大檔案、資料庫大量查詢。

**網路請求**：雖然 Dart 的 HTTP 是非同步的，但 JSON 解析可能耗時。

**圖片處理**：圖片解碼、縮放、濾鏡。

## 非同步操作：async/await

Dart 的非同步是用 `async`/`await`，不會阻塞 Main Isolate。

```dart
// ✅ 正確：非同步操作
Future<List<ChargingStation>> loadStations() async {
  // 雖然在 Main Isolate，但不會阻塞
  final response = await http.get(Uri.parse(apiUrl));
  final data = jsonDecode(response.body);
  return data.map((json) => ChargingStation.fromJson(json)).toList();
}

// ❌ 錯誤：同步操作（阻塞）
List<ChargingStation> loadStationsSync() {
  // 這會阻塞 UI，不要這樣做
  final response = http.get(Uri.parse(apiUrl)).then((r) => r).timeout(...);
  // ...
}
```

### 什麼時候用 async/await

**API 呼叫**：`http.get()`, `dio.get()`

**資料庫操作**：`db.query()`, `db.insert()`

**檔案讀寫**：`File.readAsString()`, `File.writeAsString()`

**延遲執行**：`Future.delayed()`

**Stream 操作**：`stream.listen()`, `await for`

這些操作看似在 Main Isolate，但實際的 I/O 在其他執行緒（由 Dart VM 管理），不會阻塞 UI。

## Isolate：真正的多執行緒

如果有 CPU 密集運算，`async/await` 還是會阻塞（因為計算本身在 Main Isolate）。要用 **Isolate**。

### 使用 compute()

Flutter 提供 `compute()` 函式，簡化 Isolate 的使用：

```dart
// 耗時的計算
List<ChargingStation> _processLargeData(List<Map<String, dynamic>> jsonList) {
  // 複雜的資料處理
  return jsonList.map((json) {
    // 假設這裡有很複雜的計算
    return ChargingStation.fromJson(json);
  }).toList();
}

// 在背景執行
Future<List<ChargingStation>> loadStations() async {
  final response = await http.get(Uri.parse(apiUrl));
  final jsonList = jsonDecode(response.body) as List;
  
  // 在另一個 Isolate 處理
  final stations = await compute(_processLargeData, jsonList);
  
  return stations;
}
```

`compute()` 會：
1. 建立新的 Isolate
2. 在新 Isolate 執行函式
3. 回傳結果
4. 清理 Isolate

### 自訂 Isolate

如果需要更複雜的背景任務（例如持續執行的 worker），用 `Isolate.spawn()`：

```dart
import 'dart:isolate';

class BackgroundProcessor {
  Isolate? _isolate;
  SendPort? _sendPort;
  
  Future<void> start() async {
    final receivePort = ReceivePort();
    
    _isolate = await Isolate.spawn(
      _backgroundTask,
      receivePort.sendPort,
    );
    
    _sendPort = await receivePort.first;
  }
  
  static void _backgroundTask(SendPort sendPort) {
    final receivePort = ReceivePort();
    sendPort.send(receivePort.sendPort);
    
    receivePort.listen((message) {
      // 處理背景任務
      final result = _processData(message);
      sendPort.send(result);
    });
  }
  
  Future<dynamic> process(dynamic data) async {
    final receivePort = ReceivePort();
    _sendPort!.send([data, receivePort.sendPort]);
    return await receivePort.first;
  }
  
  void dispose() {
    _isolate?.kill();
  }
}
```

不過實務上比較少用，`compute()` 已經夠用。

## 實戰案例：充電站資料處理

假設 API 回傳 10000 筆充電站資料，需要：
1. JSON 解析
2. 計算距離
3. 排序
4. 篩選可用的

這些計算很耗時，要放背景。

```dart
class StationDataProcessor {
  static Future<List<ChargingStation>> processStations({
    required List<Map<String, dynamic>> jsonList,
    required LatLng userLocation,
  }) async {
    return await compute(_processInBackground, {
      'jsonList': jsonList,
      'userLocation': userLocation,
    });
  }
  
  static List<ChargingStation> _processInBackground(Map<String, dynamic> params) {
    final jsonList = params['jsonList'] as List<Map<String, dynamic>>;
    final userLocation = params['userLocation'] as LatLng;
    
    // 1. 解析 JSON
    final stations = jsonList.map((json) => ChargingStation.fromJson(json)).toList();
    
    // 2. 計算距離
    for (final station in stations) {
      station.distance = _calculateDistance(
        userLocation,
        station.location,
      );
    }
    
    // 3. 排序（由近到遠）
    stations.sort((a, b) => a.distance.compareTo(b.distance));
    
    // 4. 篩選可用的
    final available = stations.where((s) => s.availableChargers > 0).toList();
    
    return available;
  }
  
  static double _calculateDistance(LatLng from, LatLng to) {
    // Haversine formula
    const earthRadius = 6371; // km
    final dLat = _toRadians(to.latitude - from.latitude);
    final dLng = _toRadians(to.longitude - from.longitude);
    
    final a = sin(dLat / 2) * sin(dLat / 2) +
        cos(_toRadians(from.latitude)) *
        cos(_toRadians(to.latitude)) *
        sin(dLng / 2) * sin(dLng / 2);
    
    final c = 2 * atan2(sqrt(a), sqrt(1 - a));
    return earthRadius * c;
  }
  
  static double _toRadians(double degrees) => degrees * pi / 180;
}

// ViewModel 使用
class MapViewModel extends StateNotifier<MapState> {
  Future<void> loadStations() async {
    state = state.copyWith(isLoading: true);
    
    try {
      // 1. 取得 API 資料（非同步，但不耗時）
      final response = await chargingApi.fetchAllStations();
      final jsonList = response.data as List<Map<String, dynamic>>;
      
      // 2. 背景處理（Isolate）
      final stations = await StationDataProcessor.processStations(
        jsonList: jsonList,
        userLocation: state.userLocation,
      );
      
      // 3. 更新 UI
      state = state.copyWith(
        isLoading: false,
        stations: stations,
      );
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        error: e.toString(),
      );
    }
  }
}
```

## 錯誤處理策略

### 問題：重複的錯誤訊息

使用者操作時，可能觸發多個 API 呼叫，如果都失敗，會跳出多個錯誤訊息，體驗很差。

```dart
// ❌ 不好的做法
Future<void> loadData() async {
  try {
    await api.fetchData();
  } catch (e) {
    // 每次錯誤都顯示
    showErrorDialog(e.toString());
  }
}
```

### 解決方案 1：錯誤訊息去重

```dart
class ErrorHandler {
  static final instance = ErrorHandler._();
  ErrorHandler._();
  
  final Set<String> _shownErrors = {};
  Timer? _clearTimer;
  
  void showError(BuildContext context, String message) {
    // 5 秒內相同錯誤不重複顯示
    if (_shownErrors.contains(message)) {
      return;
    }
    
    _shownErrors.add(message);
    
    // 5 秒後清除
    _clearTimer?.cancel();
    _clearTimer = Timer(Duration(seconds: 5), () {
      _shownErrors.clear();
    });
    
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }
}
```

### 解決方案 2：統一錯誤處理

在 Repository 層統一處理，不要讓每個 ViewModel 都處理錯誤。

```dart
abstract class Failure {
  final String message;
  final FailureType type;
  
  Failure(this.message, this.type);
}

enum FailureType {
  network,      // 網路問題（可重試）
  server,       // 伺服器錯誤
  unauthorized, // 未授權（需要登入）
  validation,   // 資料驗證錯誤
  unknown,      // 未知錯誤
}

class NetworkFailure extends Failure {
  NetworkFailure(String message) : super(message, FailureType.network);
}

class ServerFailure extends Failure {
  ServerFailure(String message) : super(message, FailureType.server);
}
```

在 ViewModel 根據錯誤類型決定是否顯示：

```dart
void _handleError(Failure failure) {
  switch (failure.type) {
    case FailureType.network:
      // 網路錯誤，顯示重試按鈕
      state = state.copyWith(
        error: '網路連線失敗',
        canRetry: true,
      );
      break;
      
    case FailureType.unauthorized:
      // 未授權，導航到登入頁
      _navigateToLogin();
      break;
      
    case FailureType.validation:
      // 驗證錯誤，顯示在表單
      state = state.copyWith(
        validationError: failure.message,
      );
      break;
      
    case FailureType.server:
    case FailureType.unknown:
      // 其他錯誤，顯示通用訊息
      _showError(failure.message);
      break;
  }
}
```

### 解決方案 3：全域錯誤攔截器

在 Dio（HTTP 套件）設定攔截器，統一處理 API 錯誤：

```dart
class ErrorInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    String message;
    FailureType type;
    
    switch (err.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        message = '連線逾時，請檢查網路';
        type = FailureType.network;
        break;
        
      case DioExceptionType.badResponse:
        final statusCode = err.response?.statusCode;
        if (statusCode == 401) {
          message = '未授權，請重新登入';
          type = FailureType.unauthorized;
        } else if (statusCode != null && statusCode >= 500) {
          message = '伺服器錯誤，請稍後再試';
          type = FailureType.server;
        } else {
          message = err.response?.data['message'] ?? '請求失敗';
          type = FailureType.validation;
        }
        break;
        
      case DioExceptionType.cancel:
        // 使用者取消，不顯示錯誤
        handler.next(err);
        return;
        
      default:
        message = '發生錯誤，請稍後再試';
        type = FailureType.unknown;
    }
    
    // 記錄錯誤
    _logError(err, message);
    
    // 轉換成自訂錯誤
    final failure = _createFailure(message, type);
    handler.reject(DioException(
      requestOptions: err.requestOptions,
      error: failure,
    ));
  }
}
```

## API 請求取消

使用者快速切換頁面，舊的 API 請求應該取消，避免浪費資源和錯誤訊息。

```dart
class MapViewModel extends StateNotifier<MapState> {
  CancelToken? _cancelToken;
  
  Future<void> loadStations() async {
    // 取消前一個請求
    _cancelToken?.cancel();
    _cancelToken = CancelToken();
    
    state = state.copyWith(isLoading: true);
    
    try {
      final response = await chargingApi.fetchStations(
        cancelToken: _cancelToken,
      );
      
      state = state.copyWith(
        isLoading: false,
        stations: response,
      );
    } on DioException catch (e) {
      if (CancelToken.isCancel(e)) {
        // 請求被取消，不處理
        return;
      }
      
      state = state.copyWith(
        isLoading: false,
        error: e.toString(),
      );
    }
  }
  
  @override
  void dispose() {
    _cancelToken?.cancel();
    super.dispose();
  }
}
```

## Loading 狀態管理

多個 API 同時呼叫，Loading 狀態要正確管理。

```dart
class MapState {
  final bool isLoadingStations;
  final bool isLoadingHistory;
  final bool isLoadingProfile;
  
  // 整體 Loading（任一個在載入）
  bool get isLoading => isLoadingStations || isLoadingHistory || isLoadingProfile;
}

class MapViewModel extends StateNotifier<MapState> {
  Future<void> loadStations() async {
    state = state.copyWith(isLoadingStations: true);
    
    try {
      final stations = await chargingApi.fetchStations();
      state = state.copyWith(
        isLoadingStations: false,
        stations: stations,
      );
    } catch (e) {
      state = state.copyWith(isLoadingStations: false);
    }
  }
  
  Future<void> loadHistory() async {
    state = state.copyWith(isLoadingHistory: true);
    
    try {
      final history = await chargingApi.fetchHistory();
      state = state.copyWith(
        isLoadingHistory: false,
        history: history,
      );
    } catch (e) {
      state = state.copyWith(isLoadingHistory: false);
    }
  }
}
```

UI 可以根據不同的 Loading 顯示不同的 indicator。

## 防抖（Debounce）與節流（Throttle）

使用者快速點擊按鈕，會觸發多次 API，要防止。

### Debounce

延遲執行，如果在延遲期間又觸發，重新計時。適合搜尋框。

```dart
class SearchViewModel extends StateNotifier<SearchState> {
  Timer? _debounce;
  
  void onSearchChanged(String query) {
    if (_debounce?.isActive ?? false) _debounce!.cancel();
    
    _debounce = Timer(Duration(milliseconds: 500), () {
      _performSearch(query);
    });
  }
  
  Future<void> _performSearch(String query) async {
    // 執行搜尋
  }
  
  @override
  void dispose() {
    _debounce?.cancel();
    super.dispose();
  }
}
```

### Throttle

限制執行頻率，一段時間內最多執行一次。適合按鈕點擊。

```dart
class ThrottleHelper {
  DateTime? _lastExecuteTime;
  final Duration duration;
  
  ThrottleHelper({this.duration = const Duration(seconds: 1)});
  
  bool shouldExecute() {
    final now = DateTime.now();
    
    if (_lastExecuteTime == null || 
        now.difference(_lastExecuteTime!) > duration) {
      _lastExecuteTime = now;
      return true;
    }
    
    return false;
  }
}

class ChargingViewModel extends StateNotifier<ChargingState> {
  final _throttle = ThrottleHelper();
  
  Future<void> startCharging() async {
    if (!_throttle.shouldExecute()) {
      return; // 太快點擊，忽略
    }
    
    // 執行充電
  }
}
```

## 實務心得

執行流程控制是 Mobile App 開發的重點，直接影響使用者體驗。

主線程絕對不能做耗時操作，這是鐵律。用 `async/await` 處理 I/O，用 `compute()` 處理 CPU 密集運算。

錯誤處理要統一，不要讓錯誤訊息滿天飛。分類錯誤類型，該重試的重試，該導航的導航，該靜默處理的就不顯示。

防抖和節流很重要，避免不必要的 API 呼叫，也避免使用者誤觸。

下週要來整理 UI/UX 的設計原則，以及在這個充電站 App 專案中實際應用的一些經驗。
