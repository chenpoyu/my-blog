---
layout: post
title: "Flutter 效能分析與優化：找出 App 的效能瓶頸"
date: 2024-09-02 15:50:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Performance, Optimization, DevTools]
---

上週補完測試，這週來處理效能問題。測試階段發現地圖載入大量 marker 時會卡頓，充電歷史列表滑動也不夠順暢，是時候用 Flutter DevTools 深入分析了。

## Flutter DevTools

Flutter DevTools 是官方的效能分析工具，功能很強大。

### 啟動 DevTools

```bash
# 在 debug 模式執行 App
flutter run

# 另開終端啟動 DevTools
flutter pub global activate devtools
flutter pub global run devtools
```

或直接在 VS Code / Android Studio 點擊 DevTools 圖示。

### Performance View

Performance View 顯示每一幀的渲染時間。

**目標**: 60 FPS = 16.67ms/frame（綠色區域）

如果超過 16.67ms（紅色區域），就會掉幀。

在 DevTools 點擊 "Record" 開始錄製，操作 App，然後停止。可以看到每一幀的 Timeline。

### Timeline Event

Timeline 顯示每一幀的事件：

- **Build**: Widget build 時間
- **Layout**: Layout 計算時間
- **Paint**: 繪製時間
- **Rasterize**: 光柵化時間

點擊某一幀可以看到詳細的 Event Tree，找出哪個 Widget 耗時最多。

## 常見效能問題

### 問題 1：過度 Rebuild

Widget 不必要的 rebuild 是最常見的效能問題。

**檢查方法**：在 `build()` 加 print，看是否過度觸發。

```dart
@override
Widget build(BuildContext context) {
  print('MapScreen rebuild'); // Debug
  return Scaffold(/* ... */);
}
```

**解決方案**：

1. 用 `const` constructor

```dart
// ❌ 每次都 rebuild
Text('Title')

// ✅ const widget 不會 rebuild
const Text('Title')
```

2. 拆分 Widget

```dart
// ❌ 整個 Screen rebuild
class MapScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final stations = ref.watch(stationProvider);
    
    return Scaffold(
      appBar: AppBar(title: Text('地圖')),
      body: Column(
        children: [
          // 搜尋框變化會導致整個 Screen rebuild
          SearchBar(/* ... */),
          GoogleMap(/* 很耗時 */),
          StationList(stations: stations),
        ],
      ),
    );
  }
}

// ✅ 拆分獨立 Widget
class MapScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('地圖')),
      body: Column(
        children: [
          const SearchSection(),  // 獨立 Widget
          const MapSection(),     // 獨立 Widget
          const StationListSection(), // 獨立 Widget
        ],
      ),
    );
  }
}
```

3. 用 `Consumer` 精準監聽

```dart
// ❌ 整個 Widget rebuild
class StationCard extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(stationViewModelProvider);
    
    return Card(
      child: Column(
        children: [
          // state 變化會 rebuild 整個 Card
          Text(state.station.name),
          Text(state.station.address),
          ElevatedButton(/* ... */),
        ],
      ),
    );
  }
}

// ✅ 只 rebuild 需要的部分
class StationCard extends StatelessWidget {
  final Station station;
  
  const StationCard({required this.station});
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          Text(station.name),
          Text(station.address),
          Consumer(
            builder: (context, ref, child) {
              final isFavorite = ref.watch(
                favoriteProvider.select((s) => s.isFavorite(station.id)),
              );
              return FavoriteButton(isFavorite: isFavorite);
            },
          ),
        ],
      ),
    );
  }
}
```

### 問題 2：ListView 效能

大量資料的 ListView 要用 `builder`。

```dart
// ❌ 一次建立所有 Widget
ListView(
  children: stations.map((s) => StationCard(station: s)).toList(),
)

// ✅ 只建立可見的 Widget
ListView.builder(
  itemCount: stations.length,
  itemBuilder: (context, index) {
    return StationCard(station: stations[index]);
  },
)
```

如果 item 高度固定，加上 `itemExtent`：

```dart
ListView.builder(
  itemCount: stations.length,
  itemExtent: 120.0,  // 固定高度，提升效能
  itemBuilder: (context, index) {
    return StationCard(station: stations[index]);
  },
)
```

### 問題 3：圖片載入

圖片是效能殺手，要優化。

1. **快取**

```dart
CachedNetworkImage(
  imageUrl: station.imageUrl,
  placeholder: (context, url) => CircularProgressIndicator(),
  memCacheWidth: 400,  // 限制記憶體快取大小
)
```

2. **縮圖**

