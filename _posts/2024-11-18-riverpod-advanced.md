---
layout: post
title: "Riverpod 進階：狀態管理最佳實踐"
date: 2024-11-18 09:50:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Riverpod, State Management]
---

用 Riverpod 幾個月了，這週整理進階技巧：如何組織 Provider、處理複雜狀態、避免常見陷阱。

## Provider 的類型選擇

選對 Provider 類型很重要。

### Provider vs StateProvider

```dart
// Provider: 不會變的值
final apiBaseUrlProvider = Provider<String>((ref) {
  return 'https://api.example.com';
});

// StateProvider: 簡單狀態
final counterProvider = StateProvider<int>((ref) => 0);

// 使用
class CounterWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    
    return Column(
      children: [
        Text('Count: $count'),
        ElevatedButton(
          onPressed: () => ref.read(counterProvider.notifier).state++,
          child: Text('Increment'),
        ),
      ],
    );
  }
}
```

### StateNotifierProvider

複雜狀態用 StateNotifierProvider。

```dart
// 充電站列表狀態
class StationListState {
  final List<Station> stations;
  final bool isLoading;
  final String? error;
  final StationFilter filter;
  
  const StationListState({
    this.stations = const [],
    this.isLoading = false,
    this.error,
    this.filter = const StationFilter(),
  });
  
  StationListState copyWith({
    List<Station>? stations,
    bool? isLoading,
    String? error,
    StationFilter? filter,
  }) {
    return StationListState(
      stations: stations ?? this.stations,
      isLoading: isLoading ?? this.isLoading,
      error: error,
      filter: filter ?? this.filter,
    );
  }
}

// StateNotifier
class StationListNotifier extends StateNotifier<StationListState> {
  final StationRepository _repository;
  
  StationListNotifier(this._repository) 
      : super(const StationListState());
  
  Future<void> loadStations() async {
    state = state.copyWith(isLoading: true, error: null);
    
    final result = await _repository.getStations(state.filter);
    
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
  
  void updateFilter(StationFilter filter) {
    state = state.copyWith(filter: filter);
    loadStations();
  }
  
  void refreshStation(Station station) {
    final updatedStations = state.stations.map((s) {
      return s.id == station.id ? station : s;
    }).toList();
    
    state = state.copyWith(stations: updatedStations);
  }
}

// Provider
final stationListProvider = 
    StateNotifierProvider<StationListNotifier, StationListState>((ref) {
  final repository = ref.watch(stationRepositoryProvider);
  return StationListNotifier(repository);
});
```

### FutureProvider & StreamProvider

非同步資料用這兩個。

```dart
// FutureProvider: 一次性非同步資料
final userProfileProvider = FutureProvider.autoDispose<UserProfile>((ref) async {
  final repository = ref.watch(userRepositoryProvider);
  return repository.getUserProfile();
});

// 使用
class ProfileScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final profileAsync = ref.watch(userProfileProvider);
    
    return profileAsync.when(
      data: (profile) => ProfileView(profile: profile),
      loading: () => CircularProgressIndicator(),
      error: (error, stack) => ErrorView(error: error),
    );
  }
}

// StreamProvider: 持續更新的資料
final chargingStatusProvider = StreamProvider.autoDispose<ChargingStatus>((ref) {
  final service = ref.watch(chargingServiceProvider);
  return service.statusStream;
});
```

## Provider 依賴組合

善用 Provider 之間的依賴。

```dart
// 底層 Provider
final dioProvider = Provider<Dio>((ref) {
  final dio = Dio(BaseOptions(
    baseUrl: ref.watch(apiBaseUrlProvider),
  ));
  
  // 加入 Interceptor
  dio.interceptors.add(
    ref.watch(authInterceptorProvider),
  );
  
  return dio;
});

// Repository Provider 依賴 Dio
final stationRepositoryProvider = Provider<StationRepository>((ref) {
  final dio = ref.watch(dioProvider);
  return StationRepositoryImpl(dio);
});

// Notifier Provider 依賴 Repository
final stationListProvider = 
    StateNotifierProvider<StationListNotifier, StationListState>((ref) {
  final repository = ref.watch(stationRepositoryProvider);
  return StationListNotifier(repository);
});
```

## 避免不必要的 rebuild

### 使用 select

只監聽部分狀態。

```dart
class StationNameWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 只在 stations 改變時 rebuild，isLoading 改變不會觸發
    final stations = ref.watch(
      stationListProvider.select((state) => state.stations)
    );
    
    return ListView.builder(
      itemCount: stations.length,
      itemBuilder: (context, index) {
        return Text(stations[index].name);
      },
    );
  }
}
```

### Provider 分拆

把大的 Provider 拆成小的。

