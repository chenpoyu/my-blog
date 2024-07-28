---
layout: post
title: "Flutter Google Maps 整合：充電站地圖實作"
date: 2024-07-29 14:45:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Google Maps, 地理位置]
---

這週主要在處理 Google Maps 整合。充電站 App 的核心功能就是地圖，使用者要能看到附近的充電站、點擊查看詳情、導航過去。

Google Maps 在 Flutter 的整合還算順利,但也遇到一些坑。

## 套件選擇

Flutter 使用 Google Maps 主要有兩個官方套件：

**google_maps_flutter**：Google 官方維護，功能完整。

**flutter_map**：開源社群維護，使用 OpenStreetMap，免費但功能較少。

專案選用 **google_maps_flutter**，因為：
- 官方維護，穩定性高
- 功能完整（標記、路線、熱力圖）
- UI 跟原生 Google Maps 一致，使用者熟悉
- 專案有預算（Google Maps API 要收費）

## 設定 API Key

### Android 設定

1. 到 Google Cloud Console 建立專案
2. 啟用 Maps SDK for Android
3. 建立 API Key（記得限制使用範圍）
4. 在 `android/app/src/main/AndroidManifest.xml` 加入：

```xml
<manifest ...>
  <application ...>
    <meta-data
      android:name="com.google.android.geo.API_KEY"
      android:value="YOUR_API_KEY_HERE"/>
    ...
  </application>
</manifest>
```

### iOS 設定

1. 啟用 Maps SDK for iOS
2. 在 `ios/Runner/AppDelegate.swift` 加入：

```swift
import UIKit
import Flutter
import GoogleMaps

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    GMSServices.provideAPIKey("YOUR_API_KEY_HERE")
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

3. 更新 `ios/Podfile` 的最低版本：

```ruby
platform :ios, '12.0'
```

## 基本地圖顯示

```dart
// presentation/map/map_screen.dart
import 'package:google_maps_flutter/google_maps_flutter.dart';

class MapScreen extends ConsumerStatefulWidget {
  @override
  _MapScreenState createState() => _MapScreenState();
}

class _MapScreenState extends ConsumerState<MapScreen> {
  GoogleMapController? _mapController;
  
  static const _initialPosition = CameraPosition(
    target: LatLng(25.0330, 121.5654),  // 台北
    zoom: 14.0,
  );
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: GoogleMap(
        initialCameraPosition: _initialPosition,
        onMapCreated: (controller) {
          _mapController = controller;
        },
        myLocationEnabled: true,
        myLocationButtonEnabled: true,
        compassEnabled: true,
        mapToolbarEnabled: false,
      ),
    );
  }
}
```

## 取得使用者位置

使用 **geolocator** 套件：

```dart
import 'package:geolocator/geolocator.dart';

Future<Position?> _getCurrentLocation() async {
  // 檢查權限
  LocationPermission permission = await Geolocator.checkPermission();
  
  if (permission == LocationPermission.denied) {
    permission = await Geolocator.requestPermission();
    if (permission == LocationPermission.denied) {
      // 使用者拒絕
      return null;
    }
  }
  
  if (permission == LocationPermission.deniedForever) {
    // 永久拒絕，引導使用者去設定
    _showPermissionDialog();
    return null;
  }
  
  // 取得位置
  return await Geolocator.getCurrentPosition(
    desiredAccuracy: LocationAccuracy.high,
  );
}

@override
void initState() {
  super.initState();
  _initializeMap();
}

Future<void> _initializeMap() async {
  final position = await _getCurrentLocation();
  
  if (position != null) {
    final location = LatLng(position.latitude, position.longitude);
    
    // 移動地圖到使用者位置
    _mapController?.animateCamera(
      CameraUpdate.newCameraPosition(
        CameraPosition(target: location, zoom: 15.0),
      ),
    );
    
    // 載入附近充電站
    ref.read(mapViewModelProvider.notifier).loadNearbyStations(location);
  }
}
```

## 顯示充電站標記

```dart
class _MapScreenState extends ConsumerStatefulWidget {
  Set<Marker> _markers = {};
  
  @override
  Widget build(BuildContext context) {
    final state = ref.watch(mapViewModelProvider);
    
    // 當充電站資料更新，更新標記
    _updateMarkers(state.stations);
    
    return GoogleMap(
      // ...
      markers: _markers,
      onTap: _onMapTapped,
    );
  }
  
  void _updateMarkers(List<ChargingStation> stations) {
    _markers = stations.map((station) {
      return Marker(
        markerId: MarkerId(station.id),
        position: LatLng(station.location.latitude, station.location.longitude),
        icon: _getMarkerIcon(station),
        infoWindow: InfoWindow(
          title: station.name,
          snippet: '${station.availableChargers} 個充電樁可用',
          onTap: () => _onStationTapped(station),
        ),
      );
    }).toSet();
    
    setState(() {});
  }
  
