---
layout: post
title: "Flutter UI/UX 設計實踐：充電站 App 的設計心得"
date: 2024-08-19 11:15:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, UI/UX, Material Design, 使用者體驗]
---

最後一週整理這陣子開發充電站 App 的 UI/UX 心得。Flutter 的 Widget 很強大,但要做出好的使用者體驗,不只是會寫 Code 就好,要考慮很多細節。

## Material Design vs 自訂設計

Flutter 預設用 Material Design,Google 的設計語言。對開發者來說很方便,但所有 App 看起來都像 Google 風格。

我們這個專案選擇**基於 Material Design,但客製化關鍵元件**。

### 保留 Material Design 的部分

**導航結構**：`Scaffold`, `AppBar`, `BottomNavigationBar` 這些基礎架構保留,使用者已經很習慣。

**互動元素**：`FloatingActionButton`, `SnackBar`, `Dialog` 等,互動邏輯很成熟。

**動畫**：Material 的 transition 和 ripple 效果很流暢。

### 客製化的部分

**顏色主題**：完全自訂品牌色。

**卡片樣式**：充電站卡片、充電樁按鈕,都是自訂設計。

**圖示**：核心功能用自訂 SVG 圖示,不用 Material Icons。

**字型**：使用公司指定的字型（Noto Sans TC）。

```dart
// theme.dart
final appTheme = ThemeData(
  // 基礎 Material Design 3
  useMaterial3: true,
  
  // 品牌色
  colorScheme: ColorScheme.fromSeed(
    seedColor: Color(0xFF00A86B), // 充電綠
    brightness: Brightness.light,
  ),
  
  // 字型
  fontFamily: 'NotoSansTC',
  
  // AppBar 樣式
  appBarTheme: AppBarTheme(
    centerTitle: false,
    elevation: 0,
    backgroundColor: Colors.white,
    foregroundColor: Colors.black87,
    titleTextStyle: TextStyle(
      fontSize: 20,
      fontWeight: FontWeight.w600,
      color: Colors.black87,
      fontFamily: 'NotoSansTC',
    ),
  ),
  
  // Card 樣式
  cardTheme: CardTheme(
    elevation: 2,
    shape: RoundedRectangleBorder(
      borderRadius: BorderRadius.circular(12),
    ),
    clipBehavior: Clip.antiAlias,
  ),
  
  // Button 樣式
  elevatedButtonTheme: ElevatedButtonThemeData(
    style: ElevatedButton.styleFrom(
      padding: EdgeInsets.symmetric(horizontal: 24, vertical: 12),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(8),
      ),
      elevation: 0,
    ),
  ),
);
```

## 響應式佈局

Mobile App 要支援不同尺寸：小螢幕（iPhone SE）、大螢幕（iPad）、橫向模式。

### 使用 LayoutBuilder

```dart
class StationListView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        // 小螢幕：單欄列表
        if (constraints.maxWidth < 600) {
          return ListView.builder(
            itemCount: stations.length,
            itemBuilder: (context, index) => StationCard(
              station: stations[index],
            ),
          );
        }
        
        // 平板：雙欄網格
        return GridView.builder(
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: 2,
            childAspectRatio: 1.5,
            crossAxisSpacing: 16,
            mainAxisSpacing: 16,
          ),
          itemCount: stations.length,
          itemBuilder: (context, index) => StationCard(
            station: stations[index],
          ),
        );
      },
    );
  }
}
```

### 適應性文字大小

不要寫死字型大小,用 Theme 的文字樣式：

```dart
// ❌ 不好
Text(
  'Title',
  style: TextStyle(fontSize: 20),
)

// ✅ 好
Text(
  'Title',
  style: Theme.of(context).textTheme.titleLarge,
)
```

使用者可以在系統設定調整文字大小,`Theme.of(context).textTheme` 會自動適配。

## Loading 狀態設計

Loading 狀態的設計很影響體驗,不能只是轉圈圈。

### 骨架屏（Skeleton）

