---
layout: post
title: "Flutter 效能優化實戰技巧"
date: 2024-12-16 10:25:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Performance, Optimization]
---

App 效能直接影響使用者體驗。這週整理實戰中遇到的效能問題和解決方案。

## 渲染效能優化

### 減少 Widget rebuild

找出不必要的 rebuild。

```dart
// ❌ 不好的做法：整個頁面都會 rebuild
class StationListPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(stationListProvider);
    
    return Scaffold(
      appBar: AppBar(title: Text('充電站列表')),
      body: Column(
        children: [
          // 搜尋框
          SearchBar(onSearch: (keyword) => /* ... */),
          
          // 列表（isLoading 改變會讓整個頁面 rebuild）
          if (state.isLoading)
            CircularProgressIndicator()
          else
            StationList(stations: state.stations),
        ],
      ),
    );
  }
}

// ✅ 好的做法：分離狀態
class StationListPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('充電站列表')),
      body: Column(
        children: [
          SearchBar(onSearch: (keyword) => /* ... */),
          
          // 只有這部分會 rebuild
          const _LoadingOrList(),
        ],
      ),
    );
  }
}

class _LoadingOrList extends ConsumerWidget {
  const _LoadingOrList();
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 只監聽需要的狀態
    final isLoading = ref.watch(
      stationListProvider.select((s) => s.isLoading)
    );
    final stations = ref.watch(
      stationListProvider.select((s) => s.stations)
    );
    
    if (isLoading) return CircularProgressIndicator();
    return StationList(stations: stations);
  }
}
```

### const Constructor

盡量使用 const Widget。

```dart
// ❌ 每次 rebuild 都建立新的 Widget
return Column(
  children: [
    Icon(Icons.ev_station, size: 24),
    SizedBox(height: 8),
    Text('充電站'),
  ],
);

// ✅ 使用 const，Flutter 會重用
return Column(
  children: [
    const Icon(Icons.ev_station, size: 24),
    const SizedBox(height: 8),
    const Text('充電站'),
  ],
);
```

### RepaintBoundary

隔離動畫，避免影響其他 Widget。

```dart
class ChargingAnimation extends StatefulWidget {
  @override
  _ChargingAnimationState createState() => _ChargingAnimationState();
}

class _ChargingAnimationState extends State<ChargingAnimation>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: Duration(seconds: 2),
      vsync: this,
    )..repeat();
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 動畫包在 RepaintBoundary 裡
        RepaintBoundary(
          child: AnimatedBuilder(
            animation: _controller,
            builder: (context, child) {
              return Transform.rotate(
                angle: _controller.value * 2 * 3.14159,
                child: Icon(Icons.charging_station),
              );
            },
          ),
        ),
        
        // 這些 Widget 不會因為動畫而 repaint
        const SizedBox(height: 16),
        const Text('充電中...'),
      ],
    );
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

## 列表效能

### ListView.builder vs ListView

```dart
// ❌ 不好：一次建立所有 Widget
ListView(
  children: stations.map((station) => StationCard(station)).toList(),
);

// ✅ 好：只建立可見的 Widget
ListView.builder(
  itemCount: stations.length,
  itemBuilder: (context, index) {
    return StationCard(stations[index]);
  },
);

// ✅ 更好：加上 itemExtent（如果高度固定）
ListView.builder(
  itemCount: stations.length,
  itemExtent: 100, // 固定高度，效能更好
  itemBuilder: (context, index) {
    return StationCard(stations[index]);
  },
);
```

### 複雜列表項目優化

```dart
// ❌ 不好：每個項目都重新建立
ListView.builder(
  itemCount: stations.length,
  itemBuilder: (context, index) {
    final station = stations[index];
    return Card(
      child: ListTile(
        leading: CachedNetworkImage(imageUrl: station.imageUrl),
        title: Text(station.name),
        subtitle: Column(
          children: [
            Text(station.address),
            Row(
              children: [
                Icon(Icons.star, color: Colors.yellow),
                Text('${station.rating}'),
              ],
            ),
          ],
        ),
      ),
    );
  },
);

