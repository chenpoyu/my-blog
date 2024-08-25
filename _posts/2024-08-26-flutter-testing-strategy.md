---
layout: post
title: "Flutter 測試策略：Unit Test、Widget Test 與 Integration Test"
date: 2024-08-26 10:35:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Testing, Quality Assurance]
---

開發到一個階段，測試變得很重要。這週開始補測試，充電站 App 的核心功能要確保品質，不能每次改點東西就要手動測一遍。

Flutter 的測試分三層：**Unit Test**（單元測試）、**Widget Test**（元件測試）、**Integration Test**（整合測試）。每一層測試不同面向，搭配使用才能完整覆蓋。

## 測試金字塔

測試金字塔的概念：底層多、上層少。

```
       /\
      /IT\      Integration Test (少量，慢，接近真實)
     /----\
    /Widget\    Widget Test (中量，快，UI邏輯)
   /--------\
  /Unit Test\  Unit Test (大量，超快，純邏輯)
 /------------\
```

**Unit Test** 最多，測試 business logic、data layer、use cases。

**Widget Test** 中等，測試 UI 元件的行為、狀態變化。

**Integration Test** 最少，測試完整流程（像真實使用者操作）。

## 環境設定

Flutter 專案預設就有測試環境，在 `test/` 目錄。

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^5.4.0           # Mock 工具
  build_runner: ^2.4.0      # 產生 Mock 程式碼
  integration_test:
    sdk: flutter            # Integration Test
```

## Unit Test：測試 Business Logic

### 測試 Use Case

```dart
// test/domain/use_cases/get_nearby_stations_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:mockito/annotations.dart';
import 'package:dartz/dartz.dart';

// 產生 Mock
@GenerateMocks([ChargingStationRepository])
import 'get_nearby_stations_test.mocks.dart';

void main() {
  late GetNearbyStations useCase;
  late MockChargingStationRepository mockRepository;
  
  setUp(() {
    mockRepository = MockChargingStationRepository();
    useCase = GetNearbyStations(mockRepository);
  });
  
  group('GetNearbyStations', () {
    final tLocation = LatLng(25.0330, 121.5654);
    final tRadius = 5.0;
    final tStations = [
      ChargingStation(
        id: '1',
        name: 'Test Station',
        location: LatLng(25.0340, 121.5660),
      ),
    ];
    
    test('should get stations from repository', () async {
      // arrange
      when(mockRepository.getNearbyStations(any, any))
          .thenAnswer((_) async => Right(tStations));
      
      // act
      final result = await useCase(
        location: tLocation,
        radius: tRadius,
      );
      
      // assert
      expect(result, Right(tStations));
      verify(mockRepository.getNearbyStations(tLocation, tRadius));
      verifyNoMoreInteractions(mockRepository);
    });
    
    test('should return NetworkFailure when network error', () async {
      // arrange
      when(mockRepository.getNearbyStations(any, any))
          .thenAnswer((_) async => Left(NetworkFailure('Connection failed')));
      
      // act
      final result = await useCase(
        location: tLocation,
        radius: tRadius,
      );
      
      // assert
      expect(result, Left(NetworkFailure('Connection failed')));
    });
  });
}
```

執行測試：

```bash
# 產生 Mock
flutter pub run build_runner build

