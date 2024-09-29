---
layout: post
title: "Flutter Deep Link 與 App Links：從網頁喚起 App"
date: 2024-09-30 11:50:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Deep Link, App Links, Universal Links]
---

這週處理 Deep Link 功能。使用者點擊充電站分享連結時，應該直接開啟 App 並導航到該充電站頁面，而不是開啟網頁。

Deep Link 有三種：
- **Custom URL Scheme** (舊方法)：`myapp://station/123`
- **Universal Links** (iOS)：`https://example.com/station/123` 直接開啟 App
- **App Links** (Android)：同上，Android 版本

現在主推 Universal Links 和 App Links，因為可以 fall back 到網頁。

## Custom URL Scheme（基礎）

最簡單的方法，但使用者體驗較差（會跳出選擇 App 的對話框）。

### Android 設定

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
  <application>
    <activity android:name=".MainActivity">
      <!-- 現有的 intent-filter -->
      <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
      </intent-filter>
      
      <!-- Deep Link intent-filter -->
      <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        
        <!-- 自訂 scheme -->
        <data
          android:scheme="chargingapp"
          android:host="station" />
      </intent-filter>
    </activity>
  </application>
</manifest>
```

### iOS 設定

```xml
<!-- ios/Runner/Info.plist -->
<dict>
  <!-- 現有設定 -->
  
  <!-- Deep Link -->
  <key>CFBundleURLTypes</key>
  <array>
    <dict>
      <key>CFBundleTypeRole</key>
      <string>Editor</string>
      <key>CFBundleURLSchemes</key>
      <array>
        <string>chargingapp</string>
      </array>
    </dict>
  </array>
</dict>
```

現在 `chargingapp://station/123` 會開啟 App。

## 處理 Deep Link

使用 `uni_links` 或 `go_router` 處理。

### 使用 go_router（推薦）

```yaml
dependencies:
  go_router: ^12.0.0
```

```dart
// router/app_router.dart
import 'package:go_router/go_router.dart';

final router = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => MapScreen(),
    ),
    GoRoute(
      path: '/station/:id',
      builder: (context, state) {
        final stationId = state.pathParameters['id']!;
        return StationDetailScreen(stationId: stationId);
      },
    ),
    GoRoute(
      path: '/charging/:sessionId',
      builder: (context, state) {
        final sessionId = state.pathParameters['sessionId']!;
        return ChargingScreen(sessionId: sessionId);
      },
    ),
    GoRoute(
      path: '/promotion/:promoId',
      builder: (context, state) {
        final promoId = state.pathParameters['promoId']!;
        return PromotionScreen(promoId: promoId);
      },
    ),
  ],
  errorBuilder: (context, state) => ErrorScreen(error: state.error),
);

// 在 MaterialApp 使用
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: router,
      title: 'Charging App',
    );
  }
}
```

`go_router` 會自動處理 Deep Link，路徑匹配就導航。

測試：
- iOS: `xcrun simctl openurl booted "chargingapp://station/123"`
- Android: `adb shell am start -W -a android.intent.action.VIEW -d "chargingapp://station/123" com.example.chargingapp`

### 使用 uni_links（手動處理）

如果不用 go_router，可以手動處理：

```yaml
dependencies:
  uni_links: ^0.5.1
```

