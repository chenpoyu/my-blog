---
layout: post
title: "Flutter 專案架構回顧與重構"
date: 2024-12-02 11:20:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Architecture, Refactoring]
---

充電站 App 開發幾個月了，這週回顧整個專案架構，整理哪些做得好、哪些需要改進。

## 初期架構決策

當初選擇 MVVM + DDD 架構。

### 為什麼選 MVVM

相比 MVC 和 MVP，MVVM 更適合 Flutter。

```
MVC: Model - View - Controller
  問題：Controller 容易變肥大，測試困難

MVP: Model - View - Presenter
  問題：Presenter 和 View 一對一綁定，重用性差

MVVM: Model - View - ViewModel
  優點：ViewModel 可重用，容易測試，配合 Riverpod 很順暢
```

### DDD 分層

```
lib/
├── core/                # 核心共用
│   ├── error/
│   ├── network/
│   └── utils/
├── features/            # 功能模組
│   ├── station/
│   │   ├── data/       # 資料層
│   │   │   ├── datasources/
│   │   │   ├── models/
│   │   │   └── repositories/
│   │   ├── domain/     # 領域層
│   │   │   ├── entities/
│   │   │   ├── repositories/
│   │   │   └── usecases/
│   │   └── presentation/ # 展示層
│   │       ├── pages/
│   │       ├── providers/
│   │       └── widgets/
│   └── charging/
│       └── ...
└── main.dart
```

這個結構清晰，但也有缺點：

**優點：**
- 職責分明，容易找檔案
- 測試隔離性好
- 團隊協作不易衝突

**缺點：**
- 小功能也要建一堆資料夾
- 檔案數量多，導航麻煩
- 過度設計（對小專案來說）

## 做對的事

### 1. 錯誤處理統一

用 `dartz` 的 `Either` 統一錯誤處理。

```dart
// 不會拋出 Exception，而是回傳 Left(Failure) 或 Right(Result)
Future<Either<Failure, List<Station>>> getStations() async {
  try {
    final response = await _api.getStations();
    return Right(response.map((json) => Station.fromJson(json)).toList());
  } on DioException catch (e) {
    return Left(NetworkFailure(e.message));
  } catch (e) {
    return Left(UnknownFailure(e.toString()));
  }
}

// 使用
final result = await repository.getStations();
result.fold(
  (failure) => showError(failure.message),
  (stations) => updateUI(stations),
);
```

這樣做的好處是錯誤不會被忽略，編譯器會強制你處理。

### 2. Riverpod 狀態管理

比 Provider 好太多了。

```dart
// 型別安全
final stationProvider = StateNotifierProvider<StationNotifier, StationState>((ref) {
  return StationNotifier(ref.watch(stationRepositoryProvider));
});

// 編譯時檢查，不會有 runtime error
final state = ref.watch(stationProvider); // OK
final state = ref.watch(unknownProvider);  // 編譯錯誤
```

### 3. 分離環境設定

開發、測試、正式環境分開。

```dart
// lib/config/env.dart
abstract class Env {
  static String get apiBaseUrl {
    if (kReleaseMode) {
      return 'https://api.production.com';
    } else if (kProfileMode) {
      return 'https://api.staging.com';
    } else {
      return 'https://api.dev.com';
    }
  }
  
  static bool get enableLogging => !kReleaseMode;
}
```

或者用 `--dart-define`：

```bash
flutter run --dart-define=ENV=dev
flutter build apk --dart-define=ENV=prod
```

### 4. 依賴注入

用 `get_it` 管理依賴。

```dart
// lib/injection.dart
final getIt = GetIt.instance;

Future<void> setupDependencies() async {
  // Network
  getIt.registerLazySingleton<Dio>(() => Dio(/* config */));
  
  // Repositories
  getIt.registerLazySingleton<StationRepository>(
    () => StationRepositoryImpl(getIt<Dio>()),
  );
  
  // UseCases
  getIt.registerLazySingleton(() => GetStationsUseCase(getIt<StationRepository>()));
}

// 測試時可以替換
void setupTestDependencies() {
  getIt.registerLazySingleton<StationRepository>(
    () => MockStationRepository(),
  );
}
```

## 做錯的事

### 1. 過度抽象

初期太追求「完美」架構。

```dart
// 過度設計：每個 API 都包一層 UseCase
class GetStationByIdUseCase {
  final StationRepository repository;
  
  GetStationByIdUseCase(this.repository);
  
  Future<Either<Failure, Station>> call(String id) {
    return repository.getStationById(id); // 只是轉發，沒有業務邏輯
  }
}

// 其實可以直接呼叫 Repository
final station = await repository.getStationById(id);
```

**教訓：** UseCase 只在有複雜業務邏輯時才需要，不要為了架構而架構。

### 2. Model 和 Entity 分離

