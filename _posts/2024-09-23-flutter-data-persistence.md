---
layout: post
title: "Flutter 狀態持久化：SharedPreferences、Hive 與 Drift"
date: 2024-09-23 14:05:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Data Persistence, SharedPreferences, Hive, Drift]
---

這週研究資料持久化。App 需要儲存使用者偏好、快取資料、離線內容。Flutter 有幾種方案：**SharedPreferences**（簡單設定）、**Hive**（輕量 NoSQL）、**Drift**（SQL 資料庫）。

## SharedPreferences：簡單鍵值對

適合儲存簡單的設定：語言偏好、主題、是否顯示引導等。

### 安裝

```yaml
dependencies:
  shared_preferences: ^2.2.0
```

### 基本使用

```dart
import 'package:shared_preferences/shared_preferences.dart';

// 儲存
Future<void> saveSettings() async {
  final prefs = await SharedPreferences.getInstance();
  
  await prefs.setString('username', 'john_doe');
  await prefs.setInt('userId', 12345);
  await prefs.setBool('isDarkMode', true);
  await prefs.setDouble('lastLatitude', 25.0330);
  await prefs.setStringList('favoriteStations', ['station1', 'station2']);
}

// 讀取
Future<void> loadSettings() async {
  final prefs = await SharedPreferences.getInstance();
  
  final username = prefs.getString('username') ?? 'Guest';
  final userId = prefs.getInt('userId') ?? 0;
  final isDarkMode = prefs.getBool('isDarkMode') ?? false;
  final lastLatitude = prefs.getDouble('lastLatitude') ?? 0.0;
  final favorites = prefs.getStringList('favoriteStations') ?? [];
}

// 刪除
Future<void> removeSettings() async {
  final prefs = await SharedPreferences.getInstance();
  
  await prefs.remove('username');
  // 或清除全部
  await prefs.clear();
}
```

### 封裝 Settings Service

```dart
class SettingsService {
  static const _keyLanguage = 'language';
  static const _keyThemeMode = 'theme_mode';
  static const _keyNotificationsEnabled = 'notifications_enabled';
  
  Future<String> getLanguage() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getString(_keyLanguage) ?? 'zh_TW';
  }
  
  Future<void> setLanguage(String language) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_keyLanguage, language);
  }
  
  Future<ThemeMode> getThemeMode() async {
    final prefs = await SharedPreferences.getInstance();
    final mode = prefs.getString(_keyThemeMode) ?? 'system';
    
    return switch (mode) {
      'light' => ThemeMode.light,
      'dark' => ThemeMode.dark,
      _ => ThemeMode.system,
    };
  }
  
  Future<void> setThemeMode(ThemeMode mode) async {
    final prefs = await SharedPreferences.getInstance();
    final modeString = switch (mode) {
      ThemeMode.light => 'light',
      ThemeMode.dark => 'dark',
      ThemeMode.system => 'system',
    };
    await prefs.setString(_keyThemeMode, modeString);
  }
}
```

### 與 Riverpod 整合

```dart
// providers/settings_provider.dart
final settingsServiceProvider = Provider((ref) => SettingsService());

final languageProvider = StateNotifierProvider<LanguageNotifier, String>((ref) {
  return LanguageNotifier(ref.read(settingsServiceProvider));
});

class LanguageNotifier extends StateNotifier<String> {
  final SettingsService _settingsService;
  
  LanguageNotifier(this._settingsService) : super('zh_TW') {
    _loadLanguage();
  }
  
  Future<void> _loadLanguage() async {
    state = await _settingsService.getLanguage();
  }
  
  Future<void> setLanguage(String language) async {
    await _settingsService.setLanguage(language);
    state = language;
  }
}
```

## Hive：輕量 NoSQL 資料庫

適合儲存結構化資料：收藏的充電站、充電歷史紀錄等。

### 特點

- **快速**：比 SQLite 快很多
- **輕量**：純 Dart 實作
- **Type-safe**：強型別
- **加密**：支援加密儲存

### 安裝

```yaml
dependencies:
  hive: ^2.2.3
  hive_flutter: ^1.1.0

dev_dependencies:
  hive_generator: ^2.0.0
  build_runner: ^2.4.0
```

### 定義 Model

```dart
// models/favorite_station.dart
import 'package:hive/hive.dart';

part 'favorite_station.g.dart';

@HiveType(typeId: 0)
class FavoriteStation extends HiveObject {
  @HiveField(0)
  String id;
  
  @HiveField(1)
  String name;
  
  @HiveField(2)
  double latitude;
  
  @HiveField(3)
  double longitude;
  
  @HiveField(4)
  DateTime addedAt;
  
  FavoriteStation({
    required this.id,
    required this.name,
    required this.latitude,
    required this.longitude,
    required this.addedAt,
  });
}
```

產生程式碼：

```bash
flutter pub run build_runner build
```

### 初始化

```dart
import 'package:hive_flutter/hive_flutter.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // 初始化 Hive
  await Hive.initFlutter();
  
  // 註冊 Adapter
  Hive.registerAdapter(FavoriteStationAdapter());
  
  // 開啟 Box
  await Hive.openBox<FavoriteStation>('favorites');
  
  runApp(MyApp());
}
```

