---
layout: post
title: "ASP.NET Core 快取機制實戰"
date: 2020-07-28 15:30:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Caching, Redis, Performance]
---

這週研究 ASP.NET Core 的快取機制，包含記憶體快取和分散式快取。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0** + **Redis 6.x**

## 為什麼需要快取

快取可以：
- 減少資料庫查詢
- 降低外部 API 呼叫
- 提升回應速度
- 降低伺服器負載

## 記憶體快取 (IMemoryCache)

### 基本使用

```csharp
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly ApplicationDbContext _context;

    public ProductService(IMemoryCache cache, ApplicationDbContext context)
    {
        _cache = cache;
        _context = context;
    }

    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product_{id}";

        // 嘗試從快取取得
        if (_cache.TryGetValue(cacheKey, out Product cachedProduct))
        {
            return cachedProduct;
        }

        // 快取未命中，從資料庫查詢
        var product = await _context.Products.FindAsync(id);

        if (product != null)
        {
            // 設定快取選項
            var cacheOptions = new MemoryCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(5))
                .SetAbsoluteExpiration(TimeSpan.FromHours(1))
                .SetPriority(CacheItemPriority.Normal)
                .RegisterPostEvictionCallback((key, value, reason, state) =>
                {
                    Console.WriteLine($"快取移除: {key}, 原因: {reason}");
                });

            _cache.Set(cacheKey, product, cacheOptions);
        }

        return product;
    }
}
```

### 快取選項

```csharp
// 滑動過期：每次存取後重新計時
.SetSlidingExpiration(TimeSpan.FromMinutes(5))

// 絕對過期：固定時間後過期
.SetAbsoluteExpiration(TimeSpan.FromHours(1))

// 特定時間過期
.SetAbsoluteExpiration(DateTimeOffset.Now.AddHours(1))

// 優先權（記憶體不足時，優先移除低優先權項目）
.SetPriority(CacheItemPriority.High)

// 大小限制
.SetSize(1)
```

### GetOrCreate 方法

```csharp
public async Task<List<Category>> GetCategoriesAsync()
{
    return await _cache.GetOrCreateAsync("categories_all", async entry =>
    {
        entry.SlidingExpiration = TimeSpan.FromMinutes(30);
        entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(2);

        return await _context.Categories
            .AsNoTracking()
            .ToListAsync();
    });
}
```

### 快取依賴

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product_{id}";
    var dependentKey = $"category_{productCategoryId}";

    return await _cache.GetOrCreateAsync(cacheKey, async entry =>
    {
        // 建立 CancellationTokenSource 作為依賴
        var cts = new CancellationTokenSource();
        entry.RegisterPostEvictionCallback((key, value, reason, state) =>
        {
            // 當此快取被移除時，觸發相依快取的移除
            cts.Cancel();
        });

        entry.AddExpirationToken(new CancellationChangeToken(cts.Token));

        return await _context.Products.FindAsync(id);
    });
}
```

## 分散式快取 (IDistributedCache)

分散式快取適用於：
- 多伺服器部署
- 需要跨應用程式共享快取
- 快取資料需要持久化

### SQL Server 分散式快取

```bash
dotnet add package Microsoft.Extensions.Caching.SqlServer
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDistributedSqlServerCache(options =>
    {
        options.ConnectionString = Configuration.GetConnectionString("DefaultConnection");
        options.SchemaName = "dbo";
        options.TableName = "AppCache";
    });
}
```

建立快取資料表：

```bash
dotnet sql-cache create "Server=...;Database=MyDb" dbo AppCache
```

### Redis 分散式快取

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = Configuration.GetConnectionString("Redis");
        options.InstanceName = "MyShopApi_";
    });
}
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,password=yourpassword,ssl=false,abortConnect=false"
  }
}
```

### 使用分散式快取

```csharp
public class ProductService
{
    private readonly IDistributedCache _cache;

    public ProductService(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product_{id}";

        // 從快取取得
        var cachedData = await _cache.GetStringAsync(cacheKey);
        if (cachedData != null)
        {
            return JsonSerializer.Deserialize<Product>(cachedData);
        }

        // 從資料庫查詢
        var product = await _context.Products.FindAsync(id);

        if (product != null)
        {
            var options = new DistributedCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(5))
                .SetAbsoluteExpiration(TimeSpan.FromHours(1));

            var serialized = JsonSerializer.Serialize(product);
            await _cache.SetStringAsync(cacheKey, serialized, options);
        }

        return product;
    }

    public async Task RemoveProductCacheAsync(int id)
    {
        var cacheKey = $"product_{id}";
        await _cache.RemoveAsync(cacheKey);
    }
}
```

