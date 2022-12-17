---
layout: post
title: "Entity Framework Core 效能優化"
date: 2020-07-21 14:55:00 +0800
categories: [框架, .NET]
tags: [EF Core, Performance, Optimization]
---

這週研究 EF Core 的效能優化技巧，避免常見的效能陷阱。

> 本文環境：**EF Core 3.1** + **.NET Core 3.1 LTS**

## 常見效能問題

### N+1 查詢問題

這是 ORM 最常見的效能殺手：

```csharp
// 錯誤：產生 N+1 次查詢
var products = await _context.Products.ToListAsync(); // 1次查詢

foreach (var product in products) // N次查詢
{
    Console.WriteLine($"{product.Name} - {product.Category.Name}");
    // 每次存取 Category 都會查詢資料庫
}
```

產生的 SQL：

```sql
SELECT * FROM Products;  -- 1次
SELECT * FROM Categories WHERE Id = 1;  -- 第1個產品
SELECT * FROM Categories WHERE Id = 2;  -- 第2個產品
-- ... N次查詢
```

正確做法：

```csharp
var products = await _context.Products
    .Include(p => p.Category)  // Eager Loading
    .ToListAsync();

foreach (var product in products)
{
    Console.WriteLine($"{product.Name} - {product.Category.Name}");
}
```

產生的 SQL：

```sql
SELECT p.*, c.*
FROM Products p
INNER JOIN Categories c ON p.CategoryId = c.Id;
```

Hibernate 也有相同問題，解決方式類似。

### 過度查詢

只需要部分欄位時，不要查詢整個實體：

```csharp
// 錯誤：查詢所有欄位
var products = await _context.Products
    .ToListAsync();
var names = products.Select(p => p.Name).ToList();

// 正確：只查詢需要的欄位
var names = await _context.Products
    .Select(p => p.Name)
    .ToListAsync();
```

產生的 SQL 對比：

```sql
-- 錯誤
SELECT Id, Name, Description, Price, Stock, CategoryId, CreatedAt, UpdatedAt
FROM Products;

-- 正確
SELECT Name FROM Products;
```

### 追蹤開銷

唯讀查詢不需要追蹤：

```csharp
// 錯誤：追蹤實體（耗費記憶體）
var products = await _context.Products
    .Where(p => p.Price > 1000)
    .ToListAsync();

// 正確：不追蹤
var products = await _context.Products
    .AsNoTracking()
    .Where(p => p.Price > 1000)
    .ToListAsync();
```

追蹤會：
- 消耗記憶體儲存實體狀態
- 降低查詢效能
- 只在需要更新時才使用

## 批次操作

### 批次插入

```csharp
// 錯誤：逐筆插入
foreach (var product in products)
{
    _context.Products.Add(product);
    await _context.SaveChangesAsync(); // 每次都往返資料庫
}

// 正確：批次插入
_context.Products.AddRange(products);
await _context.SaveChangesAsync(); // 一次寫入
```

### 批次更新

```csharp
// 錯誤：逐筆更新
var products = await _context.Products
    .Where(p => p.CategoryId == 1)
    .ToListAsync();

foreach (var product in products)
{
    product.Price *= 1.1m;
}
await _context.SaveChangesAsync();

// 正確：使用原始 SQL
await _context.Database.ExecuteSqlRawAsync(
    "UPDATE Products SET Price = Price * 1.1 WHERE CategoryId = {0}", 1);
```

或使用 EF Core 擴充套件：

```bash
dotnet add package Z.EntityFramework.Extensions.EFCore
```

```csharp
await _context.Products
    .Where(p => p.CategoryId == 1)
    .UpdateAsync(p => new Product { Price = p.Price * 1.1m });
```

## 投影優化

### 使用 Select 投影

```csharp
// 錯誤：查詢整個實體
var products = await _context.Products
    .Include(p => p.Category)
    .ToListAsync();

var dtos = products.Select(p => new ProductDto
{
    Id = p.Id,
    Name = p.Name,
    CategoryName = p.Category.Name
}).ToList();

// 正確：直接投影
var dtos = await _context.Products
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        CategoryName = p.Category.Name
    })
    .ToListAsync();
```