載入列表時,先顯示骨架,比純 Loading 更好。

```dart
class StationListSkeleton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: 5,
      itemBuilder: (context, index) => Card(
        child: Padding(
          padding: EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // 標題骨架
              Container(
                width: 200,
                height: 20,
                decoration: BoxDecoration(
                  color: Colors.grey[300],
                  borderRadius: BorderRadius.circular(4),
                ),
              ),
              SizedBox(height: 8),
              // 副標骨架
              Container(
                width: 150,
                height: 16,
                decoration: BoxDecoration(
                  color: Colors.grey[300],
                  borderRadius: BorderRadius.circular(4),
                ),
              ),
              SizedBox(height: 12),
              // 按鈕骨架
              Container(
                width: 100,
                height: 36,
                decoration: BoxDecoration(
                  color: Colors.grey[300],
                  borderRadius: BorderRadius.circular(8),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

// 使用
class StationListView extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(stationViewModelProvider);
    
    if (state.isLoading && state.stations.isEmpty) {
      return StationListSkeleton();
    }
    
    return ListView.builder(/* ... */);
  }
}
```

### Shimmer 效果

加上 shimmer 動畫,骨架看起來更生動：

```dart
import 'package:shimmer/shimmer.dart';

Widget buildSkeleton() {
  return Shimmer.fromColors(
    baseColor: Colors.grey[300]!,
    highlightColor: Colors.grey[100]!,
    child: Container(/* skeleton UI */),
  );
}
```

### 下拉刷新

用 `RefreshIndicator` 實作下拉刷新：

```dart
RefreshIndicator(
  onRefresh: () async {
    await ref.read(stationViewModelProvider.notifier).refresh();
  },
  child: ListView.builder(/* ... */),
)
```

## 錯誤狀態設計

錯誤發生時,不只是顯示錯誤訊息,要提供解決方案。

### 網路錯誤

```dart
class NetworkErrorView extends StatelessWidget {
  final VoidCallback onRetry;
  
  const NetworkErrorView({required this.onRetry});
  
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(
            Icons.wifi_off,
            size: 64,
            color: Colors.grey,
          ),
          SizedBox(height: 16),
          Text(
            '網路連線失敗',
            style: Theme.of(context).textTheme.titleLarge,
          ),
          SizedBox(height: 8),
          Text(
            '請檢查網路連線後重試',
            style: Theme.of(context).textTheme.bodyMedium?.copyWith(
              color: Colors.grey,
            ),
          ),
          SizedBox(height: 24),
          ElevatedButton.icon(
            onPressed: onRetry,
            icon: Icon(Icons.refresh),
            label: Text('重試'),
          ),
        ],
      ),
    );
  }
}
```

### 空狀態

沒有資料時,顯示友善的空狀態：

```dart
class EmptyStationView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          // 自訂插圖
          SvgPicture.asset(
            'assets/images/empty_station.svg',
            width: 200,
          ),
          SizedBox(height: 24),
          Text(
            '附近沒有充電站',
            style: Theme.of(context).textTheme.titleLarge,
          ),
          SizedBox(height: 8),
          Text(
            '試試調整搜尋範圍',
            style: Theme.of(context).textTheme.bodyMedium?.copyWith(
              color: Colors.grey,
            ),
          ),
        ],
      ),
    );
  }
}
```

## 動畫與過場

適當的動畫讓 App 有生命力,但不要過度。

### Hero 動畫

列表到詳細頁的過場：

```dart
// 列表頁
Hero(
  tag: 'station_${station.id}',
  child: StationCard(station: station),
)

// 詳細頁
Hero(
  tag: 'station_${station.id}',
  child: StationImage(station: station),
)
```

### 淡入動畫

圖片載入完成後淡入：

```dart
FadeInImage.memoryNetwork(
  placeholder: kTransparentImage,
  image: imageUrl,
  fadeInDuration: Duration(milliseconds: 300),
)
```

或用 `cached_network_image`：

