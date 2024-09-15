---
layout: post
title: "Flutter 國際化與在地化：多語言支援實作"
date: 2024-09-16 09:25:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, i18n, l10n, Internationalization]
---

這週處理多語言支援。充電站 App 要上架到台灣、香港、新加坡，需要支援繁體中文、簡體中文、英文。Flutter 的國際化（i18n）和在地化（l10n）機制還算完整。

**i18n (Internationalization)**: 讓 App 可以支援多語言的架構設計。

**l10n (Localization)**: 實際翻譯和在地化內容（日期格式、貨幣符號等）。

## 官方 intl 套件

Flutter 官方推薦用 `intl` 套件。

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: ^0.18.0

dev_dependencies:
  build_runner: ^2.4.0
  intl_translation: ^0.18.0
```

啟用 l10n 工具：

```yaml
# pubspec.yaml
flutter:
  generate: true
```

設定檔：

```yaml
# l10n.yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

## ARB 檔案

ARB (Application Resource Bundle) 是 JSON 格式的翻譯檔。

### 英文（範本）

{% raw %}
```json
{
  "@@locale": "en",
  
  "appTitle": "Charging App",
  "@appTitle": {
    "description": "Title of the application"
  },
  
  "welcomeMessage": "Welcome to Charging App",
  "@welcomeMessage": {
    "description": "Welcome message on home screen"
  },
  
  "stationCount": "{count, plural, =0{No stations} =1{1 station} other{{count} stations}}",
  "@stationCount": {
    "description": "Number of charging stations",
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  },
  
  "availableChargers": "{available} / {total} available",
  "@availableChargers": {
    "placeholders": {
      "available": {
        "type": "int"
      },
      "total": {
        "type": "int"
      }
    }
  },
  
  "distanceKm": "{distance} km away",
  "@distanceKm": {
    "placeholders": {
      "distance": {
        "type": "double",
        "format": "decimalPattern"
      }
    }
  },
  
  "chargingStatus": "Charging: {progress}%",
  "@chargingStatus": {
    "placeholders": {
      "progress": {
        "type": "int"
      }
    }
  },
  
  "startCharging": "Start Charging",
  "stopCharging": "Stop Charging",
  "history": "History",
  "profile": "Profile",
  "settings": "Settings",
  
  "networkError": "Network connection failed",
  "retry": "Retry",
  "cancel": "Cancel",
  "confirm": "Confirm"
}
```
{% endraw %}

### 繁體中文

{% raw %}
```json
{
  "@@locale": "zh_TW",
  
  "appTitle": "充電 App",
  "welcomeMessage": "歡迎使用充電 App",
  
  "stationCount": "{count, plural, =0{沒有充電站} other{{count} 個充電站}}",
  "availableChargers": "{available} / {total} 可用",
  "distanceKm": "距離 {distance} 公里",
  "chargingStatus": "充電中：{progress}%",
  
  "startCharging": "開始充電",
  "stopCharging": "停止充電",
  "history": "歷史紀錄",
  "profile": "個人資料",
  "settings": "設定",
  
  "networkError": "網路連線失敗",
  "retry": "重試",
  "cancel": "取消",
  "confirm": "確認"
}
```
{% endraw %}

### 簡體中文

{% raw %}
```json
{
  "@@locale": "zh_CN",
  
  "appTitle": "充电 App",
  "welcomeMessage": "欢迎使用充电 App",
  
  "stationCount": "{count, plural, =0{没有充电站} other{{count} 个充电站}}",
  "availableChargers": "{available} / {total} 可用",
  "distanceKm": "距离 {distance} 公里",
  "chargingStatus": "充电中：{progress}%",
  
  "startCharging": "开始充电",
  "stopCharging": "停止充电",
  "history": "历史记录",
  "profile": "个人资料",
  "settings": "设置",
  
  "networkError": "网络连接失败",
  "retry": "重试",
  "cancel": "取消",
  "confirm": "确认"
}
```
{% endraw %}

建立檔案：
- `lib/l10n/app_en.arb`
- `lib/l10n/app_zh_TW.arb`
- `lib/l10n/app_zh_CN.arb`

## 產生程式碼

```bash
flutter gen-l10n
```

會在 `.dart_tool/flutter_gen/gen_l10n/` 產生：
- `app_localizations.dart`
- `app_localizations_en.dart`
- `app_localizations_zh.dart`

## 在 App 中使用

### 設定 MaterialApp

