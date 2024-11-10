---
layout: post
title: "Flutter 與 Native 整合：Platform Channels"
date: 2024-11-11 15:15:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Native, Platform Channels, iOS, Android]
---

有些功能 Flutter 原生不支援，需要呼叫 Native 程式碼。這週研究 Platform Channels，實作藍牙、感應器等原生功能。

## Platform Channels 基礎

Flutter 與 Native 透過 Platform Channels 溝通。

### MethodChannel

最常用的方式，呼叫 Native 方法並取得回傳值。

```dart
// Flutter 端
class BatteryService {
  static const platform = MethodChannel('com.example.app/battery');
  
  Future<int?> getBatteryLevel() async {
    try {
      final int result = await platform.invokeMethod('getBatteryLevel');
      return result;
    } on PlatformException catch (e) {
      print("Failed to get battery level: '${e.message}'");
      return null;
    }
  }
}
```

### Android 實作

```kotlin
// android/app/src/main/kotlin/.../MainActivity.kt
import android.content.Context
import android.content.ContextWrapper
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity: FlutterActivity() {
    private val CHANNEL = "com.example.app/battery"
    
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        
        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            CHANNEL
        ).setMethodCallHandler { call, result ->
            when (call.method) {
                "getBatteryLevel" -> {
                    val batteryLevel = getBatteryLevel()
                    if (batteryLevel != -1) {
                        result.success(batteryLevel)
                    } else {
                        result.error("UNAVAILABLE", "Battery level not available", null)
                    }
                }
                else -> result.notImplemented()
            }
        }
    }
    
    private fun getBatteryLevel(): Int {
        val batteryManager = getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        return batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
    }
}
```

### iOS 實作

```swift
// ios/Runner/AppDelegate.swift
import UIKit
import Flutter

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        let controller = window?.rootViewController as! FlutterViewController
        let batteryChannel = FlutterMethodChannel(
            name: "com.example.app/battery",
            binaryMessenger: controller.binaryMessenger
        )
        
        batteryChannel.setMethodCallHandler { [weak self] (call, result) in
            guard call.method == "getBatteryLevel" else {
                result(FlutterMethodNotImplemented)
                return
            }
            self?.receiveBatteryLevel(result: result)
        }
        
        GeneratedPluginRegistrant.register(with: self)
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
    
    private func receiveBatteryLevel(result: FlutterResult) {
        let device = UIDevice.current
        device.isBatteryMonitoringEnabled = true
        
        if device.batteryState == .unknown {
            result(FlutterError(
                code: "UNAVAILABLE",
                message: "Battery level not available",
                details: nil
            ))
        } else {
            let batteryLevel = Int(device.batteryLevel * 100)
            result(batteryLevel)
        }
    }
}
```

## EventChannel

持續接收 Native 事件流。

### 充電狀態監聽

```dart
// Flutter 端
class ChargingStatusService {
  static const eventChannel = EventChannel('com.example.app/charging');
  
  Stream<bool> get chargingStream {
    return eventChannel.receiveBroadcastStream().map((event) => event as bool);
  }
}

// 使用
class ChargingWidget extends StatefulWidget {
  @override
  _ChargingWidgetState createState() => _ChargingWidgetState();
}

class _ChargingWidgetState extends State<ChargingWidget> {
  bool _isCharging = false;
  late StreamSubscription _subscription;
  
  @override
  void initState() {
    super.initState();
    _subscription = ChargingStatusService()
        .chargingStream
        .listen((isCharging) {
      setState(() => _isCharging = isCharging);
    });
  }
  
  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Text(_isCharging ? '充電中' : '未充電');
  }
}
```

### Android EventChannel

```kotlin
class MainActivity: FlutterActivity() {
    private val EVENT_CHANNEL = "com.example.app/charging"
    
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        
        EventChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            EVENT_CHANNEL
        ).setStreamHandler(object : EventChannel.StreamHandler {
            private var chargingStateReceiver: BroadcastReceiver? = null
            
            override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
                chargingStateReceiver = createChargingStateReceiver(events)
                val filter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
                registerReceiver(chargingStateReceiver, filter)
            }
            
            override fun onCancel(arguments: Any?) {
                unregisterReceiver(chargingStateReceiver)
                chargingStateReceiver = null
            }
        })
    }
    
    private fun createChargingStateReceiver(
        events: EventChannel.EventSink?
    ): BroadcastReceiver {
        return object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                val status = intent.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
                val isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING ||
                                status == BatteryManager.BATTERY_STATUS_FULL
                events?.success(isCharging)
            }
        }
    }
}
```

## 傳遞複雜資料

使用 Map 傳遞結構化資料。