```dart
// 不好的做法：一個巨大的 AppState
class AppState {
  final User? user;
  final List<Station> stations;
  final List<ChargingHistory> history;
  final AppSettings settings;
  // 太多東西了...
}

// 好的做法：分開管理
final userProvider = StateNotifierProvider<UserNotifier, User?>(...);
final stationListProvider = StateNotifierProvider<StationListNotifier, StationListState>(...);
final historyProvider = StateNotifierProvider<HistoryNotifier, HistoryState>(...);
final settingsProvider = StateNotifierProvider<SettingsNotifier, AppSettings>(...);
```

## 處理副作用

用 `ref.listen` 處理副作用（如導航、顯示 SnackBar）。

```dart
class StationListScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 監聽錯誤並顯示 SnackBar
    ref.listen<StationListState>(
      stationListProvider,
      (previous, next) {
        if (next.error != null) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text(next.error!)),
          );
        }
      },
    );
    
    final state = ref.watch(stationListProvider);
    
    return Scaffold(
      body: state.isLoading
          ? Center(child: CircularProgressIndicator())
          : StationList(stations: state.stations),
    );
  }
}
```

## 測試 Provider

Riverpod 很好測試。

```dart
void main() {
  test('StationListNotifier loads stations', () async {
    // Mock Repository
    final mockRepository = MockStationRepository();
    when(mockRepository.getStations(any))
        .thenAnswer((_) async => Right([
          Station(id: '1', name: 'Station 1'),
          Station(id: '2', name: 'Station 2'),
        ]));
    
    // 建立 ProviderContainer
    final container = ProviderContainer(
      overrides: [
        stationRepositoryProvider.overrideWithValue(mockRepository),
      ],
    );
    
    // 取得 Notifier
    final notifier = container.read(stationListProvider.notifier);
    
    // 執行操作
    await notifier.loadStations();
    
    // 驗證結果
    final state = container.read(stationListProvider);
    expect(state.stations.length, 2);
    expect(state.isLoading, false);
    expect(state.error, null);
    
    container.dispose();
  });
}
```

## 家族 Provider (Family)

動態參數的 Provider。

```dart
// 依照 ID 取得充電站詳情
final stationDetailProvider = FutureProvider.family<Station, String>(
  (ref, stationId) async {
    final repository = ref.watch(stationRepositoryProvider);
    final result = await repository.getStationById(stationId);
    return result.fold(
      (failure) => throw failure,
      (station) => station,
    );
  },
);

// 使用
class StationDetailScreen extends ConsumerWidget {
  final String stationId;
  
  const StationDetailScreen({required this.stationId});
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final stationAsync = ref.watch(stationDetailProvider(stationId));
    
    return stationAsync.when(
      data: (station) => StationDetailView(station: station),
      loading: () => LoadingView(),
      error: (error, stack) => ErrorView(error: error),
    );
  }
}
```

## AutoDispose

不使用時自動清理。

```dart
// 使用 autoDispose 避免記憶體洩漏
final stationDetailProvider = FutureProvider.autoDispose.family<Station, String>(
  (ref, stationId) async {
    // 離開頁面時自動取消請求
    final cancelToken = CancelToken();
    ref.onDispose(() => cancelToken.cancel());
    
    final repository = ref.watch(stationRepositoryProvider);
    return repository.getStationById(stationId, cancelToken: cancelToken);
  },
);

// 保持 Provider 存活
ref.watch(stationDetailProvider(id).future).then((_) {
  // 保持 Provider，即使離開頁面
  ref.maintainState();
});
```

## 全域狀態 vs 局部狀態

不是所有狀態都要放 Provider。

```dart
// 全域狀態：用 Provider
final userProvider = StateNotifierProvider<UserNotifier, User?>(...);
final themeProvider = StateNotifierProvider<ThemeNotifier, ThemeMode>(...);

// 局部狀態：用 StatefulWidget
class StationFilterDialog extends StatefulWidget {
  @override
  _StationFilterDialogState createState() => _StationFilterDialogState();
}

class _StationFilterDialogState extends State<StationFilterDialog> {
  // 這些狀態只在 Dialog 內使用，不需要 Provider
  bool _showAvailableOnly = false;
  RangeValues _priceRange = RangeValues(0, 100);
  
  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      content: Column(
        children: [
          CheckboxListTile(
            title: Text('只顯示可用的'),
            value: _showAvailableOnly,
            onChanged: (value) {
              setState(() => _showAvailableOnly = value ?? false);
            },
          ),
          RangeSlider(
            values: _priceRange,
            min: 0,
            max: 100,
            onChanged: (values) {
              setState(() => _priceRange = values);
            },
          ),
        ],
      ),
    );
  }
}
```

## 實務心得

Riverpod 比 Provider 好用很多，compile-time safety 很棒。

`autoDispose` 預設加上去，需要保持狀態時才用 `maintainState`。

複雜狀態用 `freezed` 產生 `copyWith`，省很多時間。

Provider 不要巢狀太深，兩三層就夠了。

用 `ref.listen` 處理副作用，不要在 `build` 裡做。

測試時善用 `ProviderContainer` 和 `overrides`。

下週研究 Flutter 的效能監控和分析工具，以及如何找出效能瓶頸。