```dart
import 'package:uni_links/uni_links.dart';
import 'dart:async';

class DeepLinkService {
  StreamSubscription? _subscription;
  
  Future<void> init() async {
    // 處理 App 關閉時的 Deep Link
    try {
      final initialLink = await getInitialLink();
      if (initialLink != null) {
        _handleDeepLink(initialLink);
      }
    } catch (e) {
      print('Error getting initial link: $e');
    }
    
    // 監聽 App 開啟時的 Deep Link
    _subscription = linkStream.listen(
      (String? link) {
        if (link != null) {
          _handleDeepLink(link);
        }
      },
      onError: (err) {
        print('Error listening to links: $err');
      },
    );
  }
  
  void _handleDeepLink(String link) {
    final uri = Uri.parse(link);
    
    // chargingapp://station/123
    if (uri.host == 'station') {
      final stationId = uri.pathSegments.first;
      navigatorKey.currentState?.pushNamed(
        '/station',
        arguments: stationId,
      );
    }
    // chargingapp://charging/456
    else if (uri.host == 'charging') {
      final sessionId = uri.pathSegments.first;
      navigatorKey.currentState?.pushNamed(
        '/charging',
        arguments: sessionId,
      );
    }
  }
  
  void dispose() {
    _subscription?.cancel();
  }
}
```

## Universal Links (iOS)

Universal Links 讓 iOS 使用 HTTPS 連結直接開啟 App，沒有安裝則開啟網頁。

### 步驟 1：設定 AASA 檔案

在網站根目錄放置 `apple-app-site-association` 檔案：

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.example.chargingapp",
        "paths": [
          "/station/*",
          "/charging/*",
          "/promotion/*"
        ]
      }
    ]
  }
}
```

- `TEAMID`: 在 Apple Developer 的 Team ID
- `paths`: 要處理的路徑

檔案位置：`https://example.com/.well-known/apple-app-site-association`

**注意**：
- 不要有副檔名
- Content-Type 必須是 `application/json`
- 必須用 HTTPS

驗證：`https://example.com/.well-known/apple-app-site-association` 可以訪問。

### 步驟 2：iOS 設定

在 Xcode 開啟專案：

1. **Signing & Capabilities** → **Add Capability** → **Associated Domains**
2. 新增 domain：`applinks:example.com`

或直接編輯：

```xml
<!-- ios/Runner/Runner.entitlements -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.developer.associated-domains</key>
  <array>
    <string>applinks:example.com</string>
    <string>applinks:www.example.com</string>
  </array>
</dict>
</plist>
```

### 步驟 3：處理連結

用 `go_router` 自動處理，或手動：

```dart
// 在 AppDelegate.swift (如果要客製化處理)
import Flutter
import UIKit

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    continue userActivity: NSUserActivity,
    restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void
  ) -> Bool {
    if userActivity.activityType == NSUserActivityTypeBrowsingWeb,
       let url = userActivity.webpageURL {
      // 傳遞給 Flutter
      let urlString = url.absoluteString
      let channel = FlutterMethodChannel(
        name: "deep_link",
        binaryMessenger: window?.rootViewController as! FlutterBinaryMessenger
      )
      channel.invokeMethod("onDeepLink", arguments: urlString)
      return true
    }
    return super.application(application, continue: userActivity, restorationHandler: restorationHandler)
  }
}
```

## App Links (Android)

Android 版的 Universal Links。

### 步驟 1：設定 assetlinks.json

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.chargingapp",
    "sha256_cert_fingerprints": [
      "AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99"
    ]
  }
}]
```

檔案位置：`https://example.com/.well-known/assetlinks.json`

取得 SHA256 fingerprint：

```bash
# Debug keystore
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android

# Release keystore
keytool -list -v -keystore /path/to/keystore.jks -alias your-key-alias
```

### 步驟 2：Android 設定

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity android:name=".MainActivity">
  <!-- Deep Link (HTTP/HTTPS) -->
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    
    <data
      android:scheme="https"
      android:host="example.com"
      android:pathPrefix="/station" />
    <data
      android:scheme="https"
      android:host="example.com"
      android:pathPrefix="/charging" />
    <data
      android:scheme="https"
      android:host="example.com"
      android:pathPrefix="/promotion" />
  </intent-filter>
</activity>
```

`android:autoVerify="true"` 會自動驗證 assetlinks.json。

### 驗證

```bash
# 測試
adb shell am start -W -a android.intent.action.VIEW -d "https://example.com/station/123" com.example.chargingapp

