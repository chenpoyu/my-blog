---
layout: post
title: "iOS 定位服務與地圖整合"
date: 2018-05-18
categories: [iOS, Swift]
tags: [Swift, iOS, Core Location, MapKit, GPS]
---

這週開始研究定位服務。在 Java Web 開發中，定位通常是透過使用者的 IP 或瀏覽器的 Geolocation API，精確度有限。但在 iOS 上，可以直接存取 GPS、WiFi、基地台等多種定位來源，精確度可以達到幾公尺內。

## 定位服務的運作原理

iOS 的定位系統稱為 **Core Location**，它整合了多種定位技術：

1. **GPS**：最精確（5-10 公尺），但耗電也最高，需要戶外環境
2. **WiFi**：利用已知的 WiFi 熱點位置推算（20-50 公尺），室內可用
3. **基地台**：透過行動網路基地台定位（100-1000 公尺），訊號範圍大但精度低
4. **藍牙信標（iBeacon）**：室內精確定位（1-5 公尺），需要額外硬體

iOS 會根據**需求的精確度和電力狀況**自動選擇最合適的技術組合。這個智慧管理對開發者透明，我們只需要告訴系統「我需要多精確的位置」。

這個抽象層的設計很優雅，類似 Java 的 JDBC。你不需要知道底層是 MySQL 還是 PostgreSQL，只要透過統一介面操作。Core Location 也是如此，你不用管是用 GPS 還是 WiFi，系統會選擇最佳方案。

## 請求定位權限

和推送通知一樣，定位服務也需要**明確的使用者授權**。而且定位權限分得更細：

**使用期間授權（When In Use）**
- App 在前景時才能定位
- 適合導航、地圖搜尋等即時功能
- 使用者相對容易接受

**永遠授權（Always）**
- App 在背景也能持續定位
- 適合追蹤運動軌跡、地理圍欄等
- 使用者較謹慎，Apple 審查也更嚴格

### 設定 Info.plist

首先要在 `Info.plist` 中說明為什麼需要定位：

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>我們需要您的位置來顯示附近的餐廳</string>

<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>我們需要在背景追蹤您的運動軌跡</string>
```

這些描述會在權限請求彈窗中顯示。**描述要具體說明用途**，不要只寫「為了提供更好的服務」這種空話。

我看過很多 app 的權限描述寫得很模糊，使用者一律拒絕。清楚說明能提高授權率。

### 程式碼請求權限

```swift
import CoreLocation

class LocationManager: NSObject, CLLocationManagerDelegate {
    let locationManager = CLLocationManager()
    
    override init() {
        super.init()
        locationManager.delegate = self
    }
    
    func requestPermission() {
        // 檢查目前授權狀態
        let status = CLLocationManager.authorizationStatus()
        
        switch status {
        case .notDetermined:
            // 第一次請求，顯示權限對話框
            locationManager.requestWhenInUseAuthorization()
            
        case .authorizedWhenInUse:
            print("已授權使用期間定位")
            
        case .authorizedAlways:
            print("已授權永遠定位")
            
        case .denied, .restricted:
            print("使用者拒絕或受限制")
            // 引導使用者去設定中開啟
            showSettingsAlert()
            
        @unknown default:
            break
        }
    }
    
    // 授權狀態改變時呼叫
    func locationManager(
        _ manager: CLLocationManager,
        didChangeAuthorization status: CLAuthorizationStatus
    ) {
        print("授權狀態變更：\(status)")
    }
}
```

`authorizationStatus()` 方法讓你在請求前先檢查狀態。如果使用者已經拒絕，反覆請求不會有效果，要引導他們去「設定」手動開啟。

這和 Java 中的權限檢查類似，但 iOS 的權限模型更細緻，而且一旦拒絕就很難挽回。

## 開始接收位置更新

取得權限後，就能開始接收位置資料：

```swift
func startUpdatingLocation() {
    // 設定精確度需求
    locationManager.desiredAccuracy = kCLLocationAccuracyBest
    
    // 設定最小移動距離（公尺）才觸發更新
    locationManager.distanceFilter = 10
    
    // 開始更新
    locationManager.startUpdatingLocation()
}

