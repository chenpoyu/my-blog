---
layout: post
title: "Flutter 安全性：程式碼混淆、加密與防護"
date: 2024-10-28 13:55:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Security, Obfuscation, Encryption]
---

App 安全性很重要，尤其涉及使用者資料和金流。這週研究 Flutter 的安全防護：程式碼混淆、敏感資料加密、網路安全等。

## 程式碼混淆 (Code Obfuscation)

混淆讓反編譯後的程式碼難以閱讀。

### 啟用混淆

```bash
# Android
flutter build apk --obfuscate --split-debug-info=build/app/outputs/symbols

# iOS
flutter build ios --obfuscate --split-debug-info=build/ios/outputs/symbols
```

`--obfuscate`: 混淆程式碼
`--split-debug-info`: 分離 debug symbols（用於 Crashlytics 還原）

### 保留特定類別

某些類別不能混淆（例如用於序列化的 Model）。

```dart
// lib/models/user.dart
@pragma('vm:entry-point')
class User {
  final String id;
  final String name;
  
  User({required this.id, required this.name});
  
  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['id'],
    name: json['name'],
  );
}
```

### ProGuard (Android)

```
# android/app/proguard-rules.pro
-keep class com.example.chargingapp.** { *; }
-keep class io.flutter.** { *; }

# 保留序列化 Model
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer

# Firebase
-keep class com.google.firebase.** { *; }
```

## 敏感資料加密

不要明文儲存 Token、密碼、API Key。

### Flutter Secure Storage

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

```dart
class SecureStorageService {
  final _storage = FlutterSecureStorage();
  
  // 儲存
  Future<void> saveToken(String token) async {
    await _storage.write(key: 'auth_token', value: token);
  }
  
  // 讀取
  Future<String?> getToken() async {
    return await _storage.read(key: 'auth_token');
  }
  
  // 刪除
  Future<void> deleteToken() async {
    await _storage.delete(key: 'auth_token');
  }
  
  // 清空全部
  Future<void> deleteAll() async {
    await _storage.deleteAll();
  }
}
```

iOS 使用 Keychain，Android 使用 KeyStore。

### 加密敏感資料

如果需要自行加密：

```yaml
dependencies:
  encrypt: ^5.0.0
```

```dart
import 'package:encrypt/encrypt.dart';

class EncryptionService {
  late final Key _key;
  late final IV _iv;
  late final Encrypter _encrypter;
  
  EncryptionService() {
    // 實際應用中，key 應該從安全來源取得
    _key = Key.fromSecureRandom(32);
    _iv = IV.fromSecureRandom(16);
    _encrypter = Encrypter(AES(_key));
  }
  
  String encrypt(String plainText) {
    final encrypted = _encrypter.encrypt(plainText, iv: _iv);
    return encrypted.base64;
  }
  
  String decrypt(String encryptedText) {
    final encrypted = Encrypted.fromBase64(encryptedText);
    return _encrypter.decrypt(encrypted, iv: _iv);
  }
}

// 使用
final encryption = EncryptionService();
final encryptedPassword = encryption.encrypt('myPassword123');
final decryptedPassword = encryption.decrypt(encryptedPassword);
```

## 網路安全

### SSL Pinning

防止中間人攻擊 (MITM)。

```yaml
dependencies:
  dio: ^5.0.0
```

```dart
class ApiClient {
  late Dio _dio;
  
  ApiClient() {
    _dio = Dio(BaseOptions(
      baseUrl: 'https://api.example.com',
    ));
    
    // SSL Pinning
    (_dio.httpClientAdapter as DefaultHttpClientAdapter).onHttpClientCreate = 
        (client) {
      client.badCertificateCallback = 
          (X509Certificate cert, String host, int port) {
        // 驗證憑證指紋
        final certSha256 = sha256.convert(cert.der).toString();
        const expectedSha256 = 'AA:BB:CC:DD...'; // 你的憑證指紋
        
        return certSha256 == expectedSha256;
      };
      return client;
    };
  }
}
```

取得憑證指紋：

```bash
openssl s_client -connect api.example.com:443 < /dev/null | openssl x509 -fingerprint -sha256 -noout
```

### HTTPS Only

```dart
class ApiClient {
  Dio _createDio() {
    final dio = Dio(BaseOptions(
      baseUrl: 'https://api.example.com', // 只用 HTTPS
      connectTimeout: Duration(seconds: 10),
      receiveTimeout: Duration(seconds: 10),
    ));
    
    // 阻擋 HTTP 請求
    dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) {
        if (!options.uri.isScheme('HTTPS')) {
          return handler.reject(
            DioException(
              requestOptions: options,
              error: 'HTTP not allowed, use HTTPS only',
            ),
          );
        }
        handler.next(options);
      },
    ));
    
    return dio;
  }
}
```

## 防止截圖 (Android)

某些敏感畫面（如支付）防止截圖。