### 匿名型別投影

```csharp
var summary = await _context.Orders
    .GroupBy(o => o.OrderDate.Date)
    .Select(g => new
    {
        Date = g.Key,
        Count = g.Count(),
        Total = g.Sum(o => o.TotalAmount)
    })
    .ToListAsync();
```

## 分頁最佳化

```csharp
public class PagedResult<T>
{
    public List<T> Items { get; set; }
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
}

public async Task<PagedResult<ProductDto>> GetProductsAsync(
    int pageNumber, int pageSize, string searchTerm = null)
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrEmpty(searchTerm))
    {
        query = query.Where(p => p.Name.Contains(searchTerm));
    }

    var totalCount = await query.CountAsync();

    var items = await query
        .OrderBy(p => p.Id)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price
        })
        .ToListAsync();

    return new PagedResult<ProductDto>
    {
        Items = items,
        TotalCount = totalCount,
        PageNumber = pageNumber,
        PageSize = pageSize
    };
}
```

## 查詢拆分

對於包含多個集合的查詢，使用 AsSplitQuery：

```csharp
// 單一查詢（可能產生笛卡爾積）
var blogs = await _context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .ToListAsync();

// 拆分查詢（多個獨立查詢）
var blogs = await _context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .AsSplitQuery()
    .ToListAsync();
```

產生的 SQL：

```sql
-- AsSplitQuery
SELECT * FROM Blogs;
SELECT * FROM Posts WHERE BlogId IN (...);
SELECT * FROM Contributors WHERE BlogId IN (...);

-- 單一查詢
SELECT *
FROM Blogs b
LEFT JOIN Posts p ON b.Id = p.BlogId
LEFT JOIN Contributors c ON b.Id = c.BlogId;
```

## 編譯查詢

對於頻繁執行的查詢，使用編譯查詢：

```csharp
private static readonly Func<ApplicationDbContext, int, Task<Product>> _getProductQuery =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
        context.Products
            .Include(p => p.Category)
            .FirstOrDefault(p => p.Id == id));

public async Task<Product> GetProductAsync(int id)
{
    return await _getProductQuery(_context, id);
}
```

編譯查詢會快取查詢計畫，避免重複編譯。

## 索引優化

### 建立索引

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 單一欄位索引
    modelBuilder.Entity<Product>()
        .HasIndex(p => p.Name);

    // 唯一索引
    modelBuilder.Entity<User>()
        .HasIndex(u => u.Email)
        .IsUnique();

    // 複合索引
    modelBuilder.Entity<OrderItem>()
        .HasIndex(oi => new { oi.OrderId, oi.ProductId });

    // 包含欄位的索引（涵蓋索引）
    modelBuilder.Entity<Product>()
        .HasIndex(p => p.CategoryId)
        .IncludeProperties(p => new { p.Name, p.Price });

    // 篩選索引
    modelBuilder.Entity<Product>()
        .HasIndex(p => p.UpdatedAt)
        .HasFilter("[UpdatedAt] IS NOT NULL");
}
```

### 檢查執行計畫

```csharp
var query = _context.Products
    .Where(p => p.Price > 1000)
    .Include(p => p.Category);

var sql = query.ToQueryString();
Console.WriteLine(sql);
```

## 快取策略

### 記憶體快取

```csharp
public class ProductService
{
    private readonly ApplicationDbContext _context;
    private readonly IMemoryCache _cache;

    public ProductService(ApplicationDbContext context, IMemoryCache cache)
    {
        _context = context;
        _cache = cache;
    }

    public async Task<List<Category>> GetCategoriesAsync()
    {
        var cacheKey = "categories_all";

        if (!_cache.TryGetValue(cacheKey, out List<Category> categories))
        {
            categories = await _context.Categories
                .AsNoTracking()
                .ToListAsync();

            var cacheOptions = new MemoryCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(30))
                .SetAbsoluteExpiration(TimeSpan.FromHours(2));

            _cache.Set(cacheKey, categories, cacheOptions);
        }