```dart
CachedNetworkImage(
  imageUrl: imageUrl,
  placeholder: (context, url) => Center(
    child: CircularProgressIndicator(),
  ),
  fadeInDuration: Duration(milliseconds: 300),
)
```

### 滑動動畫

充電樁狀態改變時,滑動更新：

```dart
AnimatedSwitcher(
  duration: Duration(milliseconds: 300),
  child: Text(
    charger.status,
    key: ValueKey(charger.status), // 改變時觸發動畫
  ),
)
```

## 無障礙設計

無障礙設計（Accessibility）很重要,但常被忽略。

### Semantics

幫 Widget 加上語義描述,讓 VoiceOver/TalkBack 能朗讀：

```dart
Semantics(
  label: '可用充電樁: ${charger.availableCount} 個',
  child: Text('${charger.availableCount}'),
)
```

### 觸控區域大小

按鈕觸控區域至少 48x48 dp（Material Design 建議）：

```dart
// ❌ 太小
IconButton(
  iconSize: 20,
  padding: EdgeInsets.zero,
  onPressed: onTap,
  icon: Icon(Icons.favorite),
)

// ✅ 足夠大
IconButton(
  onPressed: onTap,
  icon: Icon(Icons.favorite),
  // 預設 48x48
)
```

### 顏色對比

文字和背景要有足夠對比,WCAG 標準至少 4.5:1。

```dart
// 使用 Theme 的顏色可以確保對比度
Text(
  'Content',
  style: TextStyle(
    color: Theme.of(context).colorScheme.onBackground,
  ),
)
```

## 手勢操作

除了點擊,還有滑動、長按、拖曳等手勢。

### 滑動刪除

充電歷史記錄可以滑動刪除：

```dart
Dismissible(
  key: Key(history.id),
  direction: DismissDirection.endToStart,
  background: Container(
    color: Colors.red,
    alignment: Alignment.centerRight,
    padding: EdgeInsets.only(right: 16),
    child: Icon(Icons.delete, color: Colors.white),
  ),
  confirmDismiss: (direction) async {
    return await showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('刪除紀錄'),
        content: Text('確定要刪除這筆充電紀錄嗎？'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: Text('取消'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: Text('刪除'),
          ),
        ],
      ),
    );
  },
  onDismissed: (direction) {
    _deleteHistory(history.id);
  },
  child: ChargingHistoryCard(history: history),
)
```

### 長按選單

長按充電站顯示選單：

```dart
GestureDetector(
  onLongPress: () {
    showModalBottomSheet(
      context: context,
      builder: (context) => Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          ListTile(
            leading: Icon(Icons.favorite),
            title: Text('加入最愛'),
            onTap: () {
              _addToFavorites(station);
              Navigator.pop(context);
            },
          ),
          ListTile(
            leading: Icon(Icons.directions),
            title: Text('開始導航'),
            onTap: () {
              _startNavigation(station);
              Navigator.pop(context);
            },
          ),
          ListTile(
            leading: Icon(Icons.share),
            title: Text('分享'),
            onTap: () {
              _shareStation(station);
              Navigator.pop(context);
            },
          ),
        ],
      ),
    );
  },
  child: StationCard(station: station),
)
```

## 表單輸入

表單設計要減少使用者輸入負擔。

### 智慧鍵盤

根據輸入類型顯示對應鍵盤：

```dart
TextField(
  keyboardType: TextInputType.phone,  // 數字鍵盤
  decoration: InputDecoration(
    labelText: '手機號碼',
  ),
)

TextField(
  keyboardType: TextInputType.emailAddress,  // Email 鍵盤
  decoration: InputDecoration(
    labelText: '電子郵件',
  ),
)
```

### 輸入驗證

即時驗證,不要等送出才檢查：

```dart
TextFormField(
  decoration: InputDecoration(
    labelText: '車牌號碼',
  ),
  validator: (value) {
    if (value == null || value.isEmpty) {
      return '請輸入車牌號碼';
    }
    
    // 台灣車牌格式：ABC-1234
    final regex = RegExp(r'^[A-Z]{3}-\d{4}$');
    if (!regex.hasMatch(value)) {
      return '車牌格式錯誤（例：ABC-1234）';
    }
    
    return null;
  },
)
```

