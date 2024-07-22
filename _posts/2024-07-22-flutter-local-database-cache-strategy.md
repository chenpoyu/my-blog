---
layout: post
title: "Flutter 本地資料庫設計：冷熱資料分離策略"
date: 2024-07-22 10:10:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, SQLite, 快取策略, 資料庫設計]
---

這週主要在處理本地資料庫的設計。這個充電站 App 有個特殊需求：盡量減少 API 呼叫，因為使用者可能在訊號不好的地方（停車場、地下室），而且頻繁呼叫 API 會影響體驗和流量。

所以要設計一套本地快取機制，區分「冷資料」和「熱資料」，避免過度依賴網路。

## 冷資料 vs 熱資料

先釐清什麼是冷資料、熱資料：

**冷資料（Cold Data）**：不常變動的資料，可以長時間快取。

範例：
- 充電站基本資訊（位置、名稱、營業時間）
- 充電站設備規格（插頭型號、最大功率）
- 區域資料（縣市、行政區）
- 收費方案

這些資料可能幾個月才更新一次，不需要每次都從 API 抓。

**熱資料（Hot Data）**：經常變動的資料，需要較頻繁更新。

範例：
- 充電站即時狀態（可用/使用中/故障）
- 充電樁剩餘數量
- 排隊人數
- 即時電價（可能隨尖離峰變動）

這些資料變化快，但也不能每秒都更新，要設計合理的更新頻率。

**使用者資料（User Data）**：使用者操作產生的資料。

範例：
- 收藏的充電站
- 充電歷史紀錄
- 偏好設定

這些資料要永久保存在本地，也要同步到雲端。

## 資料庫選擇

Flutter 常用的本地資料庫：

**sqflite**：SQLite 封裝，關聯式資料庫，適合結構化資料。

**Hive**：NoSQL，Key-Value 儲存，效能極快，適合簡單資料。

**Isar**：新的 NoSQL，效能比 Hive 更好，但生態較新。

**shared_preferences**：只能存簡單的 key-value，適合設定值。

專案選用 **sqflite**，因為：
- 資料結構複雜，有關聯性（充電站、充電樁、評論）
- 需要複雜查詢（附近的站、篩選條件）
- 團隊熟悉 SQL

## 資料庫結構

設計幾張主要的表：

### 充電站表（stations）

```sql
CREATE TABLE stations (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  address TEXT,
  latitude REAL NOT NULL,
  longitude REAL NOT NULL,
  operator TEXT,
  phone TEXT,
  opening_hours TEXT,
  amenities TEXT,  -- JSON 格式（有無廁所、餐廳等）
  created_at INTEGER,
  updated_at INTEGER,
  last_sync_at INTEGER  -- 最後同步時間
);
```

### 充電樁表（chargers）

```sql
CREATE TABLE chargers (
  id TEXT PRIMARY KEY,
  station_id TEXT NOT NULL,
  connector_type TEXT NOT NULL,  -- CCS1, CCS2, CHAdeMO, Type2
  max_power INTEGER,  -- kW
  status TEXT,  -- available, in_use, offline, maintenance
  price_per_kwh REAL,
  last_sync_at INTEGER,
  FOREIGN KEY (station_id) REFERENCES stations(id)
);
```

### 快取狀態表（cache_status）

記錄每種資料的快取狀態，決定是否要更新。

```sql
CREATE TABLE cache_status (
  cache_key TEXT PRIMARY KEY,
  last_updated INTEGER,  -- 最後更新時間
  ttl INTEGER,  -- 快取存活時間（秒）
  data_type TEXT  -- cold, hot, user
);
```

### 收藏表（favorites）

```sql
CREATE TABLE favorites (
  user_id TEXT,
  station_id TEXT,
  created_at INTEGER,
  PRIMARY KEY (user_id, station_id)
);
```

### 充電紀錄表（charging_sessions）

```sql
CREATE TABLE charging_sessions (
  id TEXT PRIMARY KEY,
  station_id TEXT,
  charger_id TEXT,
  user_id TEXT,
  start_time INTEGER,
  end_time INTEGER,
  energy_delivered REAL,  -- kWh
  total_cost REAL,
  status TEXT,  -- active, completed, cancelled
  synced INTEGER DEFAULT 0,  -- 是否已同步到雲端
  created_at INTEGER
);
```

## 快取策略

不同類型的資料，用不同的快取策略。

### 冷資料策略：Cache-First

