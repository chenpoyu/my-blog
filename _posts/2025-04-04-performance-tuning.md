---
layout: post
title: "效能調校的藝術：從症狀找到病因"
date: 2025-04-04 10:25:00 +0800
categories: [效能優化]
tags: [效能調校, .NET Core, SQL, 資料庫優化]
---

## 突如其來的效能問題

這週測試團隊回報：「活動頁面載入很慢,要等 3-4 秒。」

我自己測試了一下，確實很慢。但奇怪的是，上週還好好的，怎麼突然變慢？

打開 Application Insights 看 trace，發現 API 的回應時間從平常的 50ms 暴增到 3500ms。

「一定有什麼地方出問題了。」

## 不要憑感覺，要看數據

很多人遇到效能問題，第一反應是：「我覺得應該是 XXX 的問題。」

然後就開始改 code，加快取、改演算法、優化 SQL⋯⋯

這是最沒效率的做法。

**效能調校的第一步是：找出 bottleneck（瓶頸）**。

不是「我覺得」，而是「數據顯示」。

### 用 Application Insights 找瓶頸

Application Insights 的 Performance 頁面可以看到每個 API 的回應時間分佈：

```
GET /api/activities
├─ 執行時間: 3500ms
├─ Dependency calls:
│   ├─ SQL Database: 3200ms ⚠️
│   ├─ Redis: 15ms
│   └─ gRPC (Member): 8ms
```

一看就知道：**問題在資料庫查詢**。

3200ms 的資料庫查詢，太誇張了。

### 找出慢的 SQL

Application Insights 會記錄每個 SQL query，包括執行時間：

```sql
-- 執行時間: 3200ms
SELECT a.*, 
       COUNT(p.Id) as ParticipantCount
FROM Activity.Activities a
LEFT JOIN Activity.Participants p ON a.Id = p.ActivityId
WHERE a.IsActive = 1 
  AND a.StartDate >= GETDATE()
GROUP BY a.Id, a.Title, a.Description, ...
ORDER BY a.CreatedAt DESC
```

找到了！就是這個查詢。

## 分析 SQL 的問題

把這個 SQL 拿到 SQL Server Management Studio 跑一次，看 execution plan：

```
Hash Match (Aggregate)  Cost: 85%
└─ Hash Match (Left Outer Join)  Cost: 12%
    ├─ Table Scan: Activities  Cost: 2%  (Rows: 5000)
    └─ Table Scan: Participants  Cost: 1%  (Rows: 150000)
```

問題找到了：
1. **Table Scan**：Activities 和 Participants 都是全表掃描
2. **Hash Join**：因為沒有索引,只能用 hash join（很慢）
3. **參與者數量太多**：15 萬筆資料要全部掃一遍

### 解決方法 1：加索引

最明顯的問題是沒有索引。

```sql
-- 在 Activities 加索引
CREATE INDEX IX_Activities_IsActive_StartDate 
ON Activity.Activities (IsActive, StartDate) 
INCLUDE (Title, Description, CreatedAt);

-- 在 Participants 加索引
CREATE INDEX IX_Participants_ActivityId 
ON Activity.Participants (ActivityId);
```

加完索引後，執行時間從 3200ms 降到 180ms。

Execution plan 變成：

```
Hash Match (Aggregate)  Cost: 60%
└─ Nested Loops (Left Outer Join)  Cost: 35%
    ├─ Index Seek: Activities  Cost: 3%  (Rows: 50)
    └─ Index Seek: Participants  Cost: 2%  (Rows: 3000)
```

現在用 Index Seek,不是 Table Scan。而且因為有 WHERE 條件，只掃 50 筆活動，不是全部 5000 筆。

### 解決方法 2：改查詢邏輯

180ms 還是有點慢。繼續優化。

問題在於 `COUNT(p.Id)`。每個活動都要算參與人數，即使很多活動的參與人數是 0。

改成分兩次查詢：

```csharp
// 第一次查詢：取得活動列表
var activities = await _dbContext.Activities
    .Where(a => a.IsActive && a.StartDate >= DateTime.UtcNow)
    .OrderByDescending(a => a.CreatedAt)
    .Take(20)  // 分頁，一次只取 20 筆
    .ToListAsync();

var activityIds = activities.Select(a => a.Id).ToList();

// 第二次查詢：取得參與人數（只查這 20 個活動的）
var participantCounts = await _dbContext.Participants
    .Where(p => activityIds.Contains(p.ActivityId))
    .GroupBy(p => p.ActivityId)
    .Select(g => new { ActivityId = g.Key, Count = g.Count() })
    .ToDictionaryAsync(x => x.ActivityId, x => x.Count);

// 組合結果
var result = activities.Select(a => new ActivityDto
{
    Id = a.Id,
    Title = a.Title,
    ParticipantCount = participantCounts.GetValueOrDefault(a.Id, 0)
}).ToList();
```

執行時間從 180ms 降到 35ms。

為什麼？
- 第一次查詢只取 20 筆活動：15ms
- 第二次查詢只算這 20 個活動的參與人數：18ms
- 記憶體裡組合：2ms

比原本的一個大 JOIN 快多了。

## N+1 Query Problem

改完活動頁面，測試另一個頁面：會員的訂單列表。

這個頁面更慢：**5 秒**。

看 Application Insights：

```
GET /api/orders?memberId=M001
├─ 執行時間: 5000ms
├─ Dependency calls:
│   ├─ SQL: SELECT * FROM Orders WHERE MemberId = 'M001'  (20ms)
│   ├─ SQL: SELECT * FROM Products WHERE Id = 'P001'  (8ms)
│   ├─ SQL: SELECT * FROM Products WHERE Id = 'P002'  (8ms)
│   ├─ SQL: SELECT * FROM Products WHERE Id = 'P003'  (8ms)
│   ... (重複 500 次)
```

