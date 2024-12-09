---
layout: post
title: "Flutter 團隊協作與開發規範"
date: 2024-12-09 16:05:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Team, Best Practices]
---

隨著團隊擴大，開發規範變得很重要。這週整理 Flutter 專案的團隊協作經驗和最佳實踐。

## 程式碼規範

統一的程式碼風格讓協作更順暢。

### Linter 設定

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

linter:
  rules:
    # 額外規則
    always_declare_return_types: true
    always_put_required_named_parameters_first: true
    avoid_print: true
    avoid_relative_lib_imports: true
    prefer_const_constructors: true
    prefer_const_declarations: true
    prefer_final_fields: true
    prefer_final_locals: true
    require_trailing_commas: true
    sort_pub_dependencies: true
    
analyzer:
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
  errors:
    missing_required_param: error
    missing_return: error
```

### 命名規範

```dart
// ✅ 好的命名
class StationRepository { }            // 類別：PascalCase
final String stationName = '';         // 變數：camelCase
const int maxRetryCount = 3;           // 常數：camelCase
enum ChargingStatus { idle, charging } // Enum：PascalCase，值用 camelCase

// 私有成員加底線
class _StationCardState { }
final String _apiKey = '';

// Boolean 變數用 is/has/can 開頭
bool isCharging = false;
bool hasError = false;
bool canStart = true;

// ❌ 不好的命名
class station_repository { }      // 不要用 snake_case
final String StationName = '';    // 不要用 PascalCase
const int MAX_RETRY_COUNT = 3;    // 不要用 SCREAMING_CASE
```

### 檔案命名

```
✅ station_repository.dart
✅ charging_service.dart
✅ station_list_page.dart

❌ StationRepository.dart    (不要用 PascalCase)
❌ charging-service.dart     (不要用 kebab-case)
```

## Git 工作流程

### 分支策略

```
main          (生產環境，保護分支)
  ├─ develop  (開發環境)
  │   ├─ feature/station-search
  │   ├─ feature/charging-history
  │   └─ bugfix/map-crash
  └─ hotfix/payment-error
