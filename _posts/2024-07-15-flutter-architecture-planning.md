---
layout: post
title: "Flutter 專案啟動：架構選擇與專案規劃"
date: 2024-07-15 13:25:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, MVVM, DDD, 架構設計]
---

公司接了一個電車充電站的 App 專案，選用 Flutter 作為開發框架。之前有研究過 Flutter，但這次是真正要上線的產品，要考慮的東西更多。

這週主要在做架構規劃，確定技術方向，避免之後重構。

## 為什麼選 Flutter

其實一開始有討論要用 Native（iOS + Android）還是跨平台。最後選 Flutter 的原因：

**一套程式碼跨平台**：不用維護兩份 code，開發效率高。

**效能不錯**：Flutter 直接編譯成 native code，不像某些框架用 WebView，效能接近原生。

**UI 彈性大**：Material 和 Cupertino 兩套 Widget，也能完全客製化。這個專案的 UI 比較特殊，需要客製。

**生態系成熟**：需要的套件（地圖、推播、資料庫）都有成熟的解決方案。

**團隊熟悉 Dart**：之前有人用過，學習成本低。

## 架構選擇：MVVM + DDD

專案要維護好幾年，架構很重要。團隊討論後決定用 **MVVM + DDD**。

### MVVM（Model-View-ViewModel）

```
View (UI)
  ↕
ViewModel (業務邏輯 + 狀態管理)
  ↕
Model (資料模型)
  ↕
Repository (資料來源)
```

**View**：純 UI，顯示資料和接收使用者操作，不包含業務邏輯。

**ViewModel**：處理業務邏輯、狀態管理、呼叫 Repository。View 透過 ViewModel 取得資料和觸發操作。

**Model**：資料模型，單純的 Dart class。

**Repository**：統一的資料來源介面，處理 API、資料庫、快取等。

### DDD（Domain-Driven Design）

DDD 強調「領域模型」，把業務邏輯和資料存取分離。

專案分層：

```
lib/
  ├── presentation/     # UI 層（View + ViewModel）
  ├── domain/          # 領域層（業務邏輯、Entity、Use Case）
  ├── data/            # 資料層（Repository、API、Database）
  └── core/            # 共用工具（常數、擴充、工具類）
```

**Domain 層**：
- Entity：核心業務物件（充電站、充電紀錄、使用者）
- Use Case：具體的業務操作（搜尋充電站、開始充電、結束充電）
- Repository Interface：定義資料操作介面

**Data 層**：
- Repository Implementation：實作 Domain 層定義的介面
- Data Source：API、資料庫、快取
- DTO（Data Transfer Object）：API 回傳的資料格式

**Presentation 層**：
- View（Widget）：UI 元件
- ViewModel：狀態管理，呼叫 Use Case

### 為什麼用這套架構

**關注點分離**：UI、業務邏輯、資料存取各司其職，修改一層不影響其他層。

**可測試性**：每層都能獨立測試。ViewModel 不依賴具體的 Repository 實作，可以用 Mock 測試。

**可維護性**：新增功能或修改邏輯，知道要改哪一層。

**團隊協作**：不同人可以同時開發不同層，減少衝突。

## 專案結構

實際的資料夾結構：

```
lib/
├── main.dart
├── app.dart
│
├── core/
│   ├── constants/           # 常數（API URL、顏色、文字）
│   ├── errors/             # 錯誤定義
│   ├── extensions/         # Dart 擴充
│   ├── themes/             # 主題設定
│   ├── utils/              # 工具類（日期、格式化）
│   └── widgets/            # 共用 Widget
│
├── domain/
│   ├── entities/           # 業務實體
│   │   ├── charging_station.dart
│   │   ├── charging_session.dart
│   │   └── user.dart
│   │
│   ├── repositories/       # Repository 介面
│   │   ├── charging_station_repository.dart
│   │   └── user_repository.dart
│   │
│   └── usecases/          # Use Case
│       ├── get_nearby_stations.dart
│       ├── start_charging.dart
│       └── get_charging_history.dart
│
├── data/
│   ├── models/            # DTO（對應 API 格式）
│   │   ├── charging_station_model.dart
│   │   └── user_model.dart
│   │
│   ├── repositories/      # Repository 實作
│   │   └── charging_station_repository_impl.dart
│   │
│   └── datasources/       # 資料來源
│       ├── remote/        # API
│       │   └── charging_api.dart
│       └── local/         # 本地資料庫
│           └── charging_database.dart
│
└── presentation/
    ├── map/               # 地圖功能
    │   ├── map_screen.dart
    │   └── map_viewmodel.dart
    │
    ├── station_detail/    # 充電站詳情
    │   ├── station_detail_screen.dart
    │   └── station_detail_viewmodel.dart
    │
    └── charging/          # 充電中畫面
        ├── charging_screen.dart
        └── charging_viewmodel.dart
```

## 依賴注入

MVVM + DDD 會有很多依賴關係，用 **get_it** 做依賴注入（DI）。