        return categories;
    }
}
```

### 分散式快取

```csharp
public class ProductService
{
    private readonly IDistributedCache _cache;

    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product_{id}";
        var cachedData = await _cache.GetStringAsync(cacheKey);

        if (cachedData != null)
        {
            return JsonSerializer.Deserialize<Product>(cachedData);
        }

        var product = await _context.Products
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Id == id);

        if (product != null)
        {
            var options = new DistributedCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(10));

            await _cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(product),
                options);
        }

        return product;
    }
}
```

## 連線池設定

```csharp
// Startup.cs
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        // 設定指令逾時
        sqlOptions.CommandTimeout(30);
        
        // 啟用重試機制
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null);
        
        // 連線池大小
        sqlOptions.MaxBatchSize(100);
    }));
```

連線字串設定：

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyShopDb;User Id=sa;Password=Pass123;Min Pool Size=5;Max Pool Size=100;Connection Timeout=30;"
  }
}
```

## 效能監控

### EF Core 日誌

```json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

### 效能計數器

```csharp
public class PerformanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMiddleware> _logger;

    public PerformanceMiddleware(RequestDelegate next, ILogger<PerformanceMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();

        await _next(context);

        sw.Stop();

        if (sw.ElapsedMilliseconds > 1000)
        {
            _logger.LogWarning(
                "慢速請求: {Method} {Path} 耗時 {Elapsed}ms",
                context.Request.Method,
                context.Request.Path,
                sw.ElapsedMilliseconds);
        }
    }
}
```

### MiniProfiler

```bash
dotnet add package MiniProfiler.AspNetCore.Mvc
dotnet add package MiniProfiler.EntityFrameworkCore
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddMiniProfiler(options =>
    {
        options.RouteBasePath = "/profiler";
    }).AddEntityFramework();

    services.AddDbContext<ApplicationDbContext>(options =>
    {
        options.UseSqlServer(connectionString);
        options.EnableSensitiveDataLogging(); // 開發環境
    });
}

public void Configure(IApplicationBuilder app)
{
    app.UseMiniProfiler();
    // ...
}
```

存取 `/profiler/results-index` 查看效能資料。

## 最佳實踐檢查清單

- [ ] 避免 N+1 查詢，使用 Include
- [ ] 唯讀查詢使用 AsNoTracking
- [ ] 使用投影減少資料量
- [ ] 批次操作取代逐筆處理
- [ ] 為常查詢的欄位建立索引
- [ ] 使用快取減少資料庫查詢
- [ ] 大量資料使用分頁
- [ ] 避免在迴圈中查詢資料庫
- [ ] 使用編譯查詢優化熱路徑
- [ ] 監控慢速查詢

## 效能測試工具

### BenchmarkDotNet

```bash
dotnet add package BenchmarkDotNet
```

```csharp
[MemoryDiagnoser]
public class ProductQueryBenchmarks
{
    private ApplicationDbContext _context;

    [GlobalSetup]
    public void Setup()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer("...")
            .Options;
        
        _context = new ApplicationDbContext(options);
    }

    [Benchmark]
    public async Task<List<Product>> WithTracking()
    {
        return await _context.Products.ToListAsync();
    }

    [Benchmark]
    public async Task<List<Product>> WithoutTracking()
    {
        return await _context.Products.AsNoTracking().ToListAsync();
    }

    [GlobalCleanup]
    public void Cleanup()
    {
        _context?.Dispose();
    }
}
```

## 小結

EF Core 效能優化重點：
- 理解查詢生成的 SQL
- 避免 N+1 問題
- 適當使用 AsNoTracking
- 批次操作取代逐筆處理
- 建立適當的索引
- 使用快取策略

相較於 Hibernate：
- 效能特性類似
- 優化方式相通
- LINQ 更容易寫出高效查詢
- 內建工具更完善

下週將探討 ASP.NET Core 的快取機制。