  BitmapDescriptor _getMarkerIcon(ChargingStation station) {
    // 根據狀態顯示不同圖示
    if (station.availableChargers > 0) {
      return BitmapDescriptor.defaultMarkerWithHue(BitmapDescriptor.hueGreen);
    } else {
      return BitmapDescriptor.defaultMarkerWithHue(BitmapDescriptor.hueRed);
    }
  }
}
```

## 自訂 Marker 圖示

預設的 Marker 太陽春，要自訂：

```dart
Future<BitmapDescriptor> _createCustomMarker(ChargingStation station) async {
  // 方法 1: 使用圖片資源
  return await BitmapDescriptor.fromAssetImage(
    ImageConfiguration(size: Size(48, 48)),
    'assets/images/charger_available.png',
  );
  
  // 方法 2: 動態產生（推薦）
  final pictureRecorder = ui.PictureRecorder();
  final canvas = Canvas(pictureRecorder);
  
  // 畫一個圓形背景
  final paint = Paint()
    ..color = station.availableChargers > 0 ? Colors.green : Colors.red;
  canvas.drawCircle(Offset(24, 24), 24, paint);
  
  // 畫文字（可用充電樁數量）
  final textPainter = TextPainter(
    text: TextSpan(
      text: '${station.availableChargers}',
      style: TextStyle(color: Colors.white, fontSize: 16, fontWeight: FontWeight.bold),
    ),
    textDirection: TextDirection.ltr,
  );
  textPainter.layout();
  textPainter.paint(canvas, Offset(24 - textPainter.width / 2, 24 - textPainter.height / 2));
  
  // 轉成圖片
  final picture = pictureRecorder.endRecording();
  final image = await picture.toImage(48, 48);
  final bytes = await image.toByteData(format: ui.ImageByteFormat.png);
  
  return BitmapDescriptor.fromBytes(bytes!.buffer.asUint8List());
}

// 快取 Marker 避免重複產生
Map<String, BitmapDescriptor> _markerCache = {};

Future<BitmapDescriptor> _getMarkerIcon(ChargingStation station) async {
  final key = '${station.availableChargers > 0 ? "available" : "unavailable"}';
  
  if (_markerCache.containsKey(key)) {
    return _markerCache[key]!;
  }
  
  final icon = await _createCustomMarker(station);
  _markerCache[key] = icon;
  return icon;
}
```

## Marker 點擊事件

```dart
void _onStationTapped(ChargingStation station) {
  // 方法 1: 顯示 Bottom Sheet
  showModalBottomSheet(
    context: context,
    builder: (context) => StationDetailSheet(station: station),
  );
  
  // 方法 2: 導航到詳情頁
  // Navigator.push(
  //   context,
  //   MaterialPageRoute(
  //     builder: (context) => StationDetailScreen(station: station),
  //   ),
  // );
}
```

## 地圖樣式自訂

Google Maps 可以自訂顏色、隱藏元素等。

到 [Google Maps Styling Wizard](https://mapstyle.withgoogle.com/) 產生 JSON。

```dart
Future<void> _setMapStyle() async {
  final style = await rootBundle.loadString('assets/map_style.json');
  _mapController?.setMapStyle(style);
}

@override
void initState() {
  super.initState();
  _setMapStyle();
}
```

`assets/map_style.json`：

```json
[
  {
    "featureType": "poi",
    "elementType": "labels",
    "stylers": [{"visibility": "off"}]
  },
  {
    "featureType": "transit",
    "elementType": "labels",
    "stylers": [{"visibility": "off"}]
  }
]
```

這樣可以隱藏 POI（商店、餐廳）和大眾運輸標籤，讓地圖更簡潔。

## 叢集（Clustering）

如果充電站很多，Marker 會重疊。用叢集（Cluster）把鄰近的 Marker 合併。

Flutter 沒有官方的 Clustering，要用 **google_maps_cluster_manager**：

```dart
import 'package:google_maps_cluster_manager/google_maps_cluster_manager.dart';

class _MapScreenState extends ConsumerStatefulWidget {
  late ClusterManager _clusterManager;
  Set<Marker> _markers = {};
  
  @override
  void initState() {
    super.initState();
    
    _clusterManager = ClusterManager<ChargingStation>(
      [],  // 初始為空
      _updateMarkers,
      markerBuilder: _markerBuilder,
    );
  }
  
  Future<Marker> _markerBuilder(Cluster<ChargingStation> cluster) async {
    if (cluster.isMultiple) {
      // 多個充電站的叢集
      return Marker(
        markerId: MarkerId(cluster.getId()),
        position: cluster.location,
        icon: await _createClusterMarker(cluster.count),
      );
    } else {
      // 單一充電站
      final station = cluster.items.first;
      return Marker(
        markerId: MarkerId(station.id),
        position: LatLng(station.location.latitude, station.location.longitude),
        icon: await _getMarkerIcon(station),
        onTap: () => _onStationTapped(station),
      );
    }
  }
  
  Future<BitmapDescriptor> _createClusterMarker(int count) async {
    // 類似前面的自訂 Marker，但顯示數字
    // ...
  }
  
  void _updateMarkers(Set<Marker> markers) {
    setState(() {
      _markers = markers;
    });
  }
  
  void _onCameraMove(CameraPosition position) {
    _clusterManager.onCameraMove(position);
  }
  