### CRUD 操作

```dart
class FavoriteRepository {
  static const _boxName = 'favorites';
  
  Box<FavoriteStation> get _box => Hive.box<FavoriteStation>(_boxName);
  
  // 新增
  Future<void> add(FavoriteStation station) async {
    await _box.put(station.id, station);
  }
  
  // 讀取全部
  List<FavoriteStation> getAll() {
    return _box.values.toList()
      ..sort((a, b) => b.addedAt.compareTo(a.addedAt));
  }
  
  // 讀取單筆
  FavoriteStation? get(String id) {
    return _box.get(id);
  }
  
  // 檢查是否存在
  bool contains(String id) {
    return _box.containsKey(id);
  }
  
  // 刪除
  Future<void> remove(String id) async {
    await _box.delete(id);
  }
  
  // 清空
  Future<void> clear() async {
    await _box.clear();
  }
  
  // 監聽變化
  Stream<BoxEvent> watch() {
    return _box.watch();
  }
}
```

### 與 Riverpod 整合

```dart
final favoriteRepositoryProvider = Provider((ref) => FavoriteRepository());

final favoritesProvider = StreamProvider<List<FavoriteStation>>((ref) {
  final repository = ref.read(favoriteRepositoryProvider);
  
  // 初始資料
  final initialData = repository.getAll();
  
  // 監聽變化
  return repository.watch().map((_) => repository.getAll())
      .startWith(initialData);
});

// 使用
class FavoriteButton extends ConsumerWidget {
  final ChargingStation station;
  
  const FavoriteButton({required this.station});
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final repository = ref.read(favoriteRepositoryProvider);
    final isFavorite = repository.contains(station.id);
    
    return IconButton(
      icon: Icon(isFavorite ? Icons.favorite : Icons.favorite_border),
      onPressed: () async {
        if (isFavorite) {
          await repository.remove(station.id);
        } else {
          await repository.add(FavoriteStation(
            id: station.id,
            name: station.name,
            latitude: station.location.latitude,
            longitude: station.location.longitude,
            addedAt: DateTime.now(),
          ));
        }
      },
    );
  }
}
```

### 加密

```dart
import 'package:hive/hive.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

Future<void> openEncryptedBox() async {
  const secureStorage = FlutterSecureStorage();
  
  // 取得或產生加密金鑰
  var encryptionKey = await secureStorage.read(key: 'hive_key');
  if (encryptionKey == null) {
    final key = Hive.generateSecureKey();
    encryptionKey = base64UrlEncode(key);
    await secureStorage.write(key: 'hive_key', value: encryptionKey);
  }
  
  final key = base64Url.decode(encryptionKey);
  
  // 開啟加密的 Box
  await Hive.openBox<FavoriteStation>(
    'favorites',
    encryptionCipher: HiveAesCipher(key),
  );
}
```

## Drift：SQL 資料庫

適合複雜查詢和關聯資料。Drift 是 Flutter 的 SQL ORM（以前叫 Moor）。

### 安裝

```yaml
dependencies:
  drift: ^2.13.0
  sqlite3_flutter_libs: ^0.5.0
  path_provider: ^2.1.0
  path: ^1.8.3

dev_dependencies:
  drift_dev: ^2.13.0
  build_runner: ^2.4.0
```

### 定義 Table

```dart
// database/tables.dart
import 'package:drift/drift.dart';

class ChargingStations extends Table {
  TextColumn get id => text()();
  TextColumn get name => text()();
  TextColumn get address => text()();
  RealColumn get latitude => real()();
  RealColumn get longitude => real()();
  IntColumn get totalChargers => integer()();
  IntColumn get availableChargers => integer()();
  DateTimeColumn get lastUpdated => dateTime()();
  
  @override
  Set<Column> get primaryKey => {id};
}

class ChargingSessions extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get stationId => text().references(ChargingStations, #id)();
  TextColumn get chargerId => text()();
  DateTimeColumn get startTime => dateTime()();
  DateTimeColumn get endTime => dateTime().nullable()();
  RealColumn get energyKwh => real().nullable()();
  IntColumn get costNtd => integer().nullable()();
  TextColumn get status => text()();  // 'charging', 'completed', 'cancelled'
}
```

### 定義 Database