**N+1 Query Problem**！

查詢訂單列表：1 個 query
查詢每個訂單的產品：N 個 query

如果有 500 個訂單，就會執行 501 個 query。

### 解決方法：Eager Loading

用 Entity Framework 的 `Include` 一次查詢：

```csharp
// ❌ 不好 - N+1 queries
var orders = await _dbContext.Orders
    .Where(o => o.MemberId == memberId)
    .ToListAsync();

foreach (var order in orders)
{
    var product = await _dbContext.Products.FindAsync(order.ProductId);
    order.Product = product;
}

// ✅ 好 - 1 query
var orders = await _dbContext.Orders
    .Include(o => o.Product)  // Eager loading
    .Where(o => o.MemberId == memberId)
    .ToListAsync();
```

執行時間從 5000ms 降到 50ms。

## 快取的時機

有些查詢優化後還是慢，就要考慮快取。

但**不是所有東西都該快取**（上次寫過了）。

快取的判斷標準：
1. **讀取頻率高**：每分鐘被查詢好幾次
2. **變動頻率低**：資料很少改變
3. **計算成本高**：查詢或計算很耗時

比如「熱門活動」這個查詢：

```csharp
public async Task<List<ActivityDto>> GetPopularActivitiesAsync()
{
    var cacheKey = "activities:popular";
    var cached = await _cache.GetStringAsync(cacheKey);
    
    if (cached != null)
    {
        return JsonSerializer.Deserialize<List<ActivityDto>>(cached);
    }
    
    // 複雜的查詢：計算參與人數、評分、熱度
    var activities = await _dbContext.Activities
        .Where(a => a.IsActive)
        .Select(a => new
        {
            Activity = a,
            ParticipantCount = _dbContext.Participants
                .Count(p => p.ActivityId == a.Id),
            AverageRating = _dbContext.Ratings
                .Where(r => r.ActivityId == a.Id)
                .Average(r => (double?)r.Score) ?? 0
        })
        .OrderByDescending(x => x.ParticipantCount * x.AverageRating)
        .Take(10)
        .ToListAsync();
    
    var dtos = activities.Select(x => MapToDto(x.Activity, x.ParticipantCount)).ToList();
    
    // 快取 10 分鐘
    await _cache.SetStringAsync(
        cacheKey,
        JsonSerializer.Serialize(dtos),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        });
    
    return dtos;
}
```

這個查詢：
- 要 JOIN 三個表
- 要計算統計數字
- 排序很複雜
- 但結果每 10 分鐘才更新一次

非常適合快取。

## 資料庫連線池的設定

有一次壓測時發現奇怪的現象：
- 前 5 分鐘：API 回應時間正常（50ms）
- 5 分鐘後：突然暴增到 2000ms
- 10 分鐘後：部分請求 timeout

看 log 發現：`System.InvalidOperationException: Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool.`

**連線池耗盡了**！

預設的連線池大小太小：

```json
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=xxx;Database=xxx;User Id=xxx;Password=xxx;Max Pool Size=100;"
  }
}
```

預設是 100 個連線。但我們的 App Service 開了 3 個 instance，壓測時有 1000 個併發使用者。

平均每個 instance 要處理 333 個併發請求，但只有 100 個資料庫連線，當然會卡住。

調整連線池大小：

```json
"DefaultConnection": "Server=xxx;Database=xxx;User Id=xxx;Password=xxx;Max Pool Size=500;Min Pool Size=10;"
```

問題解決。

但這也提醒我們：**資料庫連線是有限資源，要小心管理**。

特別要注意：
- 用完要立刻 `Dispose`
- 不要在 loop 裡開連線
- 使用 `async/await` 避免 blocking

## 效能調校的心得

這幾週優化效能，學到幾件事：

### 1. 先量測，再優化

不要憑感覺猜測哪裡慢。用工具（Application Insights、SQL Profiler）找出真正的瓶頸。

### 2. 優先優化低垂的果實

有些優化很簡單（加個索引、改個 query），效果卻很大（從 3 秒降到 50ms）。

有些優化很複雜（換演算法、重構架構），效果卻不明顯（從 50ms 降到 45ms）。

先做簡單又有效的。

### 3. 不要過度優化

效能夠用就好，不用追求極致。

從 50ms 優化到 10ms，使用者感受不出差別。但可能要花好幾天的時間。

不值得。

### 4. 效能 vs 可讀性

有時候優化會讓 code 變複雜、變難讀。

要在效能和可讀性之間取得平衡。

比如上面的「分兩次查詢」，code 變長了，但效能提升很多，值得。

但如果只是從 50ms 降到 45ms，就不值得犧牲可讀性。

## 目前的效能指標

經過這幾週的優化：

- **P50 回應時間**：28ms（之前 50ms）
- **P95 回應時間**：85ms（之前 200ms）
- **P99 回應時間**：180ms（之前 500ms）
- **錯誤率**：0.05%（之前 0.1%）

符合我們的 SLA 目標：P95 < 100ms。

## 接下來的計畫

效能優化是持續的工作。接下來要做：

1. **Vertical Scaling**：資料庫升級到更大的 tier（目前用 S2,可能要升級到 S3）
2. **Read Replica**：讀寫分離，查詢走 read replica
3. **CDN**：靜態資源（圖片、CSS、JS）走 Azure CDN

這些都要錢,要跟客戶討論預算。

專案還剩三個多月，要確保效能穩定。

下週會寫安全性的處理，特別是認證和授權的設計。