```dart
// Flutter 端
class LocationService {
  static const platform = MethodChannel('com.example.app/location');
  
  Future<Map<String, double>?> getCurrentLocation() async {
    try {
      final Map<dynamic, dynamic> result = 
          await platform.invokeMethod('getLocation');
      return {
        'latitude': result['latitude'] as double,
        'longitude': result['longitude'] as double,
      };
    } catch (e) {
      print('Failed to get location: $e');
      return null;
    }
  }
}
```

```kotlin
// Android
"getLocation" -> {
    val location = getCurrentLocation()
    if (location != null) {
        val locationMap = mapOf(
            "latitude" to location.latitude,
            "longitude" to location.longitude
        )
        result.success(locationMap)
    } else {
        result.error("UNAVAILABLE", "Location not available", null)
    }
}
```

## 藍牙整合範例

充電站 App 需要透過藍牙連接充電樁。

```dart
// Flutter 端
class BluetoothService {
  static const platform = MethodChannel('com.example.app/bluetooth');
  static const eventChannel = EventChannel('com.example.app/bluetooth_scan');
  
  // 掃描裝置
  Stream<List<BluetoothDevice>> scanDevices() {
    return eventChannel.receiveBroadcastStream().map((event) {
      final List<dynamic> devices = event as List;
      return devices.map((d) => BluetoothDevice.fromMap(d)).toList();
    });
  }
  
  // 連接裝置
  Future<bool> connect(String deviceId) async {
    try {
      final bool result = await platform.invokeMethod('connect', {
        'deviceId': deviceId,
      });
      return result;
    } catch (e) {
      return false;
    }
  }
  
  // 發送指令
  Future<String?> sendCommand(String command) async {
    try {
      final String result = await platform.invokeMethod('sendCommand', {
        'command': command,
      });
      return result;
    } catch (e) {
      return null;
    }
  }
  
  // 斷開連接
  Future<void> disconnect() async {
    await platform.invokeMethod('disconnect');
  }
}

class BluetoothDevice {
  final String id;
  final String name;
  final int rssi;
  
  BluetoothDevice({
    required this.id,
    required this.name,
    required this.rssi,
  });
  
  factory BluetoothDevice.fromMap(Map<dynamic, dynamic> map) {
    return BluetoothDevice(
      id: map['id'] as String,
      name: map['name'] as String,
      rssi: map['rssi'] as int,
    );
  }
}
```

```kotlin
// Android 藍牙實作（簡化版）
import android.bluetooth.*

class MainActivity: FlutterActivity() {
    private var bluetoothGatt: BluetoothGatt? = null
    
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        // MethodChannel
        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            "com.example.app/bluetooth"
        ).setMethodCallHandler { call, result ->
            when (call.method) {
                "connect" -> {
                    val deviceId = call.argument<String>("deviceId")
                    connectToDevice(deviceId, result)
                }
                "sendCommand" -> {
                    val command = call.argument<String>("command")
                    sendBluetoothCommand(command, result)
                }
                "disconnect" -> {
                    bluetoothGatt?.disconnect()
                    result.success(null)
                }
                else -> result.notImplemented()
            }
        }
        
        // EventChannel for scanning
        EventChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            "com.example.app/bluetooth_scan"
        ).setStreamHandler(BluetoothScanHandler())
    }
    
    private fun connectToDevice(deviceId: String?, result: MethodChannel.Result) {
        // 實際藍牙連接邏輯
        // bluetoothGatt = device.connectGatt(...)
        result.success(true)
    }
    
    private fun sendBluetoothCommand(
        command: String?,
        result: MethodChannel.Result
    ) {
        // 發送 BLE 指令
        result.success("OK")
    }
}
```

## 權限處理

Native 功能常需要權限。

```dart
class PermissionService {
  static const platform = MethodChannel('com.example.app/permissions');
  
  Future<bool> requestLocationPermission() async {
    try {
      final bool granted = await platform.invokeMethod('requestLocation');
      return granted;
    } catch (e) {
      return false;
    }
  }
}
```

```kotlin
// Android
import android.Manifest
import android.content.pm.PackageManager
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat

"requestLocation" -> {
    if (ContextCompat.checkSelfPermission(
            this,
            Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED
    ) {
        result.success(true)
    } else {
        ActivityCompat.requestPermissions(
            this,
            arrayOf(Manifest.permission.ACCESS_FINE_LOCATION),
            LOCATION_PERMISSION_CODE
        )
        // 結果在 onRequestPermissionsResult 回調
    }
}
```

## 實務建議

Platform Channels 是同步阻塞的，不要執行耗時操作。

錯誤處理很重要，Native 可能拋出各種例外。

善用現有套件（如 `permission_handler`、`flutter_blue`），不要重造輪子。

測試時兩個平台都要測，行為可能不一致。

Native 程式碼寫在 MainActivity 會很亂，建議抽出獨立的 Handler 類別。

文件化你的 Channel 介面，團隊成員才知道怎麼用。

下週研究進階狀態管理：Riverpod 的深度應用和最佳實踐。