```dart
import 'package:flutter/services.dart';

class SecureScreen extends StatefulWidget {
  final Widget child;
  
  const SecureScreen({required this.child});
  
  @override
  _SecureScreenState createState() => _SecureScreenState();
}

class _SecureScreenState extends State<SecureScreen> {
  @override
  void initState() {
    super.initState();
    _enableSecureMode();
  }
  
  @override
  void dispose() {
    _disableSecureMode();
    super.dispose();
  }
  
  void _enableSecureMode() {
    if (Platform.isAndroid) {
      SystemChannels.platform.invokeMethod('SystemChrome.setSecureScreen', true);
    }
  }
  
  void _disableSecureMode() {
    if (Platform.isAndroid) {
      SystemChannels.platform.invokeMethod('SystemChrome.setSecureScreen', false);
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
}

// 使用
class PaymentScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return SecureScreen(
      child: Scaffold(/* payment UI */),
    );
  }
}
```

Android 需要在 MainActivity 設定：

```kotlin
// android/app/src/main/kotlin/.../MainActivity.kt
import android.view.WindowManager

class MainActivity: FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "SystemChrome")
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "setSecureScreen" -> {
                        val secure = call.arguments as Boolean
                        if (secure) {
                            window.setFlags(
                                WindowManager.LayoutParams.FLAG_SECURE,
                                WindowManager.LayoutParams.FLAG_SECURE
                            )
                        } else {
                            window.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
                        }
                        result.success(null)
                    }
                    else -> result.notImplemented()
                }
            }
    }
}
```

## Root/Jailbreak 檢測

偵測裝置是否被 Root (Android) 或 Jailbreak (iOS)。

```yaml
dependencies:
  flutter_jailbreak_detection: ^1.0.0
```

```dart
class SecurityCheckService {
  Future<bool> isDeviceSecure() async {
    bool jailbroken = await FlutterJailbreakDetection.jailbroken;
    bool developerMode = await FlutterJailbreakDetection.developerMode;
    
    return !jailbroken && !developerMode;
  }
  
  Future<void> checkDeviceSecurity() async {
    final isSecure = await isDeviceSecure();
    
    if (!isSecure) {
      // 顯示警告或阻止使用
      showDialog(
        context: context,
        barrierDismissible: false,
        builder: (context) => AlertDialog(
          title: Text('安全警告'),
          content: Text('偵測到您的裝置已被 Root/Jailbreak，為了保護您的資料安全，部分功能將被限制。'),
          actions: [
            TextButton(
              onPressed: () => exit(0),
              child: Text('關閉'),
            ),
          ],
        ),
      );
    }
  }
}
```

## API Key 保護

不要把 API Key 寫死在程式碼。

### 使用環境變數

```dart
// lib/config/env.dart
class Env {
  static const String apiKey = String.fromEnvironment(
    'API_KEY',
    defaultValue: '',
  );
  
  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'https://api.example.com',
  );
}

// 建置時傳入
// flutter build apk --dart-define=API_KEY=your_key_here
```

### 使用 Native 儲存

更安全的方式是放在 Native 層。

Android:
```kotlin
// android/app/src/main/kotlin/.../Config.kt
object Config {
    const val API_KEY = "your_api_key_here"
}

// MainActivity.kt
MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "config")
    .setMethodCallHandler { call, result ->
        when (call.method) {
            "getApiKey" -> result.success(Config.API_KEY)
            else -> result.notImplemented()
        }
    }
```

Flutter:
```dart
class ConfigService {
  static const platform = MethodChannel('config');
  
  Future<String> getApiKey() async {
    return await platform.invokeMethod('getApiKey');
  }
}
```

但最好的做法是由後端處理敏感操作，不要把 Key 放客戶端。

## 輸入驗證

防止 SQL Injection、XSS 等攻擊。

```dart
class InputValidator {
  // Email 驗證
  static bool isValidEmail(String email) {
    final regex = RegExp(
      r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    );
    return regex.hasMatch(email);
  }
  
  // 手機號碼驗證（台灣）
  static bool isValidPhone(String phone) {
    final regex = RegExp(r'^09\d{8}$');
    return regex.hasMatch(phone);
  }
  
  // 清除危險字元
  static String sanitize(String input) {
    return input
        .replaceAll('<', '&lt;')
        .replaceAll('>', '&gt;')
        .replaceAll('"', '&quot;')
        .replaceAll("'", '&#x27;')
        .replaceAll('/', '&#x2F;');
  }
  
  // 密碼強度檢查
  static bool isStrongPassword(String password) {
    // 至少 8 個字元、包含大小寫、數字、特殊符號
    if (password.length < 8) return false;
    
    final hasUppercase = password.contains(RegExp(r'[A-Z]'));
    final hasLowercase = password.contains(RegExp(r'[a-z]'));
    final hasDigit = password.contains(RegExp(r'\d'));
    final hasSpecialChar = password.contains(RegExp(r'[!@#$%^&*(),.?":{}|<>]'));
    
    return hasUppercase && hasLowercase && hasDigit && hasSpecialChar;
  }
}
```

## 實務心得

安全性是持續的過程，不是一次性的。

程式碼混淆只是增加反編譯難度，無法完全防止。敏感邏輯應該放後端。

SSL Pinning 可以防 MITM，但憑證更新時要重新發布 App。

不要過度依賴客戶端驗證，伺服器端也要驗證。

Root/Jailbreak 檢測不是 100% 可靠，只是多一層防護。

隱私和安全要平衡，過度限制會影響使用者體驗。

定期更新依賴套件，修補已知漏洞。

下週來整理 Flutter 的架構演進，以及這個充電站專案的整體回顧。