  void _onCameraIdle() {
    _clusterManager.updateMap();
  }
  
  @override
  Widget build(BuildContext context) {
    return GoogleMap(
      // ...
      markers: _markers,
      onCameraMove: _onCameraMove,
      onCameraIdle: _onCameraIdle,
    );
  }
}
```

## 效能優化

地圖上有很多 Marker，效能要注意。

### 只顯示可見範圍的 Marker

```dart
void _onCameraMove(CameraPosition position) async {
  final bounds = await _mapController?.getVisibleRegion();
  
  if (bounds != null) {
    final visibleStations = _filterStationsInBounds(
      ref.read(mapViewModelProvider).stations,
      bounds,
    );
    
    _updateMarkers(visibleStations);
  }
}

List<ChargingStation> _filterStationsInBounds(
  List<ChargingStation> stations,
  LatLngBounds bounds,
) {
  return stations.where((station) {
    final lat = station.location.latitude;
    final lng = station.location.longitude;
    
    return lat >= bounds.southwest.latitude &&
           lat <= bounds.northeast.latitude &&
           lng >= bounds.southwest.longitude &&
           lng <= bounds.northeast.longitude;
  }).toList();
}
```

### Debounce 地圖移動

使用者拖動地圖時會觸發很多次 `onCameraMove`，要用 debounce：

```dart
Timer? _debounce;

void _onCameraMove(CameraPosition position) {
  if (_debounce?.isActive ?? false) _debounce!.cancel();
  
  _debounce = Timer(Duration(milliseconds: 500), () {
    _loadStationsForVisibleArea();
  });
}
```

### 減少 Marker 重繪

不要每次都重新建立 Marker，比對差異只更新變動的：

```dart
void _updateMarkers(List<ChargingStation> stations) {
  final newMarkers = <MarkerId, Marker>{};
  
  for (final station in stations) {
    final markerId = MarkerId(station.id);
    final existingMarker = _markers.firstWhere(
      (m) => m.markerId == markerId,
      orElse: () => null,
    );
    
    // 如果已存在且狀態沒變，重用
    if (existingMarker != null && 
        _isStationUnchanged(station, existingMarker)) {
      newMarkers[markerId] = existingMarker;
    } else {
      // 建立新 Marker
      newMarkers[markerId] = await _createMarker(station);
    }
  }
  
  setState(() {
    _markers = newMarkers.values.toSet();
  });
}
```

## 導航功能

點擊充電站後，提供導航功能（開啟外部地圖 App）。

```dart
Future<void> _openNavigation(ChargingStation station) async {
  final lat = station.location.latitude;
  final lng = station.location.longitude;
  
  // iOS: 優先開啟 Apple Maps
  if (Platform.isIOS) {
    final url = 'maps://?daddr=$lat,$lng&dirflg=d';
    if (await canLaunchUrl(Uri.parse(url))) {
      await launchUrl(Uri.parse(url));
      return;
    }
  }
  
  // Android 或 iOS fallback: Google Maps
  final googleUrl = 'https://www.google.com/maps/dir/?api=1&destination=$lat,$lng';
  if (await canLaunchUrl(Uri.parse(googleUrl))) {
    await launchUrl(Uri.parse(googleUrl), mode: LaunchMode.externalApplication);
  }
}
```

## 地圖主題切換

App 有深色模式，地圖也要跟著切換。

```dart
@override
Widget build(BuildContext context) {
  final isDarkMode = Theme.of(context).brightness == Brightness.dark;
  
  return GoogleMap(
    // ...
    onMapCreated: (controller) {
      _mapController = controller;
      _setMapStyle(isDarkMode);
    },
  );
}

Future<void> _setMapStyle(bool isDarkMode) async {
  final style = isDarkMode
      ? await rootBundle.loadString('assets/map_style_dark.json')
      : await rootBundle.loadString('assets/map_style_light.json');
  
  _mapController?.setMapStyle(style);
}

// 監聽主題變化
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  final isDarkMode = Theme.of(context).brightness == Brightness.dark;
  _setMapStyle(isDarkMode);
}
```

## 遇到的問題

**iOS 編譯失敗**：要確認 Podfile 設定正確，最低版本要 12.0 以上。有時候要 `pod install --repo-update`。

**Marker 圖示模糊**：要注意 DPI，用 `ImageConfiguration(devicePixelRatio: 2.0)` 或動態產生時用更高解析度。

**地圖延遲**：初始化地圖會稍微延遲（載入 Google Maps SDK），可以加 Loading indicator。

**記憶體洩漏**：要記得在 `dispose()` 清理 `_mapController`。

## 實務心得

Google Maps 整合後，App 的視覺效果好很多。看到充電站在地圖上分布，比列表直觀。

效能方面要多注意，尤其是 Marker 數量多的時候。Clustering 很重要，不然地圖會很卡。

自訂 Marker 可以讓 UI 更符合 App 風格，但要注意不要太複雜，會影響效能。

下週要來處理 Firebase 推播通知，這也是專案的重點功能。充電完成要通知使用者，還有各種促銷活動推播。