```dart
// data/repositories/charging_station_repository_impl.dart
@override
Future<Either<Failure, List<ChargingStation>>> getNearbyStations(
  LatLng location,
  double radius,
) async {
  try {
    // 1. 先檢查本地快取
    final cached = await localDataSource.getCachedStations(location, radius);
    final cacheStatus = await localDataSource.getCacheStatus('nearby_stations');
    
    // 2. 判斷快取是否有效（24 小時內）
    if (cached.isNotEmpty && _isCacheValid(cacheStatus, Duration(hours: 24))) {
      return Right(cached.map((m) => m.toEntity()).toList());
    }
    
    // 3. 快取無效或不存在，從 API 取得
    try {
      final remote = await remoteDataSource.fetchNearbyStations(
        location.latitude,
        location.longitude,
        radius,
      );
      
      // 4. 存入本地快取
      await localDataSource.cacheStations(remote);
      await localDataSource.updateCacheStatus('nearby_stations');
      
      return Right(remote.map((m) => m.toEntity()).toList());
    } catch (e) {
      // 5. API 失敗，如果有舊快取就用（雖然過期了）
      if (cached.isNotEmpty) {
        return Right(cached.map((m) => m.toEntity()).toList());
      }
      throw e;
    }
  } catch (e) {
    return Left(CacheFailure(e.toString()));
  }
}

bool _isCacheValid(CacheStatus? status, Duration ttl) {
  if (status == null) return false;
  final now = DateTime.now().millisecondsSinceEpoch;
  return (now - status.lastUpdated) < ttl.inMilliseconds;
}
```

流程：
1. 檢查本地是否有資料
2. 檢查快取是否在有效期內（24 小時）
3. 有效 → 直接用本地資料
4. 無效 → 呼叫 API，更新快取
5. API 失敗 → 如果有舊快取，繼續用（降級策略）

### 熱資料策略：Stale-While-Revalidate

```dart
@override
Future<Either<Failure, List<ChargerStatus>>> getChargerStatus(
  String stationId,
) async {
  try {
    // 1. 立即返回快取資料（即使可能過期）
    final cached = await localDataSource.getChargerStatus(stationId);
    
    // 2. 背景更新
    _updateChargerStatusInBackground(stationId);
    
    if (cached.isNotEmpty) {
      return Right(cached.map((m) => m.toEntity()).toList());
    }
    
    // 3. 如果沒有快取，等待 API
    final remote = await remoteDataSource.fetchChargerStatus(stationId);
    await localDataSource.cacheChargerStatus(remote);
    
    return Right(remote.map((m) => m.toEntity()).toList());
  } catch (e) {
    return Left(ServerFailure(e.toString()));
  }
}

void _updateChargerStatusInBackground(String stationId) async {
  final cacheStatus = await localDataSource.getCacheStatus('charger_$stationId');
  
  // 5 分鐘內更新過就不用再更新
  if (_isCacheValid(cacheStatus, Duration(minutes: 5))) {
    return;
  }
  
  try {
    final remote = await remoteDataSource.fetchChargerStatus(stationId);
    await localDataSource.cacheChargerStatus(remote);
    await localDataSource.updateCacheStatus('charger_$stationId');
  } catch (e) {
    // 背景更新失敗不影響使用者
    print('Background update failed: $e');
  }
}
```

流程：
1. 先返回快取資料（快速回應）
2. 同時在背景更新資料
3. 下次呼叫會拿到新資料

這樣使用者體驗好（立即看到資料），資料也不會太舊。

### 使用者資料策略：Local-First with Sync

```dart
@override
Future<Either<Failure, void>> addFavorite(String stationId) async {
  try {
    final userId = await _getCurrentUserId();
    
    // 1. 立即存到本地
    await localDataSource.addFavorite(userId, stationId);
    
    // 2. 標記為未同步
    await localDataSource.markAsUnsynced('favorite', stationId);
    
    // 3. 嘗試同步到雲端（不阻塞）
    _syncFavoritesInBackground();
    
    return Right(null);
  } catch (e) {
    return Left(CacheFailure(e.toString()));
  }
}

void _syncFavoritesInBackground() async {
  final unsynced = await localDataSource.getUnsyncedFavorites();
  
  for (final favorite in unsynced) {
    try {
      await remoteDataSource.syncFavorite(favorite);
      await localDataSource.markAsSynced('favorite', favorite.id);
    } catch (e) {
      // 同步失敗，下次再試
      print('Sync failed: $e');
    }
  }
}
```

特點：
- 立即回應，不等雲端
- 離線也能操作
- 背景自動同步
- 失敗會重試

## 避免重複呼叫 API

有個問題：如果使用者快速切換頁面，可能觸發多次 API 呼叫。要避免重複請求。

### 請求去重

```dart
class ApiRequestDeduplicator {
  final Map<String, Future<dynamic>> _pendingRequests = {};
  
  Future<T> deduplicate<T>(
    String key,
    Future<T> Function() request,
  ) async {
    // 如果已有相同請求在進行中，等待結果
    if (_pendingRequests.containsKey(key)) {
      return await _pendingRequests[key] as T;
    }
    
    // 發起新請求
    final future = request();
    _pendingRequests[key] = future;
    
    try {
      final result = await future;
      return result;
    } finally {
      _pendingRequests.remove(key);
    }
  }
}

// 使用
final deduplicator = ApiRequestDeduplicator();

Future<List<ChargingStationModel>> fetchNearbyStations(
  double lat,
  double lng,
  double radius,
) async {
  final key = 'nearby_$lat\_$lng\_$radius';
  
  return await deduplicator.deduplicate(
    key,
    () => _actualApiFetch(lat, lng, radius),
  );
}
```