```dart
// core/di/injection.dart
final getIt = GetIt.instance;

void setupDependencies() {
  // Data sources
  getIt.registerLazySingleton<ChargingApi>(() => ChargingApi());
  getIt.registerLazySingleton<ChargingDatabase>(() => ChargingDatabase());
  
  // Repositories
  getIt.registerLazySingleton<ChargingStationRepository>(
    () => ChargingStationRepositoryImpl(
      remoteDataSource: getIt(),
      localDataSource: getIt(),
    ),
  );
  
  // Use cases
  getIt.registerLazySingleton(() => GetNearbyStations(getIt()));
  getIt.registerLazySingleton(() => StartCharging(getIt()));
  
  // ViewModels
  getIt.registerFactory(() => MapViewModel(
    getNearbyStations: getIt(),
  ));
}
```

在 `main.dart` 初始化：

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  setupDependencies();
  runApp(MyApp());
}
```

使用時：

```dart
class MapScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final viewModel = getIt<MapViewModel>();
    // ...
  }
}
```

## 狀態管理

Flutter 狀態管理方案很多（Provider、Bloc、Riverpod、GetX），團隊選用 **Riverpod**。

原因：
- 編譯期檢查，型別安全
- 不依賴 BuildContext
- 測試友善
- 效能好

ViewModel 用 `StateNotifier` + `StateNotifierProvider`：

```dart
// presentation/map/map_viewmodel.dart
class MapViewModel extends StateNotifier<MapState> {
  final GetNearbyStations getNearbyStations;
  
  MapViewModel({required this.getNearbyStations}) 
    : super(MapState.initial());
  
  Future<void> loadStations(LatLng location) async {
    state = state.copyWith(isLoading: true);
    
    final result = await getNearbyStations.execute(location);
    
    result.fold(
      (failure) => state = state.copyWith(
        isLoading: false,
        error: failure.message,
      ),
      (stations) => state = state.copyWith(
        isLoading: false,
        stations: stations,
      ),
    );
  }
}

// Provider
final mapViewModelProvider = StateNotifierProvider<MapViewModel, MapState>(
  (ref) => MapViewModel(getNearbyStations: getIt()),
);
```

## Entity vs Model 差異

**Entity（領域實體）**：業務物件，不包含序列化邏輯。

```dart
// domain/entities/charging_station.dart
class ChargingStation {
  final String id;
  final String name;
  final LatLng location;
  final int availableChargers;
  final double pricePerKwh;
  
  ChargingStation({
    required this.id,
    required this.name,
    required this.location,
    required this.availableChargers,
    required this.pricePerKwh,
  });
}
```

**Model（資料模型）**：對應 API 格式，包含序列化。

```dart
// data/models/charging_station_model.dart
class ChargingStationModel {
  final String id;
  final String name;
  final double latitude;
  final double longitude;
  final int availableChargers;
  final double pricePerKwh;
  
  ChargingStationModel({...});
  
  factory ChargingStationModel.fromJson(Map<String, dynamic> json) {
    return ChargingStationModel(
      id: json['id'],
      name: json['name'],
      latitude: json['lat'],
      longitude: json['lng'],
      availableChargers: json['available_chargers'],
      pricePerKwh: json['price_per_kwh'].toDouble(),
    );
  }
  
  // 轉成 Entity
  ChargingStation toEntity() {
    return ChargingStation(
      id: id,
      name: name,
      location: LatLng(latitude, longitude),
      availableChargers: availableChargers,
      pricePerKwh: pricePerKwh,
    );
  }
}
```

Repository 負責轉換：

```dart
@override
Future<Either<Failure, List<ChargingStation>>> getNearbyStations(
  LatLng location,
  double radius,
) async {
  try {
    final models = await remoteDataSource.fetchNearbyStations(
      location.latitude,
      location.longitude,
      radius,
    );
    
    final entities = models.map((model) => model.toEntity()).toList();
    return Right(entities);
  } catch (e) {
    return Left(ServerFailure(e.toString()));
  }
}
```

## 錯誤處理

用 **dartz** 的 `Either` 處理錯誤，避免 try-catch 到處飛。

```dart
// core/errors/failures.dart
abstract class Failure {
  final String message;
  Failure(this.message);
}

class ServerFailure extends Failure {
  ServerFailure(String message) : super(message);
}

class CacheFailure extends Failure {
  CacheFailure(String message) : super(message);
}

class NetworkFailure extends Failure {
  NetworkFailure(String message) : super(message);
}
```

Use Case 回傳 `Either<Failure, T>`：

```dart
class GetNearbyStations {
  final ChargingStationRepository repository;
  
  GetNearbyStations(this.repository);
  
  Future<Either<Failure, List<ChargingStation>>> execute(
    LatLng location,
  ) async {
    return await repository.getNearbyStations(location, 5.0);
  }
}
```

ViewModel 處理結果：

```dart
result.fold(
  (failure) {
    // 處理錯誤
    if (failure is NetworkFailure) {
      showError('網路連線失敗');
    } else {
      showError(failure.message);
    }
  },
  (stations) {
    // 處理成功
    state = state.copyWith(stations: stations);
  },
);
```

## 初步心得

花了幾天規劃架構，雖然一開始比較花時間，但對後續開發很重要。

MVVM + DDD 的分層清楚，每個檔案職責明確。雖然檔案變多了，但找東西很快，不會像以前全部寫在一起，改個小功能要翻很久。

Riverpod 的學習曲線有點陡，但用起來確實比 Provider 好用，尤其是不依賴 BuildContext 這點。

下週要開始實作具體功能，先從地圖和充電站列表開始。還要研究本地資料庫的設計，這個專案對快取的要求蠻高的，要減少 API 呼叫。