# 執行測試
flutter test test/domain/use_cases/get_nearby_stations_test.dart
```

### 測試 Data Layer

測試 Repository 實作：

```dart
// test/data/repositories/charging_station_repository_impl_test.dart
void main() {
  late ChargingStationRepositoryImpl repository;
  late MockChargingApi mockApi;
  late MockChargingDatabase mockDatabase;
  
  setUp(() {
    mockApi = MockChargingApi();
    mockDatabase = MockChargingDatabase();
    repository = ChargingStationRepositoryImpl(
      api: mockApi,
      database: mockDatabase,
    );
  });
  
  group('getNearbyStations', () {
    test('should return cached data when available and valid', () async {
      // arrange
      final cachedStations = [/* cached data */];
      when(mockDatabase.getStations(any, any))
          .thenAnswer((_) async => cachedStations);
      when(mockDatabase.isCacheValid(any))
          .thenAnswer((_) async => true);
      
      // act
      final result = await repository.getNearbyStations(
        LatLng(25.0330, 121.5654),
        5.0,
      );
      
      // assert
      expect(result.isRight(), true);
      result.fold(
        (l) => fail('Should return stations'),
        (stations) => expect(stations, cachedStations),
      );
      
      // 應該讀快取，不呼叫 API
      verify(mockDatabase.getStations(any, any));
      verifyNever(mockApi.fetchStations(any, any));
    });
    
    test('should fetch from API when cache is invalid', () async {
      // arrange
      final apiResponse = [/* API data */];
      when(mockDatabase.isCacheValid(any))
          .thenAnswer((_) async => false);
      when(mockApi.fetchStations(any, any))
          .thenAnswer((_) async => apiResponse);
      when(mockDatabase.saveStations(any))
          .thenAnswer((_) async => {});
      
      // act
      final result = await repository.getNearbyStations(
        LatLng(25.0330, 121.5654),
        5.0,
      );
      
      // assert
      expect(result.isRight(), true);
      verify(mockApi.fetchStations(any, any));
      verify(mockDatabase.saveStations(any));
    });
  });
}
```

### 測試 ViewModel

測試 StateNotifier：

```dart
// test/presentation/map/map_view_model_test.dart
void main() {
  late MapViewModel viewModel;
  late MockGetNearbyStations mockGetNearbyStations;
  
  setUp(() {
    mockGetNearbyStations = MockGetNearbyStations();
    viewModel = MapViewModel(
      getNearbyStations: mockGetNearbyStations,
    );
  });
  
  tearDown(() {
    viewModel.dispose();
  });
  
  group('loadStations', () {
    test('should update state to loading then success', () async {
      // arrange
      final tStations = [/* test data */];
      when(mockGetNearbyStations(location: anyNamed('location'), radius: anyNamed('radius')))
          .thenAnswer((_) async => Right(tStations));
      
      // act
      final future = viewModel.loadStations();
      
      // assert - loading state
      expect(viewModel.state.isLoading, true);
      
      await future;
      
      // assert - success state
      expect(viewModel.state.isLoading, false);
      expect(viewModel.state.stations, tStations);
      expect(viewModel.state.error, null);
    });
    
    test('should update state to error when failure', () async {
      // arrange
      when(mockGetNearbyStations(location: anyNamed('location'), radius: anyNamed('radius')))
          .thenAnswer((_) async => Left(NetworkFailure('Network error')));
      
      // act
      await viewModel.loadStations();
      
      // assert
      expect(viewModel.state.isLoading, false);
      expect(viewModel.state.error, 'Network error');
      expect(viewModel.state.stations, isEmpty);
    });
  });
}
```

## Widget Test：測試 UI 元件

Widget Test 測試 UI 的行為，不需要真實裝置。

### 測試簡單 Widget

```dart
// test/presentation/widgets/station_card_test.dart
void main() {
  testWidgets('StationCard displays station info correctly', (tester) async {
    // arrange
    final station = ChargingStation(
      id: '1',
      name: 'Test Station',
      address: 'Test Address',
      availableChargers: 3,
      totalChargers: 5,
    );
    
    // act
    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: StationCard(station: station),
        ),
      ),
    );
    
    // assert
    expect(find.text('Test Station'), findsOneWidget);
    expect(find.text('Test Address'), findsOneWidget);
    expect(find.text('3 / 5'), findsOneWidget);
  });
  
  testWidgets('StationCard shows unavailable state', (tester) async {
    // arrange
    final station = ChargingStation(
      id: '1',
      name: 'Test Station',
      availableChargers: 0,
      totalChargers: 5,
    );
    
    // act
    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: StationCard(station: station),
        ),
      ),
    );
    
    // assert
    expect(find.text('無可用充電樁'), findsOneWidget);
    
    // 檢查顏色（不可用時變灰）
    final card = tester.widget<Card>(find.byType(Card));
    expect(card.color, Colors.grey[200]);
  });
}
```

### 測試互動

```dart
testWidgets('Tapping station card navigates to detail', (tester) async {
  // arrange
  final station = ChargingStation(id: '1', name: 'Test Station');
  bool navigated = false;
  
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: StationCard(
          station: station,
          onTap: () => navigated = true,
        ),
      ),
    ),
  );
  
  // act
  await tester.tap(find.byType(StationCard));
  await tester.pumpAndSettle();
  
  // assert
  expect(navigated, true);
});
```

### 測試 StatefulWidget

```dart
testWidgets('Favorite button toggles state', (tester) async {
  // arrange
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: FavoriteButton(stationId: '1'),
      ),
    ),
  );
  
  // 初始狀態：未收藏
  expect(find.byIcon(Icons.favorite_border), findsOneWidget);
  expect(find.byIcon(Icons.favorite), findsNothing);
  
  // act - 點擊收藏
  await tester.tap(find.byType(IconButton));
  await tester.pumpAndSettle();
  
  // assert - 已收藏
  expect(find.byIcon(Icons.favorite), findsOneWidget);
  expect(find.byIcon(Icons.favorite_border), findsNothing);
  
  // act - 再次點擊取消收藏
  await tester.tap(find.byType(IconButton));
  await tester.pumpAndSettle();
  
  // assert - 未收藏
  expect(find.byIcon(Icons.favorite_border), findsOneWidget);
});
```

### 測試 Riverpod Widget

用 `ProviderScope` 包裝：

```dart
testWidgets('MapScreen displays loading then stations', (tester) async {
  // arrange
  final container = ProviderContainer(
    overrides: [
      // Mock provider
      stationViewModelProvider.overrideWith((ref) {
        return MockMapViewModel();
      }),
    ],
  );
  
  await tester.pumpWidget(
    UncontrolledProviderScope(
      container: container,
      child: MaterialApp(
        home: MapScreen(),
      ),
    ),
  );
  
  // Loading state
  expect(find.byType(CircularProgressIndicator), findsOneWidget);
  
  // 觸發狀態更新
  await tester.pumpAndSettle();
  
  // Success state
  expect(find.byType(CircularProgressIndicator), findsNothing);
  expect(find.byType(StationCard), findsWidgets);
});
```

## Integration Test：測試完整流程

Integration Test 在真實裝置/模擬器執行，測試完整 User Flow。

### 設定

建立 `integration_test/` 目錄：

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_charging_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('Charging App E2E Test', () {
    testWidgets('Complete charging flow', (tester) async {
      // 啟動 App
      app.main();
      await tester.pumpAndSettle();
      
      // 等待地圖載入
      await tester.pumpAndSettle(Duration(seconds: 3));
      expect(find.byType(GoogleMap), findsOneWidget);
      
      // 點擊充電站 marker
      await tester.tap(find.byType(Marker).first);
      await tester.pumpAndSettle();
      
      // 驗證顯示充電站資訊
      expect(find.text('開始充電'), findsOneWidget);
      
      // 點擊開始充電
      await tester.tap(find.text('開始充電'));
      await tester.pumpAndSettle();
      
      // 選擇充電樁
      await tester.tap(find.text('A1'));
      await tester.pumpAndSettle();
      
      // 確認開始充電
      await tester.tap(find.text('確認'));
      await tester.pumpAndSettle(Duration(seconds: 2));
      
      // 驗證進入充電畫面
      expect(find.text('充電中'), findsOneWidget);
      expect(find.byType(ChargingProgressIndicator), findsOneWidget);
    });
  });
}
```