// ✅ 好：抽出獨立 Widget，加上 const
ListView.builder(
  itemCount: stations.length,
  itemBuilder: (context, index) {
    return StationCard(station: stations[index]);
  },
);

class StationCard extends StatelessWidget {
  final Station station;
  
  const StationCard({required this.station});
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: ListTile(
        leading: _StationImage(url: station.imageUrl),
        title: Text(station.name),
        subtitle: _StationInfo(station: station),
      ),
    );
  }
}

class _StationImage extends StatelessWidget {
  final String url;
  
  const _StationImage({required this.url});
  
  @override
  Widget build(BuildContext context) {
    return CachedNetworkImage(
      imageUrl: url,
      width: 50,
      height: 50,
      fit: BoxFit.cover,
      placeholder: (context, url) => const CircularProgressIndicator(),
    );
  }
}
```

### AutomaticKeepAliveClientMixin

保持 Tab 狀態，避免重複載入。

```dart
class StationListTab extends StatefulWidget {
  @override
  _StationListTabState createState() => _StationListTabState();
}

class _StationListTabState extends State<StationListTab>
    with AutomaticKeepAliveClientMixin {
  
  @override
  bool get wantKeepAlive => true; // 保持狀態
  
  @override
  Widget build(BuildContext context) {
    super.build(context); // 必須呼叫
    
    return ListView.builder(/* ... */);
  }
}
```

## 圖片優化

### CachedNetworkImage

```dart
// ✅ 使用快取
CachedNetworkImage(
  imageUrl: station.imageUrl,
  width: 100,
  height: 100,
  fit: BoxFit.cover,
  placeholder: (context, url) => Container(
    width: 100,
    height: 100,
    color: Colors.grey[300],
    child: const Center(child: CircularProgressIndicator()),
  ),
  errorWidget: (context, url, error) => Container(
    width: 100,
    height: 100,
    color: Colors.grey[300],
    child: const Icon(Icons.error),
  ),
  // 設定快取
  cacheManager: CacheManager(
    Config(
      'customCacheKey',
      stalePeriod: Duration(days: 7), // 快取 7 天
      maxNrOfCacheObjects: 100,       // 最多 100 張
    ),
  ),
);
```

### 圖片尺寸

不要載入過大的圖片。

```dart
// ❌ 不好：載入原始大圖（可能 2000x2000）
Image.network('https://api.example.com/station/image.jpg');

// ✅ 好：後端提供縮圖 API
Image.network('https://api.example.com/station/image.jpg?size=200x200');

// 或者在 Flutter 端 resize
CachedNetworkImage(
  imageUrl: station.imageUrl,
  memCacheWidth: 200, // 記憶體快取寬度
  memCacheHeight: 200,
);
```

## 資料處理優化

### compute() 處理大量資料

```dart
// ❌ 不好：在 Main Isolate 處理，會卡 UI
Future<List<Station>> parseStations(String jsonString) async {
  final jsonData = json.decode(jsonString);
  return (jsonData as List)
      .map((e) => Station.fromJson(e))
      .toList();
}

// ✅ 好：用 compute 在背景執行
Future<List<Station>> parseStations(String jsonString) async {
  return await compute(_parseStationsBackground, jsonString);
}

List<Station> _parseStationsBackground(String jsonString) {
  final jsonData = json.decode(jsonString);
  return (jsonData as List)
      .map((e) => Station.fromJson(e))
      .toList();
}
```

### 避免在 build 裡做昂貴運算

```dart
// ❌ 不好
@override
Widget build(BuildContext context) {
  // 每次 rebuild 都重算
  final sortedStations = stations
      .where((s) => s.isAvailable)
      .toList()
      ..sort((a, b) => a.distance.compareTo(b.distance));
  
  return ListView.builder(
    itemCount: sortedStations.length,
    itemBuilder: (context, index) => StationCard(sortedStations[index]),
  );
}

// ✅ 好：在 State 或 Provider 裡處理
class StationListState {
  final List<Station> stations;
  