```dart
// database/app_database.dart
import 'dart:io';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as p;

part 'app_database.g.dart';

@DriftDatabase(tables: [ChargingStations, ChargingSessions])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());
  
  @override
  int get schemaVersion => 1;
  
  // 查詢附近充電站
  Future<List<ChargingStation>> getNearbyStations(
    double lat,
    double lng,
    double radiusKm,
  ) async {
    // 簡化版（實際應該用 Haversine formula）
    final latDelta = radiusKm / 111.0;
    final lngDelta = radiusKm / (111.0 * cos(lat * pi / 180));
    
    return (select(chargingStations)
      ..where((s) =>
          s.latitude.isBetweenValues(lat - latDelta, lat + latDelta) &
          s.longitude.isBetweenValues(lng - lngDelta, lng + lngDelta)))
        .get();
  }
  
  // 查詢充電歷史（分頁）
  Future<List<ChargingSession>> getChargingSessions({
    int limit = 20,
    int offset = 0,
  }) async {
    return (select(chargingSessions)
      ..orderBy([(s) => OrderingTerm.desc(s.startTime)])
      ..limit(limit, offset: offset))
        .get();
  }
  
  // 查詢特定充電站的歷史
  Future<List<ChargingSession>> getSessionsByStation(String stationId) async {
    return (select(chargingSessions)
      ..where((s) => s.stationId.equals(stationId))
      ..orderBy([(s) => OrderingTerm.desc(s.startTime)]))
        .get();
  }
  
  // 統計總充電次數和電量
  Future<SessionStats> getSessionStats() async {
    final count = chargingSessions.id.count();
    final totalEnergy = chargingSessions.energyKwh.sum();
    final totalCost = chargingSessions.costNtd.sum();
    
    final query = selectOnly(chargingSessions)
      ..addColumns([count, totalEnergy, totalCost])
      ..where(chargingSessions.status.equals('completed'));
    
    final result = await query.getSingle();
    
    return SessionStats(
      count: result.read(count) ?? 0,
      totalEnergyKwh: result.read(totalEnergy) ?? 0.0,
      totalCostNtd: result.read(totalCost) ?? 0,
    );
  }
  
  // 新增充電站
  Future<void> insertStation(ChargingStationsCompanion station) async {
    await into(chargingStations).insert(
      station,
      mode: InsertMode.replace,
    );
  }
  
  // 批次新增
  Future<void> insertStations(List<ChargingStationsCompanion> stations) async {
    await batch((batch) {
      batch.insertAll(
        chargingStations,
        stations,
        mode: InsertMode.replace,
      );
    });
  }
  
  // 更新充電站可用充電樁數量
  Future<void> updateAvailableChargers(String stationId, int available) async {
    await (update(chargingStations)..where((s) => s.id.equals(stationId)))
        .write(ChargingStationsCompanion(
          availableChargers: Value(available),
          lastUpdated: Value(DateTime.now()),
        ));
  }
  
  // 開始充電
  Future<int> startCharging(ChargingSessionsCompanion session) async {
    return await into(chargingSessions).insert(session);
  }
  
  // 結束充電
  Future<void> endCharging(
    int sessionId,
    double energyKwh,
    int costNtd,
  ) async {
    await (update(chargingSessions)..where((s) => s.id.equals(sessionId)))
        .write(ChargingSessionsCompanion(
          endTime: Value(DateTime.now()),
          energyKwh: Value(energyKwh),
          costNtd: Value(costNtd),
          status: Value('completed'),
        ));
  }
}

class SessionStats {
  final int count;
  final double totalEnergyKwh;
  final int totalCostNtd;
  
  SessionStats({
    required this.count,
    required this.totalEnergyKwh,
    required this.totalCostNtd,
  });
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final dbFolder = await getApplicationDocumentsDirectory();
    final file = File(p.join(dbFolder.path, 'charging_app.db'));
    return NativeDatabase(file);
  });
}
```

產生程式碼：

```bash
flutter pub run build_runner build
```

### 使用 Database

```dart
final databaseProvider = Provider((ref) => AppDatabase());

final nearbyStationsProvider = FutureProvider.family<
  List<ChargingStation>,
  (double, double, double)
>((ref, params) async {
  final database = ref.read(databaseProvider);
  return database.getNearbyStations(params.$1, params.$2, params.$3);
});
```

### Migration

資料庫結構變更時要做 migration：

```dart
@DriftDatabase(tables: [ChargingStations, ChargingSessions])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());
  
  @override
  int get schemaVersion => 2;  // 版本提升
  
  @override
  MigrationStrategy get migration {
    return MigrationStrategy(
      onCreate: (Migrator m) async {
        await m.createAll();
      },
      onUpgrade: (Migrator m, int from, int to) async {
        if (from < 2) {
          // 新增欄位
          await m.addColumn(chargingStations, chargingStations.address);
        }
      },
    );
  }
}
```

## 選擇建議

| 需求 | 推薦方案 |
|------|---------|
| 簡單設定（語言、主題等） | SharedPreferences |
| 少量結構化資料（收藏、快取） | Hive |
| 複雜查詢、關聯資料 | Drift |
| 需要全文搜尋 | Drift + FTS5 |
| 需要加密 | Hive (encrypted) 或 Drift |

## 實務心得

資料持久化要根據需求選擇工具，不要過度設計。

SharedPreferences 最簡單，能用就用。但不要存大量資料，效能會差。

Hive 很快，適合大部分場景。Type-safe 很好用，不容易出錯。

Drift 適合複雜應用，但學習曲線陡。如果需要 JOIN、GROUP BY 等 SQL 功能再用。

記得做資料備份和還原功能，使用者換機時很重要。

加密敏感資料（密碼、Token等），用 `flutter_secure_storage` 或 Hive 的加密功能。

下週來研究 Flutter 的深層連結（Deep Link）和 App Link，讓 App 可以從網頁或其他 App 開啟特定頁面。