### 使用位元組陣列

```csharp
// 儲存
var bytes = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(product));
await _cache.SetAsync(cacheKey, bytes, options);

// 取得
var cachedBytes = await _cache.GetAsync(cacheKey);
if (cachedBytes != null)
{
    var json = Encoding.UTF8.GetString(cachedBytes);
    var product = JsonSerializer.Deserialize<Product>(json);
}
```

## Response Caching

快取整個 HTTP 回應：

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddResponseCaching();
    services.AddControllers();
}

public void Configure(IApplicationBuilder app)
{
    app.UseResponseCaching();
    
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

使用：

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [ResponseCache(Duration = 60)] // 快取 60 秒
    public async Task<IActionResult> GetAll()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }

    [HttpGet("{id}")]
    [ResponseCache(Duration = 300, VaryByQueryKeys = new[] { "id" })]
    public async Task<IActionResult> GetById(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        return Ok(product);
    }

    [HttpGet("featured")]
    [ResponseCache(Duration = 3600, Location = ResponseCacheLocation.Any)]
    public async Task<IActionResult> GetFeatured()
    {
        var products = await _productService.GetFeaturedAsync();
        return Ok(products);
    }

    [HttpPost]
    [ResponseCache(NoStore = true, Location = ResponseCacheLocation.None)]
    public async Task<IActionResult> Create([FromBody] Product product)
    {
        await _productService.CreateAsync(product);
        return Ok();
    }
}
```

ResponseCache 參數：

- `Duration`: 快取持續時間（秒）
- `Location`: 快取位置（Any, Client, None）
- `VaryByQueryKeys`: 依查詢參數變化
- `VaryByHeader`: 依 Header 變化
- `NoStore`: 不快取

## 快取標籤輔助程式

在 Razor Pages 中：

```html
<cache expires-after="@TimeSpan.FromMinutes(5)">
    <!-- 此區塊會被快取 5 分鐘 -->
    @await Component.InvokeAsync("ProductList")
</cache>

<cache vary-by-user="true" expires-sliding="@TimeSpan.FromMinutes(10)">
    <!-- 每個使用者獨立快取 -->
    @await Component.InvokeAsync("UserProfile")
</cache>

<cache vary-by-query="category,page" expires-after="@TimeSpan.FromMinutes(5)">
    <!-- 依查詢參數快取 -->
    @await Component.InvokeAsync("ProductGrid")
</cache>
```

## 快取策略模式

### Cache-Aside (延遲載入)

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product_{id}";

    // 1. 嘗試從快取取得
    var cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null)
    {
        return JsonSerializer.Deserialize<Product>(cached);
    }

    // 2. 快取未命中，從資料庫載入
    var product = await _context.Products.FindAsync(id);

    // 3. 寫入快取
    if (product != null)
    {
        await _cache.SetStringAsync(
            cacheKey,
            JsonSerializer.Serialize(product),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            });
    }

    return product;
}
```

### Write-Through (寫入時更新)

```csharp
public async Task<Product> UpdateProductAsync(Product product)
{
    // 1. 更新資料庫
    _context.Products.Update(product);
    await _context.SaveChangesAsync();

    // 2. 更新快取
    var cacheKey = $"product_{product.Id}";
    await _cache.SetStringAsync(
        cacheKey,
        JsonSerializer.Serialize(product),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        });

    return product;
}
```

### Write-Behind (非同步寫入)

```csharp
public async Task UpdateProductAsync(Product product)
{
    // 1. 立即更新快取
    var cacheKey = $"product_{product.Id}";
    await _cache.SetStringAsync(
        cacheKey,
        JsonSerializer.Serialize(product),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        });

    // 2. 非同步更新資料庫
    _ = Task.Run(async () =>
    {
        await Task.Delay(100); // 延遲寫入
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    });
}
```

### Refresh-Ahead (預先更新)

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product_{id}";
    var cached = await _cache.GetStringAsync(cacheKey);

    if (cached != null)
    {
        var product = JsonSerializer.Deserialize<Product>(cached);

        // 快取即將過期時，背景更新
        _ = Task.Run(async () =>
        {
            await Task.Delay(TimeSpan.FromMinutes(8)); // 10分鐘快取，8分鐘後更新
            var freshProduct = await _context.Products.FindAsync(id);
            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(freshProduct),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
                });
        });

        return product;
    }

    return await LoadAndCacheProductAsync(id);
}
```

## 快取失效策略

### 時間失效

```csharp
// 固定時間過期
var options = new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
};