# 查看驗證狀態
adb shell pm get-app-links com.example.chargingapp
```

## Query Parameters 處理

Deep Link 常帶參數：`https://example.com/station/123?source=email&campaign=summer`

```dart
GoRoute(
  path: '/station/:id',
  builder: (context, state) {
    final stationId = state.pathParameters['id']!;
    final source = state.uri.queryParameters['source'];
    final campaign = state.uri.queryParameters['campaign'];
    
    // 記錄來源（用於分析）
    analytics.logEvent(
      name: 'deep_link_opened',
      parameters: {
        'station_id': stationId,
        'source': source ?? 'unknown',
        'campaign': campaign ?? 'none',
      },
    );
    
    return StationDetailScreen(stationId: stationId);
  },
)
```

## Deferred Deep Link

使用者沒有安裝 App 時，引導到 App Store/Play Store 下載，安裝後再導航到目標頁面。

用 **Branch** 或 **Firebase Dynamic Links**（已停止新專案）。

### Branch.io

```yaml
dependencies:
  flutter_branch_sdk: ^6.0.0
```

```dart
class BranchService {
  Future<void> init() async {
    FlutterBranchSdk.init(
      useTestKey: false,
      enableLogging: true,
    );
    
    // 監聽 Deep Link
    FlutterBranchSdk.listSession().listen((data) {
      if (data.containsKey('+clicked_branch_link')) {
        final stationId = data['station_id'];
        if (stationId != null) {
          navigatorKey.currentState?.pushNamed(
            '/station',
            arguments: stationId,
          );
        }
      }
    });
  }
  
  Future<String> createDeepLink(String stationId) async {
    final buo = BranchUniversalObject(
      canonicalIdentifier: 'station/$stationId',
      title: '充電站',
      contentDescription: '查看充電站詳情',
      contentMetadata: BranchContentMetaData()
        ..addCustomMetadata('station_id', stationId),
    );
    
    final lp = BranchLinkProperties(
      channel: 'sharing',
      feature: 'station_share',
    );
    
    final response = await FlutterBranchSdk.getShortUrl(
      buo: buo,
      linkProperties: lp,
    );
    
    return response.success ? response.result : '';
  }
}
```

## 分享功能

產生 Deep Link 給使用者分享：

```dart
import 'package:share_plus/share_plus.dart';

Future<void> shareStation(ChargingStation station) async {
  final deepLink = 'https://example.com/station/${station.id}';
  
  await Share.share(
    '${station.name}\n${station.address}\n\n$deepLink',
    subject: '分享充電站',
  );
}
```

## 測試 Deep Link

### iOS 測試

```bash
# 模擬器
xcrun simctl openurl booted "https://example.com/station/123"

# 實機（用 Notes 或 Safari 點擊連結）
```

### Android 測試

```bash
# 模擬器/實機
adb shell am start -W -a android.intent.action.VIEW \
  -d "https://example.com/station/123" \
  com.example.chargingapp
```

### 除錯

iOS 檢查 Universal Links 是否正確設定：
1. Settings → Developer → Universal Links Testing
2. 輸入網址測試

Android 查看驗證狀態：
```bash
adb shell pm get-app-links --user cur com.example.chargingapp
```

## 實務心得

Deep Link 設定繁瑣，但很重要。可以大幅提升使用者從網頁到 App 的轉換率。

Custom URL Scheme 最簡單，但體驗差，只適合開發測試。

Universal Links / App Links 是正確做法，但需要擁有網域和 HTTPS。

AASA / assetlinks.json 檔案一定要能從外部訪問，很多問題都是這個。

測試時要用實機，模擬器的行為有時不準確（尤其 iOS）。

記得處理錯誤情況：連結格式錯誤、資源不存在、使用者未登入等。

Deep Link 分析很重要，追蹤哪些管道帶來最多流量。

下週來研究 Flutter 的背景任務處理，讓 App 在背景也能執行某些操作。