```dart
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Charging App',
      
      // 支援的語言
      localizationsDelegates: [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      
      // 支援的 Locale
      supportedLocales: [
        Locale('en'),
        Locale('zh', 'TW'),
        Locale('zh', 'CN'),
      ],
      
      // Locale 解析策略
      localeResolutionCallback: (locale, supportedLocales) {
        // 優先使用完全匹配
        for (var supportedLocale in supportedLocales) {
          if (supportedLocale.languageCode == locale?.languageCode &&
              supportedLocale.countryCode == locale?.countryCode) {
            return supportedLocale;
          }
        }
        
        // 次要使用語言匹配
        for (var supportedLocale in supportedLocales) {
          if (supportedLocale.languageCode == locale?.languageCode) {
            return supportedLocale;
          }
        }
        
        // 預設英文
        return supportedLocales.first;
      },
      
      home: MapScreen(),
    );
  }
}
```

### 在 Widget 中使用

```dart
class MapScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;
    
    return Scaffold(
      appBar: AppBar(
        title: Text(l10n.appTitle),
      ),
      body: Column(
        children: [
          Text(l10n.welcomeMessage),
          
          // 帶參數的翻譯
          Text(l10n.stationCount(stations.length)),
          
          // 多個參數
          Text(l10n.availableChargers(3, 5)),
          
          // 格式化數字
          Text(l10n.distanceKm(2.5)),
          
          ElevatedButton(
            onPressed: () {},
            child: Text(l10n.startCharging),
          ),
        ],
      ),
      bottomNavigationBar: BottomNavigationBar(
        items: [
          BottomNavigationBarItem(
            icon: Icon(Icons.map),
            label: l10n.appTitle,
          ),
          BottomNavigationBarItem(
            icon: Icon(Icons.history),
            label: l10n.history,
          ),
          BottomNavigationBarItem(
            icon: Icon(Icons.person),
            label: l10n.profile,
          ),
        ],
      ),
    );
  }
}
```

## 動態切換語言

使用者可以在 App 內切換語言。

### 用 Riverpod 管理 Locale

```dart
// providers/locale_provider.dart
final localeProvider = StateProvider<Locale>((ref) {
  // 預設使用系統語言
  return PlatformDispatcher.instance.locale;
});
```

### 在 MaterialApp 使用

```dart
class MyApp extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeProvider);
    
    return MaterialApp(
      locale: locale,
      localizationsDelegates: [/* ... */],
      supportedLocales: [/* ... */],
      home: MapScreen(),
    );
  }
}
```

### 語言選擇器

```dart
class LanguageSelector extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final currentLocale = ref.watch(localeProvider);
    
    return DropdownButton<Locale>(
      value: currentLocale,
      items: [
        DropdownMenuItem(
          value: Locale('en'),
          child: Text('English'),
        ),
        DropdownMenuItem(
          value: Locale('zh', 'TW'),
          child: Text('繁體中文'),
        ),
        DropdownMenuItem(
          value: Locale('zh', 'CN'),
          child: Text('简体中文'),
        ),
      ],
      onChanged: (locale) {
        if (locale != null) {
          ref.read(localeProvider.notifier).state = locale;
          
          // 儲存偏好
          SharedPreferences.getInstance().then((prefs) {
            prefs.setString('locale', locale.toString());
          });
        }
      },
    );
  }
}
```

### 載入儲存的語言偏好

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  final prefs = await SharedPreferences.getInstance();
  final savedLocale = prefs.getString('locale');
  
  runApp(
    ProviderScope(
      overrides: [
        if (savedLocale != null)
          localeProvider.overrideWith((ref) => _parseLocale(savedLocale)),
      ],
      child: MyApp(),
    ),
  );
}

Locale _parseLocale(String localeString) {
  final parts = localeString.split('_');
  if (parts.length == 2) {
    return Locale(parts[0], parts[1]);
  }
  return Locale(parts[0]);
}
```

## 進階功能

### 複數規則

不同語言的複數規則不同，ARB 支援：

{% raw %}
```json
{
  "itemCount": "{count, plural, =0{No items} =1{One item} other{{count} items}}"
}
```
{% endraw %}

中文沒有複數，但仍要定義：

{% raw %}
```json
{
  "itemCount": "{count, plural, other{{count} 個項目}}"
}
```
{% endraw %}

### 性別

```json
{
  "greeting": "{gender, select, male{Mr.} female{Ms.} other{}} {name}",
  "@greeting": {
    "placeholders": {
      "gender": {},
      "name": {}
    }
  }
}
```

### 日期和時間

用 `intl` 套件格式化：

```dart
import 'package:intl/intl.dart';

