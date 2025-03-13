---
layout: post
title: "快取策略的平衡：不是所有東西都該放進 Redis"
date: 2025-03-14 09:20:00 +0800
categories: [效能優化]
tags: [Redis, 快取, .NET Core, Azure]
---

## 快取的迷思

很多人以為快取就是：「把資料庫的東西搬到 Redis，讀取就會變快。」

這沒錯，但太簡化了。

快取帶來的問題不比解決的少：
- **資料一致性**：Redis 的資料跟資料庫不同步怎麼辦？
- **記憶體成本**：Redis 用記憶體，不便宜
- **快取失效**：資料更新了，快取要怎麼清掉？
- **快取雪崩**：大量快取同時失效，資料庫瞬間爆掉

所以重點不是「要不要快取」，而是「什麼該快取，什麼不該快取」。

## 我們的快取策略

經過討論，我們把資料分成三類：

### 1. 必須快取：經常讀取、很少變動

**會員基本資料**：
- 讀取頻率：每個 API 幾乎都要查會員資料
- 變動頻率：會員資料很少改（可能幾個月才改一次名字或電話）
- 快取時效：30 分鐘

```csharp
public class MemberQuery : IMemberQuery
{
    private readonly IDistributedCache _cache;
    private readonly MemberDbContext _dbContext;
    
    public async Task<MemberDto> GetMemberAsync(string memberId)
    {
        // 先查 Redis
        var cacheKey = $"member:{memberId}";
        var cached = await _cache.GetStringAsync(cacheKey);
        
        if (cached != null)
        {
            return JsonSerializer.Deserialize<MemberDto>(cached);
        }
        
        // Redis 沒有，查資料庫
        var member = await _dbContext.Members
            .Where(m => m.Id == memberId)
            .FirstOrDefaultAsync();
        
        if (member == null) return null;
        
        var dto = MapToDto(member);
        
        // 存進 Redis，30 分鐘過期
        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(dto),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
            });
        
        return dto;
    }
}
```

**效果**：會員查詢的平均延遲從 15ms 降到 2ms（Redis 的回應時間）。

### 2. 選擇性快取：讀多寫少、但可能過時

**優惠券列表**：
- 讀取頻率：使用者每次結帳都要查可用優惠券
- 變動頻率：運營人員偶爾會新增或停用優惠券
- 快取時效：5 分鐘

這種資料可以容許「短暫的不一致」。

比如運營人員新增了一張優惠券，使用者可能要等 5 分鐘才看得到。但這不是大問題，因為優惠券通常是活動期間一直有效，不會頻繁變動。

```csharp
public async Task<List<CouponDto>> GetAvailableCouponsAsync(string memberId)
{
    var cacheKey = $"coupons:member:{memberId}";
    var cached = await _cache.GetStringAsync(cacheKey);
    
    if (cached != null)
    {
        return JsonSerializer.Deserialize<List<CouponDto>>(cached);
    }
    
    var coupons = await _dbContext.Coupons
        .Where(c => c.IsActive && c.MemberLevel == memberLevel)
        .ToListAsync();
    
    var dtos = coupons.Select(MapToDto).ToList();
    
    await _cache.SetStringAsync(
        cacheKey,
        JsonSerializer.Serialize(dtos),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
        });
    
    return dtos;
}
```

### 3. 不該快取：經常變動、要求即時

**點數餘額**：
- 讀取頻率：高
- 變動頻率：也高（使用者每次消費都會扣點數）
- 即時性要求：強（不能顯示錯誤的餘額）

點數餘額**不該快取**。

如果快取了，會發生這種情況：
1. 使用者 A 的餘額是 1000 點，存進 Redis
2. 使用者 A 花了 500 點
3. Redis 還是顯示 1000 點（因為快取沒失效）
4. 使用者 A 以為還有 1000 點，又花了 800 點
5. 結果餘額不足，交易失敗，使用者抱怨

這種資料就直接查資料庫，不要快取。