```dart
// data/models/station_model.dart
class StationModel {
  final String id;
  final String name;
  // ...
  
  factory StationModel.fromJson(Map<String, dynamic> json) => /* */;
  Map<String, dynamic> toJson() => /* */;
}

// domain/entities/station.dart
class Station {
  final String id;
  final String name;
  // ...
}

// 需要轉換
Station toEntity() => Station(id: id, name: name);
```

這樣做理論上很「乾淨」，但實務上：
- 兩個類別 90% 相同
- 一直在轉換來轉換去
- 增加維護成本

**改進：** 直接用一個 Model，加上 `fromJson` 和 `toJson` 就好。除非 API 結構和 UI 需求差很多，否則不用分。

### 3. 檔案拆太細

```
station/
├── presentation/
│   ├── pages/
│   │   ├── station_list_page.dart
│   │   └── station_detail_page.dart
│   ├── widgets/
│   │   ├── station_card.dart
│   │   ├── station_status_indicator.dart
│   │   ├── station_availability_badge.dart
│   │   ├── station_price_label.dart
│   │   └── ... (10+ 小 widgets)
│   └── providers/
│       ├── station_list_provider.dart
│       └── station_detail_provider.dart
```

結果：
- 切換檔案很頻繁
- 小 Widget 找不到在哪
- Import 一大串

**改進：** 相關的小 Widget 放同一個檔案。

```dart
// station_widgets.dart
class StationCard extends StatelessWidget { /* */ }
class StationStatusIndicator extends StatelessWidget { /* */ }
class StationAvailabilityBadge extends StatelessWidget { /* */ }
```

只有夠複雜的 Widget 才獨立成檔案。

## 重構方向

### 1. Feature-first 結構

改成以功能為主的結構。

```
lib/
├── core/
├── features/
│   ├── station/
│   │   ├── station_models.dart      # Models + Entities
│   │   ├── station_repository.dart  # Repository + DataSource
│   │   ├── station_providers.dart   # Riverpod Providers
│   │   ├── station_list_page.dart
│   │   ├── station_detail_page.dart
│   │   └── station_widgets.dart
│   └── charging/
│       └── ...
```

減少資料夾層級，相關檔案放一起。

### 2. 合併 Model 和 Entity

```dart
// station_models.dart
class Station {
  final String id;
  final String name;
  final Location location;
  final List<Charger> chargers;
  
  Station({/* */});
  
  // JSON 序列化
  factory Station.fromJson(Map<String, dynamic> json) => Station(
    id: json['id'],
    name: json['name'],
    location: Location.fromJson(json['location']),
    chargers: (json['chargers'] as List)
        .map((e) => Charger.fromJson(e))
        .toList(),
  );
  
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'location': location.toJson(),
    'chargers': chargers.map((e) => e.toJson()).toList(),
  };
}
```

用 `freezed` 更方便：

```dart
@freezed
class Station with _$Station {
  const factory Station({
    required String id,
    required String name,
    required Location location,
    required List<Charger> chargers,
  }) = _Station;
  
  factory Station.fromJson(Map<String, dynamic> json) => 
      _$StationFromJson(json);
}
```

### 3. 簡化 UseCase

只在有複雜邏輯時才用 UseCase。

```dart
// 需要 UseCase：有業務邏輯
class StartChargingUseCase {
  final ChargingRepository repository;
  final UserRepository userRepository;
  
  Future<Either<Failure, ChargingSession>> call(String stationId) async {
    // 1. 檢查使用者餘額
    final user = await userRepository.getCurrentUser();
    if (user.balance < 100) {
      return Left(InsufficientBalanceFailure());
    }
    
    // 2. 檢查充電站狀態
    final station = await repository.getStationStatus(stationId);
    if (!station.isAvailable) {
      return Left(StationUnavailableFailure());
    }
    
    // 3. 開始充電
    return repository.startCharging(stationId);
  }
}

// 不需要 UseCase：直接呼叫 Repository
final stations = await stationRepository.getStations(filter);
```

## 效能優化回顧

### 做對的：

1. **ListView.builder** 而不是 ListView
2. **CachedNetworkImage** 快取圖片
3. **地圖 Marker Clustering** 避免渲染太多 Marker
4. **sqflite** 本地快取，減少 API 呼叫
5. **Dio Interceptor** 統一加入 Token，避免重複程式碼

### 可以改進的：

1. **圖片尺寸** - 有些圖片太大，應該在後端 resize
2. **過度 rebuild** - 某些 Widget 因為 Provider 改變而不必要地 rebuild
3. **初始載入** - App 啟動時一次載入太多資料

## 測試覆蓋率

目前：
- Unit Tests: 65%
- Widget Tests: 30%
- Integration Tests: 10%

**目標：**
- Unit Tests: 80%（核心業務邏輯）
- Widget Tests: 50%（重要 UI）
- Integration Tests: 20%（關鍵流程）

## 下一步

1. 導入 `freezed` 減少 boilerplate
2. 重構成 Feature-first 結構
3. 提升測試覆蓋率
4. 加入 CI/CD 自動化測試
5. 效能監控持續優化

下週來聊 Flutter 開發的最佳實踐和團隊協作經驗。