後端提供縮圖 URL，不要載原圖。

```dart
// ❌ 載入 2MB 原圖顯示 100x100
Image.network('https://api.example.com/images/station1.jpg')

// ✅ 載入 50KB 縮圖
Image.network('https://api.example.com/images/station1_thumb.jpg')
```

3. **預載**

提前載入即將顯示的圖片：

```dart
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  precacheImage(
    NetworkImage(station.imageUrl),
    context,
  );
}
```

### 問題 4：動畫效能

複雜動畫要小心。

1. **用 `RepaintBoundary`**

隔離需要重繪的區域：

```dart
RepaintBoundary(
  child: AnimatedWidget(/* 頻繁重繪的動畫 */),
)
```

2. **避免 `setState()` 在動畫中**

用 `AnimatedBuilder` 或 `TweenAnimationBuilder`：

```dart
// ❌ 每一幀都 setState，整個 Widget rebuild
class _ChargingProgressState extends State<ChargingProgress> {
  late AnimationController _controller;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(/* ... */);
    _controller.addListener(() {
      setState(() {}); // 整個 Widget rebuild！
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        CircularProgressIndicator(value: _controller.value),
        Text('充電中: ${(_controller.value * 100).toInt()}%'),
        StationInfo(/* 不需要 rebuild 的部分也被 rebuild */),
      ],
    );
  }
}

// ✅ 用 AnimatedBuilder 只 rebuild 需要的部分
class ChargingProgress extends StatefulWidget {
  @override
  _ChargingProgressState createState() => _ChargingProgressState();
}

class _ChargingProgressState extends State<ChargingProgress>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(/* ... */);
    // 不需要 addListener
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        AnimatedBuilder(
          animation: _controller,
          builder: (context, child) {
            return CircularProgressIndicator(value: _controller.value);
          },
        ),
        AnimatedBuilder(
          animation: _controller,
          builder: (context, child) {
            return Text('充電中: ${(_controller.value * 100).toInt()}%');
          },
        ),
        const StationInfo(), // 不會 rebuild
      ],
    );
  }
}
```

### 問題 5：過度使用 `Opacity`

`Opacity` 很耗效能，能避免就避免。

```dart
// ❌ Opacity 需要 offscreen buffer
Opacity(
  opacity: isVisible ? 1.0 : 0.0,
  child: ExpensiveWidget(),
)

// ✅ 用 Visibility 或 AnimatedOpacity
AnimatedOpacity(
  opacity: isVisible ? 1.0 : 0.0,
  duration: Duration(milliseconds: 300),
  child: ExpensiveWidget(),
)

// ✅✅ 如果完全隱藏，直接不顯示
if (isVisible) ExpensiveWidget()
```

## 記憶體分析

### Memory View

DevTools 的 Memory View 顯示記憶體使用量。

點擊 "Profile Memory" 可以看到：
- **Dart Heap**: Dart 物件佔用的記憶體
- **External**: Native 資源（圖片、檔案等）
- **RSS**: 實際物件記憶體

### 記憶體洩漏檢查

記憶體洩漏是指物件不再使用但沒有被釋放。

**常見原因**：
- Timer/StreamSubscription 沒有取消
- AnimationController 沒有 dispose
- ChangeNotifier 沒有 dispose
- Static 變數持有大物件

**檢查方法**：

1. 在 DevTools 點擊 "GC" (Garbage Collection)
2. 操作 App（例如進入頁面再離開）
3. 再點擊 "GC"
4. 檢查記憶體是否回到原本水平

如果記憶體持續增長，可能有洩漏。

**解決範例**：

```dart
// ❌ 沒有取消 subscription
class ChargingScreen extends ConsumerStatefulWidget {
  @override
  _ChargingScreenState createState() => _ChargingScreenState();
}

class _ChargingScreenState extends ConsumerState<ChargingScreen> {
  late StreamSubscription _subscription;
  
  @override
  void initState() {
    super.initState();
    _subscription = chargingStream.listen((event) {
      // 處理充電事件
    });
  }
  
  // 沒有 dispose！
}

// ✅ 正確 dispose
class _ChargingScreenState extends ConsumerState<ChargingScreen> {
  late StreamSubscription _subscription;
  
  @override
  void initState() {
    super.initState();
    _subscription = chargingStream.listen((event) {
      // 處理充電事件
    });
  }
  
  @override
  void dispose() {
    _subscription.cancel(); // 取消訂閱
    super.dispose();
  }
}
```

## App Size 分析

App 安裝包大小也很重要。

### 檢查 APK/IPA 大小