```

**規則：**
- `main`: 只接受來自 `develop` 或 `hotfix` 的 PR
- `develop`: 日常開發分支
- `feature/*`: 新功能
- `bugfix/*`: Bug 修復
- `hotfix/*`: 緊急修復，直接從 `main` 切出

### Commit Message 規範

```bash
# 格式：<type>(<scope>): <subject>

# Type:
feat:     新功能
fix:      Bug 修復
docs:     文件
style:    格式調整（不影響程式碼邏輯）
refactor: 重構
test:     測試
chore:    雜項（工具、設定等）

# 範例
git commit -m "feat(station): add search by location"
git commit -m "fix(charging): resolve payment timeout issue"
git commit -m "refactor(map): optimize marker clustering"
git commit -m "test(station): add unit tests for repository"
```

### Pull Request 流程

1. 建立 Feature Branch
2. 開發並提交
3. 發起 PR，填寫描述
4. 至少一位 Reviewer 審核
5. CI/CD 檢查通過
6. Merge 到 Develop

**PR 描述範本：**

```markdown
## 📝 變更說明
簡述這次 PR 的目的和內容

## 🔗 相關 Issue
Closes #123

## ✅ 測試項目
- [ ] Unit Tests 通過
- [ ] Widget Tests 通過
- [ ] 手動測試通過
- [ ] 無新增 Lint Warning

## 📸 截圖（如適用）
[截圖或錄影]

## 💡 備註
其他補充說明
```

## 專案結構規範

### 統一的目錄結構

```
lib/
├── core/                 # 核心共用
│   ├── constants/
│   ├── error/
│   ├── network/
│   ├── theme/
│   └── utils/
├── features/             # 功能模組
│   ├── station/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   └── charging/
├── config/               # 設定
│   ├── env.dart
│   └── routes.dart
├── injection.dart        # 依賴注入
└── main.dart

test/                     # 測試（對應 lib 結構）
├── core/
├── features/
│   ├── station/
│   └── charging/
└── widget_test.dart
```

### Import 順序

```dart
// 1. Dart SDK
import 'dart:async';
import 'dart:io';

// 2. Flutter SDK
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

// 3. 第三方套件（按字母排序）
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

// 4. 專案內部 import（按字母排序）
import 'package:charging_app/core/error/failures.dart';
import 'package:charging_app/core/utils/logger.dart';
import 'package:charging_app/features/station/domain/entities/station.dart';
```

## 程式碼審查 (Code Review)

### Reviewer 檢查清單

**功能性：**
- [ ] 程式碼符合需求
- [ ] 邏輯正確，無明顯 Bug
- [ ] 錯誤處理完整
- [ ] Edge Case 有考慮到

**程式碼品質：**
- [ ] 命名清楚易懂
- [ ] 沒有重複程式碼
- [ ] 函式不要太長（建議 < 50 行）
- [ ] 複雜邏輯有註解

**效能：**
- [ ] 沒有不必要的 rebuild
- [ ] ListView 使用 builder
- [ ] 圖片有快取

**測試：**
- [ ] 有對應的測試
- [ ] 測試覆蓋關鍵邏輯

### 給 PR 意見的技巧

```dart
// ❌ 不好的意見
// 這裡寫得很爛

// ✅ 具體且建設性的意見
// 建議：這個方法太長了（80 行），可以拆成幾個小方法，例如：
// - `_loadStationData()`
// - `_updateUI()`
// - `_handleError()`
// 這樣會更容易測試和維護。
```

**語氣要友善：**
- "建議可以..." 而不是 "你應該..."
- "這樣寫會不會更清楚？" 而不是 "這樣寫不對"
- 讚美好的程式碼："這個重構很棒！"

## 文件規範

### README.md

```markdown
# 充電站 App

## 專案簡介
提供電動車充電站搜尋、導航、充電服務

## 環境需求
- Flutter: 3.16.0+
- Dart: 3.2.0+
- Xcode: 15.0+ (iOS)
- Android Studio: 2023.1+ (Android)

## 安裝步驟
\`\`\`bash
# 1. Clone 專案
git clone https://github.com/company/charging-app.git

# 2. 安裝依賴
flutter pub get

# 3. 產生程式碼
flutter pub run build_runner build --delete-conflicting-outputs

# 4. 執行
flutter run
\`\`\`

## 環境設定
\`\`\`bash
# 開發環境
flutter run --dart-define=ENV=dev

# 正式環境
flutter build apk --dart-define=ENV=prod
\`\`\`

## 測試
\`\`\`bash
# Unit Tests
flutter test

# Integration Tests
flutter test integration_test/
\`\`\`

## 專案結構
見 [ARCHITECTURE.md](docs/ARCHITECTURE.md)

## 貢獻指南
見 [CONTRIBUTING.md](docs/CONTRIBUTING.md)
```

### 函式註解

重要或複雜的函式加上註解。

```dart
/// 搜尋充電站
///
/// 根據 [filter] 條件搜尋附近的充電站。
/// 
/// 搜尋邏輯：
/// 1. 先從本地快取讀取
/// 2. 如果快取過期或無資料，呼叫 API
/// 3. 更新本地快取
///
/// Returns:
///   - `Right(List<Station>)`: 成功取得充電站列表
///   - `Left(Failure)`: 發生錯誤
///
/// Example:
/// ```dart
/// final result = await repository.searchStations(
///   StationFilter(location: userLocation, radius: 5000),
/// );
/// ```
Future<Either<Failure, List<Station>>> searchStations(
  StationFilter filter,
) async {
  // ...
}
```

## 測試規範

### 測試檔案命名

```
lib/features/station/data/repositories/station_repository.dart
test/features/station/data/repositories/station_repository_test.dart

檔名要對應，加上 _test 後綴
```

### 測試結構

```dart
void main() {
  group('StationRepository', () {
    late StationRepository repository;
    late MockStationApi mockApi;
    
    setUp(() {
      mockApi = MockStationApi();
      repository = StationRepositoryImpl(mockApi);
    });
    
    group('searchStations', () {
      test('should return stations when API call succeeds', () async {
        // Arrange
        when(mockApi.getStations(any))
            .thenAnswer((_) async => [/* mock data */]);
        
        // Act
        final result = await repository.searchStations(filter);
        
        // Assert
        expect(result.isRight(), true);
        result.fold(
          (failure) => fail('Should not return failure'),
          (stations) => expect(stations.length, 2),
        );
      });
      
      test('should return NetworkFailure when API call fails', () async {
        // Arrange
        when(mockApi.getStations(any))
            .thenThrow(DioException(/* ... */));
        
        // Act
        final result = await repository.searchStations(filter);
        
        // Assert
        expect(result.isLeft(), true);
        result.fold(
          (failure) => expect(failure, isA<NetworkFailure>()),
          (stations) => fail('Should not return stations'),
        );
      });
    });
  });
}
```

## CI/CD 整合

### GitHub Actions

```yaml
# .github/workflows/flutter.yml
name: Flutter CI

on:
  push:
    branches: [ develop, main ]
  pull_request:
    branches: [ develop, main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
      
      - name: Install dependencies
        run: flutter pub get
      
      - name: Analyze
        run: flutter analyze
      
      - name: Run tests
        run: flutter test --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## 實務心得

好的規範需要團隊共識，不要強加個人喜好。

Linter 很有用，但不要過度限制（有些規則太嚴格）。

Code Review 是學習機會，保持開放心態。

文件要即時更新，過期的文件比沒有文件更糟。

自動化能做的就自動化（Lint、Test、Build），減少人為疏失。

定期回顧和調整規範，隨團隊成長而演進。

下週來聊 Flutter 的未來趨勢和新技術展望。