func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations locations: [CLLocation]
) {
    guard let location = locations.last else { return }
    
    print("緯度：\(location.coordinate.latitude)")
    print("經度：\(location.coordinate.longitude)")
    print("精確度：\(location.horizontalAccuracy) 公尺")
    print("高度：\(location.altitude) 公尺")
    print("速度：\(location.speed) m/s")
    print("時間：\(location.timestamp)")
}

func locationManager(
    _ manager: CLLocationManager,
    didFailWithError error: Error
) {
    print("定位失敗：\(error.localizedDescription)")
}
```

### 精確度等級

`desiredAccuracy` 參數決定系統使用多少資源來定位：

- **kCLLocationAccuracyBestForNavigation**：最高精確度，用於轉彎導航
- **kCLLocationAccuracyBest**：GPS 級別（5-10m），高耗電
- **kCLLocationAccuracyNearestTenMeters**：10 公尺級別，適合大部分應用
- **kCLLocationAccuracyHundredMeters**：100 公尺級別，省電
- **kCLLocationAccuracyKilometer**：公里級別，只要知道大概區域

**原則**：只要求你真正需要的精確度。如果只是顯示城市天氣，用 `kCLLocationAccuracyKilometer` 就夠了，不要用 `Best` 浪費電力。

`distanceFilter` 也很重要，它設定「移動多少距離才通知」。如果設為 0，即使只移動 1 公分也會觸發更新，這在靜止時會產生很多無意義的回呼。

我在實作運動追蹤功能時，設定 `distanceFilter = 10`，也就是移動 10 公尺才記錄一次。這在跑步時很合理，又不會產生過多資料點。

## 單次定位請求

有時不需要持續追蹤，只要當下的位置：

```swift
func requestCurrentLocation() {
    locationManager.requestLocation()
}
```

這個方法會：
1. 啟動定位系統
2. 取得一次位置後自動停止
3. 呼叫 `didUpdateLocations` 或 `didFailWithError`

比起 `startUpdatingLocation()`，這個方法更省電，適合「顯示附近餐廳」這種一次性需求。

## 地理編碼：地址與座標轉換

有時我們有座標但需要顯示地址，或反過來。Core Location 提供 **Geocoding** 功能：

### 座標轉地址（Reverse Geocoding）

```swift
import CoreLocation

func getAddress(from location: CLLocation) {
    let geocoder = CLGeocoder()
    
    geocoder.reverseGeocodeLocation(location) { placemarks, error in
        if let error = error {
            print("地理編碼失敗：\(error)")
            return
        }
        
        if let placemark = placemarks?.first {
            // 取得詳細地址資訊
            let country = placemark.country ?? ""
            let city = placemark.locality ?? ""
            let street = placemark.thoroughfare ?? ""
            let number = placemark.subThoroughfare ?? ""
            
            let fullAddress = "\(country) \(city) \(street) \(number)"
            print("地址：\(fullAddress)")
            
            // 或用格式化後的地址
            if let postalAddress = placemark.postalAddress {
                print("格式化地址：\(postalAddress)")
            }
        }
    }
}
```

這個 API 會向 Apple 的伺服器查詢，所以需要網路連線。而且有**頻率限制**，不能在短時間內大量請求，否則會被節流。

### 地址轉座標（Forward Geocoding）

```swift
func getCoordinate(from address: String) {
    let geocoder = CLGeocoder()
    
    geocoder.geocodeAddressString(address) { placemarks, error in
        if let error = error {
            print("查詢失敗：\(error)")
            return
        }
        
        if let placemark = placemarks?.first,
           let location = placemark.location {
            print("座標：\(location.coordinate)")
        }
    }
}

// 使用範例
getCoordinate(from: "台北 101")
```

注意同一個地址可能對應多個結果（比如「中正路」全台灣有很多條）。API 回傳的是陣列，通常第一個是最相關的結果。

## MapKit：顯示地圖

定位資料通常要配合地圖顯示。iOS 提供 **MapKit** 框架：

```swift
import MapKit

class MapViewController: UIViewController {
    @IBOutlet weak var mapView: MKMapView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        mapView.showsUserLocation = true  // 顯示使用者位置的藍點
        mapView.delegate = self
        