```bash
flutter build apk --analyze-size
flutter build ios --analyze-size
```

會產生大小分析報告，顯示哪些資源最大。

### 優化建議

1. **圖片壓縮**

用 WebP 或壓縮 PNG/JPEG。

2. **移除未使用的資源**

```bash
flutter pub run flutter_asset_lint
```

3. **Code Splitting**

用 Deferred Loading 延遲載入不常用的功能。

4. **移除 debug 資訊**

Release 模式自動移除，但要確認：

```bash
flutter build apk --release
```

## 實戰：地圖 Marker 優化

充電站地圖有上千個 marker，全部顯示會很慢。

### 問題

```dart
// ❌ 顯示所有 marker
GoogleMap(
  markers: stations.map((s) => Marker(
    markerId: MarkerId(s.id),
    position: s.location,
    icon: customIcon,
  )).toSet(),
)
```

1000 個 marker = 卡頓。

### 優化 1：只顯示可見範圍

```dart
class MapViewModel extends StateNotifier<MapState> {
  void onCameraMove(CameraPosition position) {
    // 計算可見範圍
    final bounds = _getVisibleBounds(position);
    
    // 只保留可見範圍內的充電站
    final visibleStations = _allStations.where((station) {
      return bounds.contains(station.location);
    }).toList();
    
    state = state.copyWith(visibleStations: visibleStations);
  }
  
  LatLngBounds _getVisibleBounds(CameraPosition position) {
    // 根據 zoom level 計算範圍
    final zoom = position.zoom;
    final center = position.target;
    
    // 簡化計算（實際更複雜）
    final latDelta = 180.0 / pow(2, zoom);
    final lngDelta = 360.0 / pow(2, zoom);
    
    return LatLngBounds(
      southwest: LatLng(center.latitude - latDelta, center.longitude - lngDelta),
      northeast: LatLng(center.latitude + latDelta, center.longitude + lngDelta),
    );
  }
}
```

### 優化 2：Marker 聚合

太多 marker 時，用 Clustering 聚合。

```dart
import 'package:google_maps_cluster_manager/google_maps_cluster_manager.dart';

class MapWithClustering extends StatefulWidget {
  @override
  _MapWithClusteringState createState() => _MapWithClusteringState();
}

class _MapWithClusteringState extends State<MapWithClustering> {
  late ClusterManager _clusterManager;
  Set<Marker> _markers = {};
  
  @override
  void initState() {
    super.initState();
    _clusterManager = ClusterManager(
      stations.map((s) => ClusterItem(s.location, item: s)).toList(),
      _updateMarkers,
      markerBuilder: _markerBuilder,
    );
  }
  
  void _updateMarkers(Set<Marker> markers) {
    setState(() {
      _markers = markers;
    });
  }
  
  Future<Marker> _markerBuilder(Cluster cluster) async {
    return Marker(
      markerId: MarkerId(cluster.getId()),
      position: cluster.location,
      icon: await _getClusterIcon(cluster.count),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return GoogleMap(
      markers: _markers,
      onCameraMove: _clusterManager.onCameraMove,
      onCameraIdle: _clusterManager.updateMap,
    );
  }
}
```

### 優化 3：Debounce Camera Move

減少地圖移動時的更新頻率。

```dart
Timer? _debounceTimer;

void onCameraMove(CameraPosition position) {
  _debounceTimer?.cancel();
  
  _debounceTimer = Timer(Duration(milliseconds: 300), () {
    _updateVisibleStations(position);
  });
}
```

經過這些優化，地圖從卡頓變流暢，FPS 從 30 提升到 55+。

## 效能檢查清單

開發時的檢查清單：

- [ ] 用 `const` constructor
- [ ] 大列表用 `builder`
- [ ] 圖片加快取和壓縮
- [ ] 動畫用 `AnimatedBuilder`
- [ ] 避免過度 `setState()`
- [ ] 拆分大 Widget
- [ ] `dispose()` 清理資源
- [ ] 用 DevTools 檢查效能

## 實務心得

效能優化要**先測量，再優化**。不要憑感覺，用 DevTools 找出真正的瓶頸。

過早優化是浪費時間，先讓功能正確，再來優化效能。

最有效的優化往往是**減少不必要的工作**：不顯示的不要建立、不變的不要重算、不用的不要保留。

Flutter 本身已經很快，大部分效能問題來自**不當使用**：過度 rebuild、大圖片、複雜動畫。

記得在真實裝置測試，模擬器效能不準確。

下週來研究 Flutter 的持續整合與部署，把測試和建置流程自動化。