// 每日凌晨過期
var tomorrow = DateTime.Today.AddDays(1);
var options = new DistributedCacheEntryOptions
{
    AbsoluteExpiration = new DateTimeOffset(tomorrow)
};
```

### 手動失效

```csharp
public async Task UpdateProductAsync(Product product)
{
    _context.Products.Update(product);
    await _context.SaveChangesAsync();

    // 清除快取
    await _cache.RemoveAsync($"product_{product.Id}");
    await _cache.RemoveAsync($"products_category_{product.CategoryId}");
    await _cache.RemoveAsync("products_all");
}
```

### 批次失效

```csharp
public async Task ClearProductCachesAsync(int categoryId)
{
    var products = await _context.Products
        .Where(p => p.CategoryId == categoryId)
        .Select(p => p.Id)
        .ToListAsync();

    var tasks = products.Select(id =>
        _cache.RemoveAsync($"product_{id}"));

    await Task.WhenAll(tasks);
}
```

### 快取版本控制

```csharp
public class CacheKeyGenerator
{
    private static int _version = 1;

    public static string GetKey(string baseKey)
    {
        return $"{baseKey}_v{_version}";
    }

    public static void BumpVersion()
    {
        Interlocked.Increment(ref _version);
    }
}

// 使用
var cacheKey = CacheKeyGenerator.GetKey("products_all");

// 需要清除所有快取時
CacheKeyGenerator.BumpVersion(); // 所有快取鍵都會改變
```

## 快取監控

### 快取命中率

```csharp
public class CacheStatistics
{
    private long _hits;
    private long _misses;

    public void RecordHit() => Interlocked.Increment(ref _hits);
    public void RecordMiss() => Interlocked.Increment(ref _misses);

    public double HitRate => _hits + _misses > 0
        ? (double)_hits / (_hits + _misses)
        : 0;

    public long Hits => _hits;
    public long Misses => _misses;
}

public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly CacheStatistics _stats;

    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product_{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
        {
            _stats.RecordHit();
            return JsonSerializer.Deserialize<Product>(cached);
        }

        _stats.RecordMiss();
        return await LoadAndCacheProductAsync(id);
    }
}
```

### 快取大小監控

```csharp
services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // 設定快取大小限制
    options.CompactionPercentage = 0.25; // 壓縮百分比
});

// 使用時指定大小
_cache.Set(cacheKey, product, new MemoryCacheEntryOptions()
    .SetSize(1));
```

## 快取最佳實踐

1. **選擇適當的快取類型**
   - 單機：使用 IMemoryCache
   - 分散式：使用 Redis

2. **設定合理的過期時間**
   - 經常變更的資料：短時間快取（1-5分鐘）
   - 穩定的資料：長時間快取（數小時）
   - 靜態資料：可快取數天

3. **快取鍵命名規範**
   ```csharp
   var cacheKey = $"{entityType}_{id}_{version}";
   // product_123_v2
   ```

4. **避免快取穿透**
   ```csharp
   // 快取 null 值避免穿透
   if (product == null)
   {
       await _cache.SetStringAsync(cacheKey, "null",
           new DistributedCacheEntryOptions
           {
               AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1)
           });
   }
   ```

5. **使用快取預熱**
   ```csharp
   public async Task WarmUpCacheAsync()
   {
       var popularProducts = await _context.Products
           .OrderByDescending(p => p.ViewCount)
           .Take(100)
           .ToListAsync();

       foreach (var product in popularProducts)
       {
           var cacheKey = $"product_{product.Id}";
           await _cache.SetStringAsync(
               cacheKey,
               JsonSerializer.Serialize(product),
               new DistributedCacheEntryOptions
               {
                   AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
               });
       }
   }
   ```

## 小結

ASP.NET Core 的快取機制：
- IMemoryCache 適合單機快取
- IDistributedCache 適合分散式環境
- Response Caching 適合 HTTP 回應
- 多種快取策略可選擇

相較於 Spring Boot（Spring Cache）：
- 介面設計更簡潔
- Redis 整合更容易
- 內建支援分散式快取
- Response Caching 更直覺

下週將探討 ASP.NET Core 的錯誤處理與例外管理。