同一個請求在處理中，後續的呼叫會等待同一個結果，不會重複發送。

## 錯誤資料的處理

有時候 API 回傳錯誤資料（null、空陣列、格式不對），不應該覆蓋正常的快取。

```dart
Future<Either<Failure, List<ChargingStation>>> getNearbyStations(
  LatLng location,
  double radius,
) async {
  final cached = await localDataSource.getCachedStations(location, radius);
  
  try {
    final remote = await remoteDataSource.fetchNearbyStations(...);
    
    // 驗證 API 資料
    if (remote.isEmpty) {
      // API 回傳空，可能是錯誤，保留快取
      if (cached.isNotEmpty) {
        return Right(cached.map((m) => m.toEntity()).toList());
      }
    }
    
    // 檢查資料合理性
    if (_isDataValid(remote)) {
      await localDataSource.cacheStations(remote);
      return Right(remote.map((m) => m.toEntity()).toList());
    } else {
      // 資料異常，用快取
      return Right(cached.map((m) => m.toEntity()).toList());
    }
  } catch (e) {
    // API 失敗，用快取
    if (cached.isNotEmpty) {
      return Right(cached.map((m) => m.toEntity()).toList());
    }
    return Left(ServerFailure(e.toString()));
  }
}

bool _isDataValid(List<ChargingStationModel> data) {
  // 基本檢查
  if (data.isEmpty) return false;
  
  // 檢查必要欄位
  for (final station in data) {
    if (station.latitude < -90 || station.latitude > 90) return false;
    if (station.longitude < -180 || station.longitude > 180) return false;
    if (station.name.isEmpty) return false;
  }
  
  return true;
}
```

## 資料庫初始化

```dart
// data/datasources/local/charging_database.dart
class ChargingDatabase {
  static Database? _database;
  
  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
  
  Future<Database> _initDatabase() async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, 'charging_app.db');
    
    return await openDatabase(
      path,
      version: 1,
      onCreate: _onCreate,
      onUpgrade: _onUpgrade,
    );
  }
  
  Future<void> _onCreate(Database db, int version) async {
    await db.execute('''
      CREATE TABLE stations (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        latitude REAL NOT NULL,
        longitude REAL NOT NULL,
        address TEXT,
        operator TEXT,
        last_sync_at INTEGER
      )
    ''');
    
    await db.execute('''
      CREATE TABLE chargers (
        id TEXT PRIMARY KEY,
        station_id TEXT NOT NULL,
        connector_type TEXT NOT NULL,
        max_power INTEGER,
        status TEXT,
        price_per_kwh REAL,
        last_sync_at INTEGER,
        FOREIGN KEY (station_id) REFERENCES stations(id)
      )
    ''');
    
    await db.execute('''
      CREATE TABLE cache_status (
        cache_key TEXT PRIMARY KEY,
        last_updated INTEGER,
        ttl INTEGER,
        data_type TEXT
      )
    ''');
    
    // 建立索引加速查詢
    await db.execute('CREATE INDEX idx_station_location ON stations(latitude, longitude)');
  }
  
  Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
    // 資料庫升級邏輯
    if (oldVersion < 2) {
      // 例如：新增欄位
      await db.execute('ALTER TABLE stations ADD COLUMN phone TEXT');
    }
  }
}
```

## 定期清理

本地資料會越來越多，要定期清理。

```dart
Future<void> cleanUpOldCache() async {
  final db = await database;
  final now = DateTime.now().millisecondsSinceEpoch;
  
  // 刪除 30 天前的快取
  final threshold = now - Duration(days: 30).inMilliseconds;
  
  await db.delete(
    'stations',
    where: 'last_sync_at < ?',
    whereArgs: [threshold],
  );
  
  await db.delete(
    'chargers',
    where: 'last_sync_at < ?',
    whereArgs: [threshold],
  );
  
  // 刪除已同步的舊充電紀錄（保留 90 天）
  final recordThreshold = now - Duration(days: 90).inMilliseconds;
  await db.delete(
    'charging_sessions',
    where: 'synced = 1 AND end_time < ?',
    whereArgs: [recordThreshold],
  );
}
```

在 App 啟動時或背景執行。

## 實務心得

本地快取的設計比想像中複雜，要考慮：
- 資料的時效性（多久更新一次）
- 網路失敗的降級策略
- 錯誤資料的過濾
- 請求去重
- 定期清理

但設計好後，使用者體驗會好很多。打開 App 立即看到資料，不用等 API。訊號不好也能繼續用（雖然資料可能稍舊）。

下週要來處理 Google Map 的整合，在地圖上顯示充電站，這應該是整個 App 的核心畫面。