```csharp
public async Task<int> GetPointsBalanceAsync(string memberId)
{
    // 直接查資料庫，不快取
    var account = await _dbContext.PointsAccounts
        .Where(a => a.MemberId == memberId)
        .FirstOrDefaultAsync();
    
    return account?.Balance ?? 0;
}
```

## Cache-Aside Pattern

我們用的是 **Cache-Aside（也叫 Lazy Loading）** 模式：

1. 讀取時，先查快取
2. 快取沒有，查資料庫
3. 查到後存進快取

這個模式的好處是：
- 只有真正需要的資料才會進快取（不會浪費記憶體）
- 實作簡單

缺點是：
- 第一次讀取會比較慢（要查資料庫 + 寫快取）
- 快取失效後，第一個 request 要等比較久

不過這對我們來說不是問題，因為快取的有效期夠長（30 分鐘），而且我們的資料量不大。

## 快取失效的處理

當資料更新時，要主動清除快取：

```csharp
public async Task UpdateMemberAsync(string memberId, UpdateMemberRequest request)
{
    // 更新資料庫
    var member = await _dbContext.Members.FindAsync(memberId);
    member.Name = request.Name;
    member.Phone = request.Phone;
    await _dbContext.SaveChangesAsync();
    
    // 清除快取
    var cacheKey = $"member:{memberId}";
    await _cache.RemoveAsync(cacheKey);
}
```

這樣下次讀取時，會重新從資料庫載入最新的資料。

## Redis 的設定

我們用 Azure Cache for Redis，選了 Basic C1（1GB）：

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp:";
});
```

為什麼用 Azure Cache for Redis 而不是自己架？

- **託管服務**：不用自己維護、備份、監控
- **高可用**：有自動 failover
- **便宜**：Basic C1 一個月 $15，很划算

如果未來流量變大，再升級到 Standard 或 Premium。

## 監控快取效果

加了快取之後，要監控效果：

```csharp
public class CachedMemberQuery : IMemberQuery
{
    private readonly IDistributedCache _cache;
    private readonly MemberDbContext _dbContext;
    private readonly ILogger<CachedMemberQuery> _logger;
    
    public async Task<MemberDto> GetMemberAsync(string memberId)
    {
        var cacheKey = $"member:{memberId}";
        var cached = await _cache.GetStringAsync(cacheKey);
        
        if (cached != null)
        {
            _logger.LogInformation("快取命中：{MemberId}", memberId);
            return JsonSerializer.Deserialize<MemberDto>(cached);
        }
        
        _logger.LogInformation("快取未命中，查詢資料庫：{MemberId}", memberId);
        
        var member = await _dbContext.Members
            .Where(m => m.Id == memberId)
            .FirstOrDefaultAsync();
        
        // ... 存快取
        
        return dto;
    }
}
```

在 Application Insights 可以看到：
- **Cache Hit Rate**：快取命中率（目前約 92%）
- **Database Query Count**：資料庫查詢次數（減少了 10 倍）

## 快取不是銀彈

雖然快取很有效，但不是萬靈丹。

加快取之前要先問自己：
- 這個資料真的值得快取嗎？（讀取頻率夠高？）
- 快取過期了會有什麼影響？（使用者會看到舊資料？）
- 資料更新時，要怎麼清快取？（手動清除？還是等自然過期？）

不要為了快取而快取。

有時候優化資料庫查詢、加個索引，效果比快取更好，而且不用處理一致性問題。

## 接下來的計畫

快取策略建立好了，但還有一些要改進的地方：

1. **Cache Warming**：系統啟動時，主動載入熱門資料進快取
2. **分散式鎖**：避免 cache stampede（大量 request 同時查資料庫）
3. **監控告警**：Redis 掛掉或記憶體不足時要通知

這些會在接下來幾週慢慢實作。

專案還剩五個多月，要繼續穩紮穩打。

下週會寫測試策略，特別是怎麼測試模組之間的整合。