        // 設定初始顯示區域
        let coordinate = CLLocationCoordinate2D(latitude: 25.033, longitude: 121.565)
        let region = MKCoordinateRegion(
            center: coordinate,
            latitudinalMeters: 1000,  // 南北範圍 1000 公尺
            longitudinalMeters: 1000  // 東西範圍 1000 公尺
        )
        mapView.setRegion(region, animated: true)
    }
}
```

### 在地圖上加入標註

```swift
func addAnnotation(title: String, coordinate: CLLocationCoordinate2D) {
    let annotation = MKPointAnnotation()
    annotation.title = title
    annotation.subtitle = "點擊查看詳情"
    annotation.coordinate = coordinate
    
    mapView.addAnnotation(annotation)
}

// 自訂標註外觀
extension MapViewController: MKMapViewDelegate {
    func mapView(
        _ mapView: MKMapView,
        viewFor annotation: MKAnnotation
    ) -> MKAnnotationView? {
        // 不要自訂使用者位置的標註
        if annotation is MKUserLocation {
            return nil
        }
        
        let identifier = "CustomPin"
        var annotationView = mapView.dequeueReusableAnnotationView(withIdentifier: identifier)
        
        if annotationView == nil {
            annotationView = MKMarkerAnnotationView(
                annotation: annotation,
                reuseIdentifier: identifier
            )
            annotationView?.canShowCallout = true  // 點擊時顯示氣泡
            
            // 在氣泡右側加入詳情按鈕
            let detailButton = UIButton(type: .detailDisclosure)
            annotationView?.rightCalloutAccessoryView = detailButton
        } else {
            annotationView?.annotation = annotation
        }
        
        // 自訂標註顏色
        if let markerView = annotationView as? MKMarkerAnnotationView {
            markerView.markerTintColor = .red
            markerView.glyphImage = UIImage(systemName: "star.fill")
        }
        
        return annotationView
    }
    
    func mapView(
        _ mapView: MKMapView,
        annotationView view: MKAnnotationView,
        calloutAccessoryControlTapped control: UIControl
    ) {
        // 使用者點擊了詳情按鈕
        if let annotation = view.annotation {
            print("查看詳情：\(annotation.title ?? "")")
        }
    }
}
```

MapKit 的標註系統和 UITableView 的 cell 重用機制類似，都用 `dequeueReusableAnnotationView` 來提升效能。如果地圖上有大量標註，重用機制能大幅減少記憶體使用。

## 路徑繪製與方向指引

MapKit 可以繪製路徑並計算導航路線：

```swift
func showRoute(from source: CLLocationCoordinate2D, to destination: CLLocationCoordinate2D) {
    let request = MKDirections.Request()
    request.source = MKMapItem(placemark: MKPlacemark(coordinate: source))
    request.destination = MKMapItem(placemark: MKPlacemark(coordinate: destination))
    request.transportType = .automobile  // 開車、walking 走路、transit 大眾運輸
    
    let directions = MKDirections(request: request)
    directions.calculate { response, error in
        if let error = error {
            print("路線計算失敗：\(error)")
            return
        }
        
        guard let route = response?.routes.first else { return }
        
        // 在地圖上繪製路線
        self.mapView.addOverlay(route.polyline, level: .aboveRoads)
        
        // 調整視野以顯示整條路線
        self.mapView.setVisibleMapRect(
            route.polyline.boundingMapRect,
            edgePadding: UIEdgeInsets(top: 50, left: 50, bottom: 50, right: 50),
            animated: true
        )
        
        // 顯示路線資訊
        let distance = route.distance / 1000  // 轉換為公里
        let time = route.expectedTravelTime / 60  // 轉換為分鐘
        print("距離：\(String(format: "%.1f", distance)) 公里")
        print("預估時間：\(Int(time)) 分鐘")
    }
}

// 繪製路線的外觀
func mapView(_ mapView: MKMapView, rendererFor overlay: MKOverlay) -> MKOverlayRenderer {
    if let polyline = overlay as? MKPolyline {
        let renderer = MKPolylineRenderer(polyline: polyline)
        renderer.strokeColor = .systemBlue
        renderer.lineWidth = 4
        return renderer
    }
    return MKOverlayRenderer()
}
```

這個 API 背後呼叫的是 Apple Maps 的路線規劃服務。它會考慮即時交通狀況、道路限制等因素，和我們在 Java 後端用 Google Maps API 類似。

## 地理圍欄（Geofencing）

地理圍欄是**進入或離開特定區域時觸發通知**的功能。這可以和推送通知結合，實作位置感知的提醒：

```swift
func setupGeofence(center: CLLocationCoordinate2D, radius: CLLocationDistance) {
    // 建立圓形區域
    let region = CLCircularRegion(
        center: center,
        radius: radius,  // 半徑（公尺）
        identifier: "office-area"
    )
    
    region.notifyOnEntry = true   // 進入時通知
    region.notifyOnExit = true    // 離開時通知
    
    // 開始監控
    locationManager.startMonitoring(for: region)
}