執行 Integration Test：

```bash
flutter test integration_test/app_test.dart
```

### 測試登入流程

```dart
testWidgets('User login flow', (tester) async {
  app.main();
  await tester.pumpAndSettle();
  
  // 點擊登入按鈕
  await tester.tap(find.text('登入'));
  await tester.pumpAndSettle();
  
  // 輸入帳號密碼
  await tester.enterText(
    find.byKey(Key('email_field')),
    'test@example.com',
  );
  await tester.enterText(
    find.byKey(Key('password_field')),
    'password123',
  );
  
  // 點擊登入
  await tester.tap(find.text('送出'));
  await tester.pumpAndSettle(Duration(seconds: 3));
  
  // 驗證登入成功
  expect(find.text('歡迎回來'), findsOneWidget);
});
```

## Golden Test：視覺回歸測試

Golden Test 比對 Widget 的視覺輸出，確保 UI 沒有非預期的變化。

```dart
testWidgets('StationCard golden test', (tester) async {
  final station = ChargingStation(
    id: '1',
    name: 'Test Station',
    availableChargers: 3,
    totalChargers: 5,
  );
  
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: StationCard(station: station),
      ),
    ),
  );
  
  await expectLater(
    find.byType(StationCard),
    matchesGoldenFile('goldens/station_card.png'),
  );
});
```

第一次執行會產生基準圖片，之後的測試會比對差異。

```bash
flutter test --update-goldens  # 更新基準圖片
flutter test                   # 比對測試
```

## 測試覆蓋率

檢查測試覆蓋率：

```bash
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html
```

目標：
- **Domain layer（Use Cases）**: 90%+
- **Data layer（Repository）**: 80%+
- **Presentation layer（ViewModel）**: 70%+

UI Widget 不用追求 100% 覆蓋率，測試關鍵邏輯即可。

## 實務心得

測試是個投資，前期花時間寫，後期省更多時間。

Unit Test 最划算，寫起來快、跑起來快，能抓到大部分 business logic 的問題。

Widget Test 要取捨，不是每個 Widget 都要測，測試有複雜互動邏輯的就好。

Integration Test 很慢，只測試關鍵 User Flow（登入、充電、支付等）。

Mock 工具很重要，`mockito` 讓我們可以隔離依賴，專注測試目標。

測試覆蓋率不是越高越好，要看投資報酬率。核心功能 90%，邊緣功能 60% 就夠了。

下週要來研究 Flutter 的效能分析工具，找出 App 的效能瓶頸。