// 自動根據 Locale 格式化
String formatDate(DateTime date, Locale locale) {
  return DateFormat.yMMMd(locale.toString()).format(date);
}

// 使用
final locale = Localizations.localeOf(context);
final formattedDate = formatDate(DateTime.now(), locale);

// en: Jan 1, 2024
// zh_TW: 2024年1月1日
// zh_CN: 2024年1月1日
```

時間格式：

```dart
String formatTime(DateTime time, Locale locale) {
  return DateFormat.jm(locale.toString()).format(time);
}

// en: 3:30 PM
// zh_TW: 下午3:30
// zh_CN: 下午3:30
```

### 貨幣

```dart
String formatCurrency(double amount, Locale locale) {
  final format = NumberFormat.currency(
    locale: locale.toString(),
    symbol: _getCurrencySymbol(locale),
  );
  return format.format(amount);
}

String _getCurrencySymbol(Locale locale) {
  switch (locale.countryCode) {
    case 'TW':
      return 'NT\$';
    case 'CN':
      return '¥';
    case 'SG':
      return 'S\$';
    default:
      return '\$';
  }
}

// TW: NT$100
// CN: ¥100
// SG: S$100
```

## 圖片和資源的在地化

不同語言可能需要不同的圖片。

```
assets/
  images/
    en/
      welcome.png
    zh_TW/
      welcome.png
    zh_CN/
      welcome.png
```

載入在地化圖片：

```dart
String getLocalizedImagePath(String imageName) {
  final locale = Localizations.localeOf(context);
  final localeString = locale.countryCode != null
      ? '${locale.languageCode}_${locale.countryCode}'
      : locale.languageCode;
  
  return 'assets/images/$localeString/$imageName';
}

// 使用
Image.asset(getLocalizedImagePath('welcome.png'))
```

## 右至左 (RTL) 支援

阿拉伯文、希伯來文是從右到左。

```dart
MaterialApp(
  // ...
  builder: (context, child) {
    final locale = Localizations.localeOf(context);
    final isRTL = _isRTLLanguage(locale);
    
    return Directionality(
      textDirection: isRTL ? TextDirection.rtl : TextDirection.ltr,
      child: child!,
    );
  },
)

bool _isRTLLanguage(Locale locale) {
  return ['ar', 'he', 'fa', 'ur'].contains(locale.languageCode);
}
```

## 翻譯管理

翻譯檔越來越多，手動管理會亂。

### 使用 POEditor

[POEditor](https://poeditor.com/) 是線上翻譯管理平台。

1. 上傳 `app_en.arb`
2. 邀請翻譯人員
3. 翻譯完成後匯出 ARB

### 自動化腳本

```bash
# upload_translations.sh
#!/bin/bash

# 上傳英文範本到 POEditor
curl -X POST https://api.poeditor.com/v2/projects/upload \
  -F api_token="$POEDITOR_API_TOKEN" \
  -F id="$POEDITOR_PROJECT_ID" \
  -F updating="terms_translations" \
  -F file=@"lib/l10n/app_en.arb"
```

```bash
# download_translations.sh
#!/bin/bash

for lang in zh-TW zh-CN; do
  curl -X POST https://api.poeditor.com/v2/projects/export \
    -d api_token="$POEDITOR_API_TOKEN" \
    -d id="$POEDITOR_PROJECT_ID" \
    -d language="$lang" \
    -d type="arb" \
    | jq -r '.result.url' \
    | xargs curl -o "lib/l10n/app_${lang//-/_}.arb"
done
```

## Context-aware 翻譯

有些詞在不同情境有不同翻譯。

```json
{
  "settingsTitle": "Settings",
  "settingsButton": "Settings",
  
  "chargeNoun": "Charge",
  "chargeVerb": "Charge"
}
```

中文：

```json
{
  "settingsTitle": "設定",
  "settingsButton": "設定",
  
  "chargeNoun": "電量",
  "chargeVerb": "充電"
}
```

## 實務心得

多語言支援要從一開始就規劃，後期補很麻煩。所有 UI 文字都要用 l10n，不要寫死。

ARB 檔案的 key 命名要清楚，加上 description 說明情境，幫助翻譯人員。

複數和性別規則各語言不同，不要假設所有語言都跟英文一樣。

日期、時間、貨幣格式也是在地化的一部分，不只是翻譯文字。

測試時要切換不同語言檢查排版，有些語言的文字比較長（德文、俄文），可能會破版。

翻譯品質很重要，機器翻譯只能參考，最好請母語人士校稿。

下週來研究 Flutter 的狀態持久化，讓使用者的偏好設定和資料能夠保存。