func locationManager(
    _ manager: CLLocationManager,
    didEnterRegion region: CLRegion
) {
    print("進入區域：\(region.identifier)")
    
    // 發送本地通知
    let content = UNMutableNotificationContent()
    content.title = "到達辦公室"
    content.body = "記得打卡"
    content.sound = .default
    
    let request = UNNotificationRequest(
        identifier: "geofence-enter",
        content: content,
        trigger: nil  // 立即觸發
    )
    
    UNUserNotificationCenter.current().add(request)
}

func locationManager(
    _ manager: CLLocationManager,
    didExitRegion region: CLRegion
) {
    print("離開區域：\(region.identifier)")
}
```

地理圍欄的優點是**省電**。系統會在硬體層面監控，不需要 app 持續運行。即使 app 被終止，圍欄事件還是會觸發。

但有限制：
- 每個 app 最多監控 **20 個區域**
- 半徑至少 **100 公尺**（更小的範圍用 iBeacon）
- 需要「永遠授權」才能在背景運作

這讓我想到 Java 的 WatchService，監控檔案系統變化。都是「設定一次，系統幫你監控」的模式，比輪詢高效太多。

## 實際案例：跑步追蹤 App

結合這週學的知識，實作一個簡單的跑步記錄功能：

```swift
class RunTracker: NSObject, CLLocationManagerDelegate {
    private let locationManager = CLLocationManager()
    private var trackPoints: [CLLocation] = []
    private var totalDistance: CLLocationDistance = 0
    private var startTime: Date?
    
    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.distanceFilter = 10  // 每 10 公尺記錄一次
        locationManager.activityType = .fitness  // 提示這是運動追蹤
    }
    
    func startTracking() {
        trackPoints.removeAll()
        totalDistance = 0
        startTime = Date()
        locationManager.startUpdatingLocation()
    }
    
    func stopTracking() {
        locationManager.stopUpdatingLocation()
    }
    
    func locationManager(
        _ manager: CLLocationManager,
        didUpdateLocations locations: [CLLocation]
    ) {
        guard let newLocation = locations.last else { return }
        
        // 過濾不準確的資料
        guard newLocation.horizontalAccuracy < 20 else {
            print("精確度太低，忽略")
            return
        }
        
        // 計算距離
        if let lastLocation = trackPoints.last {
            let distance = newLocation.distance(from: lastLocation)
            totalDistance += distance
        }
        
        trackPoints.append(newLocation)
        
        // 通知 UI 更新
        updateUI()
    }
    
    func updateUI() {
        let elapsed = Date().timeIntervalSince(startTime ?? Date())
        let speed = totalDistance / elapsed * 3.6  // 轉換為 km/h
        
        print("距離：\(String(format: "%.2f", totalDistance / 1000)) 公里")
        print("時間：\(Int(elapsed / 60)) 分鐘")
        print("速度：\(String(format: "%.1f", speed)) km/h")
    }
    
    func saveRoute() {
        // 將軌跡儲存到資料庫或檔案
        // 可以用 Core Data 或 JSON
    }
}
```

`activityType` 屬性告訴系統這是什麼類型的定位需求，系統會據此優化電力使用和定位演算法。

## 小結

這週研究定位服務的過程中，最大的感受是 **iOS 在隱私和電力管理上的用心**。從分級的權限請求、到彈性的精確度設定、再到地理圍欄的硬體加速，每個設計都在平衡功能和資源消耗。

關鍵要點：
1. **只要求必要的權限**：能用 WhenInUse 就不要求 Always
2. **選擇合適的精確度**：不要無腦用 Best
3. **適時停止更新**：不需要時立刻 `stopUpdatingLocation()`
4. **過濾無效資料**：檢查 `horizontalAccuracy` 避免誤差
5. **尊重使用者隱私**：清楚說明定位用途

下週計劃研究相機和相簿整合，包括拍照、錄影、圖片選擇、以及照片編輯。這是很多 app 的核心功能，應該會很有趣。