### 自動完成

輸入充電站名稱時,顯示建議：

```dart
Autocomplete<ChargingStation>(
  optionsBuilder: (textEditingValue) {
    if (textEditingValue.text.isEmpty) {
      return const Iterable<ChargingStation>.empty();
    }
    
    return stations.where((station) {
      return station.name
          .toLowerCase()
          .contains(textEditingValue.text.toLowerCase());
    });
  },
  displayStringForOption: (station) => station.name,
  onSelected: (station) {
    _navigateToStation(station);
  },
)
```

## 通知與回饋

使用者操作後要給即時回饋。

### SnackBar

輕量級通知：

```dart
void _showSuccess(String message) {
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Text(message),
      duration: Duration(seconds: 2),
      behavior: SnackBarBehavior.floating,
      action: SnackBarAction(
        label: '查看',
        onPressed: () {
          // 導航到相關頁面
        },
      ),
    ),
  );
}
```

### 觸覺回饋

按鈕點擊時震動：

```dart
import 'package:flutter/services.dart';

void _onButtonPressed() {
  HapticFeedback.lightImpact();  // 輕震動
  
  // 執行動作
  _doSomething();
}

void _onImportantAction() {
  HapticFeedback.mediumImpact();  // 中等震動
  
  // 執行重要動作
}
```

## Dark Mode

支援深色模式,跟隨系統設定。

```dart
MaterialApp(
  theme: lightTheme,
  darkTheme: darkTheme,
  themeMode: ThemeMode.system,  // 跟隨系統
  // ...
)
```

定義深色主題：

```dart
final darkTheme = ThemeData(
  useMaterial3: true,
  brightness: Brightness.dark,
  colorScheme: ColorScheme.fromSeed(
    seedColor: Color(0xFF00A86B),
    brightness: Brightness.dark,
  ),
  // ...
);
```

圖片要根據主題調整：

```dart
Widget buildLogo(BuildContext context) {
  final isDark = Theme.of(context).brightness == Brightness.dark;
  
  return Image.asset(
    isDark ? 'assets/logo_dark.png' : 'assets/logo_light.png',
  );
}
```

## 效能優化

UI 流暢度要注意：

### 避免不必要的 rebuild

用 `const` constructor：

```dart
// ✅ const widget 不會 rebuild
const Text('Title')

// ❌ 每次都會 rebuild
Text('Title')
```

### 列表效能

長列表用 `ListView.builder`：

```dart
// ❌ 一次建立所有 item
ListView(
  children: stations.map((s) => StationCard(s)).toList(),
)

// ✅ 只建立可見的 item
ListView.builder(
  itemCount: stations.length,
  itemBuilder: (context, index) => StationCard(stations[index]),
)
```

### 圖片快取

使用 `cached_network_image` 快取圖片,避免重複下載。

## 實務心得

UI/UX 設計沒有銀彈,要根據使用者情境調整。

這個充電站 App,核心是**快速找到充電站**,所以地圖要大、載入要快、操作要少。詳細資訊放在第二層,不擋住主要功能。

Loading 和錯誤狀態要細緻處理,不能只顧 Happy Path。骨架屏、錯誤重試、空狀態,這些細節累積起來就是好的體驗。

動畫要克制,過度動畫會讓 App 感覺慢。只在關鍵過場用動畫,日常互動保持快速反應。

無障礙設計不只是幫助視障者,也讓所有人更容易使用。大的觸控區域、清楚的文字、足夠的對比,這些都是基本功。

Flutter 這系列筆記就告一段落了。從架構規劃、資料庫設計、地圖整合、推播通知、執行流程到 UI/UX,算是把完整的 App 開發流程走過一遍。這個充電站專案還在進行中,後續有新的學習再來補充。