  // Computed property
  List<Station> get availableStations {
    return stations.where((s) => s.isAvailable).toList()
      ..sort((a, b) => a.distance.compareTo(b.distance));
  }
}
```

## 地圖效能

### Marker Clustering

太多 Marker 會很慢。

```dart
// 使用 google_maps_cluster_manager
ClusterManager<Station>(
  stations,
  _updateMarkers,
  markerBuilder: _markerBuilder,
  levels: [1, 4.25, 6.75, 8.25, 11.5, 14.5, 16.0, 16.5, 20.0],
  extraPercent: 0.2,
);

void _updateMarkers(Set<Marker> markers) {
  setState(() {
    _markers = markers;
  });
}

Future<Marker> _markerBuilder(Cluster<Station> cluster) async {
  return Marker(
    markerId: MarkerId(cluster.getId()),
    position: cluster.location,
    icon: await _getMarkerIcon(cluster),
  );
}

// 自訂 Cluster Icon
Future<BitmapDescriptor> _getMarkerIcon(Cluster<Station> cluster) async {
  if (cluster.isMultiple) {
    // 多個站點：顯示數字
    return await _createNumberIcon(cluster.count);
  } else {
    // 單一站點：顯示圖示
    return BitmapDescriptor.defaultMarkerWithHue(
      cluster.items.first.isAvailable
          ? BitmapDescriptor.hueGreen
          : BitmapDescriptor.hueRed,
    );
  }
}
```

### 限制可見區域的 Marker

```dart
GoogleMap(
  onCameraMove: (position) {
    final bounds = position.visibleRegion;
    
    // 只顯示可見範圍內的 Marker
    final visibleStations = allStations.where((station) {
      return _isInBounds(station.location, bounds);
    }).toList();
    
    _updateMarkers(visibleStations);
  },
);

bool _isInBounds(LatLng location, LatLngBounds bounds) {
  return location.latitude >= bounds.southwest.latitude &&
         location.latitude <= bounds.northeast.latitude &&
         location.longitude >= bounds.southwest.longitude &&
         location.longitude <= bounds.northeast.longitude;
}
```

## 記憶體優化

### 避免記憶體洩漏

```dart
class _StationDetailState extends State<StationDetail> {
  StreamSubscription? _subscription;
  Timer? _timer;
  
  @override
  void initState() {
    super.initState();
    
    // 訂閱 Stream
    _subscription = chargingStatusStream.listen((status) {
      if (mounted) setState(() => _status = status);
    });
    
    // 定時更新
    _timer = Timer.periodic(Duration(seconds: 5), (_) {
      if (mounted) _refreshData();
    });
  }
  
  @override
  void dispose() {
    // ✅ 記得取消訂閱
    _subscription?.cancel();
    _timer?.cancel();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return /* ... */;
  }
}
```

### 圖片快取限制

```dart
// 限制記憶體快取大小
void main() {
  PaintingBinding.instance.imageCache.maximumSize = 100;      // 最多 100 張
  PaintingBinding.instance.imageCache.maximumSizeBytes = 50 << 20; // 50MB
  
  runApp(MyApp());
}
```

## 效能監控

### 開發階段檢查

```dart
void main() {
  if (kDebugMode) {
    // 檢查 Widget rebuild
    debugPrintRebuildDirtyWidgets = true;
    
    // 檢查 Layout 問題
    debugPrintMarkNeedsLayoutStacks = true;
  }
  
  runApp(MyApp());
}
```

### Profile Mode 測試

```bash
# 永遠在 Profile Mode 測試效能，不要用 Debug Mode
flutter run --profile

# 或者直接 build
flutter build apk --profile
```

## 實務心得

效能優化要用工具量測，不要憑感覺（DevTools 很好用）。

先優化明顯的大問題，不要過度優化小細節。

const Widget 能用就用，免費的效能提升。

複雜列表用 `ListView.builder` + `AutomaticKeepAliveClientMixin`。

圖片是效能殺手，一定要快取和調整尺寸。

用 `compute()` 處理大量資料，避免卡 UI。

定期用 Profile Mode 在實機測試，模擬器不準。

下週來做個 Flutter 學習的年終總結，回顧這一年的成長。
