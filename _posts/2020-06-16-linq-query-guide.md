---
layout: post
title: "LINQ 查詢語法完全指南"
date: 2020-06-16 15:10:00 +0800
categories: [框架, .NET]
tags: [LINQ, C#, 查詢]
---

LINQ (Language Integrated Query) 是 .NET 最強大的功能之一，它讓查詢操作成為語言的一部分，類似 Java 8 的 Stream API，但更加強大。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Java Stream API 回顧

在 Java 中，我們這樣操作集合：

```java
List<Product> products = productRepository.findAll();

// 過濾、排序、轉換
List<String> names = products.stream()
    .filter(p -> p.getPrice() > 1000)
    .sorted(Comparator.comparing(Product::getName))
    .map(Product::getName)
    .collect(Collectors.toList());

// 分組
Map<String, List<Product>> groupedByCategory = products.stream()
    .collect(Collectors.groupingBy(Product::getCategory));
```

## LINQ 基本語法

LINQ 有兩種語法：查詢語法和方法語法。

### 方法語法（類似 Java Stream）

```csharp
var products = _context.Products
    .Where(p => p.Price > 1000)
    .OrderBy(p => p.Name)
    .Select(p => p.Name)
    .ToList();
```

### 查詢語法（類似 SQL）

```csharp
var products = from p in _context.Products
               where p.Price > 1000
               orderby p.Name
               select p.Name;

var list = products.ToList();
```

兩種語法功能相同，方法語法更常用。

## 基本查詢操作

### Where - 過濾

```csharp
// 單一條件
var expensiveProducts = await _context.Products
    .Where(p => p.Price > 1000)
    .ToListAsync();

// 多重條件
var products = await _context.Products
    .Where(p => p.Price > 1000 && p.Stock > 0)
    .ToListAsync();

// 鏈式過濾
var products = await _context.Products
    .Where(p => p.Price > 1000)
    .Where(p => p.Stock > 0)
    .Where(p => p.CategoryId == 1)
    .ToListAsync();
```

Java Stream 對應：
```java
List<Product> products = productRepository.findAll().stream()
    .filter(p -> p.getPrice().compareTo(new BigDecimal("1000")) > 0)
    .filter(p -> p.getStock() > 0)
    .collect(Collectors.toList());
```

### Select - 投影

```csharp
// 選擇單一屬性
var names = await _context.Products
    .Select(p => p.Name)
    .ToListAsync();

// 選擇多個屬性（匿名型別）
var products = await _context.Products
    .Select(p => new
    {
        p.Id,
        p.Name,
        p.Price
    })
    .ToListAsync();

// 選擇到 DTO
var productDtos = await _context.Products
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price,
        CategoryName = p.Category.Name
    })
    .ToListAsync();
```

Java Stream 對應：
```java
List<String> names = products.stream()
    .map(Product::getName)
    .collect(Collectors.toList());

List<ProductDto> dtos = products.stream()
    .map(p -> new ProductDto(p.getId(), p.getName(), p.getPrice()))
    .collect(Collectors.toList());
```

### OrderBy - 排序

```csharp
// 升冪排序
var products = await _context.Products
    .OrderBy(p => p.Price)
    .ToListAsync();

// 降冪排序
var products = await _context.Products
    .OrderByDescending(p => p.Price)
    .ToListAsync();

// 多重排序
var products = await _context.Products
    .OrderBy(p => p.CategoryId)
    .ThenByDescending(p => p.Price)
    .ThenBy(p => p.Name)
    .ToListAsync();
```

Java Stream 對應：
```java
List<Product> products = products.stream()
    .sorted(Comparator.comparing(Product::getCategoryId)
            .thenComparing(Comparator.comparing(Product::getPrice).reversed())
            .thenComparing(Product::getName))
    .collect(Collectors.toList());
```

### Take / Skip - 分頁

```csharp
// 取前 10 筆
var products = await _context.Products
    .Take(10)
    .ToListAsync();

// 跳過前 20 筆，取 10 筆
var products = await _context.Products
    .Skip(20)
    .Take(10)
    .ToListAsync();

// 分頁
public async Task<List<Product>> GetProductsPageAsync(int pageNumber, int pageSize)
{
    return await _context.Products
        .OrderBy(p => p.Id)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();
}
```

Java Stream 對應：
```java
List<Product> products = products.stream()
    .skip(20)
    .limit(10)
    .collect(Collectors.toList());
```

## 聚合操作

### Count

```csharp
// 總數
var count = await _context.Products.CountAsync();

// 條件計數
var count = await _context.Products
    .Where(p => p.Price > 1000)
    .CountAsync();

// LongCount（長整數）
var count = await _context.Products.LongCountAsync();
```

### Sum / Average / Min / Max

```csharp
// 總和
var totalValue = await _context.Products
    .SumAsync(p => p.Price * p.Stock);

// 平均
var avgPrice = await _context.Products
    .AverageAsync(p => p.Price);

// 最小值
var minPrice = await _context.Products
    .MinAsync(p => p.Price);

// 最大值
var maxPrice = await _context.Products
    .MaxAsync(p => p.Price);
```

Java Stream 對應：
```java
BigDecimal totalValue = products.stream()
    .map(p -> p.getPrice().multiply(new BigDecimal(p.getStock())))
    .reduce(BigDecimal.ZERO, BigDecimal::add);

double avgPrice = products.stream()
    .mapToDouble(p -> p.getPrice().doubleValue())
    .average()
    .orElse(0.0);
```

### Any / All

```csharp
// 是否有任何符合條件的資料
var hasExpensive = await _context.Products
    .AnyAsync(p => p.Price > 10000);

// 是否所有資料都符合條件
var allInStock = await _context.Products
    .AllAsync(p => p.Stock > 0);
```

Java Stream 對應：
```java
boolean hasExpensive = products.stream()
    .anyMatch(p -> p.getPrice().compareTo(new BigDecimal("10000")) > 0);

boolean allInStock = products.stream()
    .allMatch(p -> p.getStock() > 0);
```

## 單一結果查詢

```csharp
// 第一筆（若無則拋出異常）
var product = await _context.Products.FirstAsync();

// 第一筆（若無則回傳 null）
var product = await _context.Products.FirstOrDefaultAsync();

// 符合條件的第一筆
var product = await _context.Products
    .FirstOrDefaultAsync(p => p.Id == 1);

// 單一筆（若有多筆則拋出異常）
var product = await _context.Products
    .SingleOrDefaultAsync(p => p.Id == 1);

// 最後一筆
var product = await _context.Products
    .OrderBy(p => p.Id)
    .LastOrDefaultAsync();
```

## 分組查詢

```csharp
// 依類別分組
var groups = await _context.Products
    .GroupBy(p => p.CategoryId)
    .Select(g => new
    {
        CategoryId = g.Key,
        Count = g.Count(),
        TotalValue = g.Sum(p => p.Price * p.Stock),
        AvgPrice = g.Average(p => p.Price)
    })
    .ToListAsync();

// 多層分組
var groups = await _context.Products
    .GroupBy(p => new { p.CategoryId, HasStock = p.Stock > 0 })
    .Select(g => new
    {
        g.Key.CategoryId,
        g.Key.HasStock,
        Count = g.Count()
    })
    .ToListAsync();
```

Java Stream 對應：
```java
Map<Integer, List<Product>> groups = products.stream()
    .collect(Collectors.groupingBy(Product::getCategoryId));

Map<Integer, Long> counts = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategoryId,
        Collectors.counting()
    ));
```

## 聯接查詢

### Inner Join

```csharp
var query = from p in _context.Products
            join c in _context.Categories on p.CategoryId equals c.Id
            select new
            {
                ProductName = p.Name,
                CategoryName = c.Name,
                p.Price
            };

var result = await query.ToListAsync();

// 方法語法
var result = await _context.Products
    .Join(_context.Categories,
        product => product.CategoryId,
        category => category.Id,
        (product, category) => new
        {
            ProductName = product.Name,
            CategoryName = category.Name,
            product.Price
        })
    .ToListAsync();
```

### Left Join

```csharp
var query = from p in _context.Products
            join c in _context.Categories on p.CategoryId equals c.Id into pc
            from c in pc.DefaultIfEmpty()
            select new
            {
                ProductName = p.Name,
                CategoryName = c != null ? c.Name : "無分類",
                p.Price
            };

var result = await query.ToListAsync();

// 方法語法
var result = await _context.Products
    .GroupJoin(_context.Categories,
        product => product.CategoryId,
        category => category.Id,
        (product, categories) => new { product, categories })
    .SelectMany(
        x => x.categories.DefaultIfEmpty(),
        (x, category) => new
        {
            ProductName = x.product.Name,
            CategoryName = category != null ? category.Name : "無分類",
            x.product.Price
        })
    .ToListAsync();
```

但在 EF Core 中，更好的方式是使用導覽屬性：

```csharp
var result = await _context.Products
    .Include(p => p.Category)
    .Select(p => new
    {
        ProductName = p.Name,
        CategoryName = p.Category.Name,
        p.Price
    })
    .ToListAsync();
```

## 集合操作

### Distinct - 去除重複

```csharp
var categoryIds = await _context.Products
    .Select(p => p.CategoryId)
    .Distinct()
    .ToListAsync();
```

### Union - 聯集

```csharp
var expensiveProducts = _context.Products.Where(p => p.Price > 5000);
var featuredProducts = _context.Products.Where(p => p.IsFeatured);

var combined = await expensiveProducts
    .Union(featuredProducts)
    .ToListAsync();
```

### Intersect - 交集

```csharp
var expensiveAndFeatured = await expensiveProducts
    .Intersect(featuredProducts)
    .ToListAsync();
```

### Except - 差集

```csharp
var expensiveNotFeatured = await expensiveProducts
    .Except(featuredProducts)
    .ToListAsync();
```

## 條件查詢

### 動態 Where

```csharp
public async Task<List<Product>> SearchProductsAsync(ProductSearchCriteria criteria)
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrEmpty(criteria.Name))
    {
        query = query.Where(p => p.Name.Contains(criteria.Name));
    }

    if (criteria.MinPrice.HasValue)
    {
        query = query.Where(p => p.Price >= criteria.MinPrice.Value);
    }

    if (criteria.MaxPrice.HasValue)
    {
        query = query.Where(p => p.Price <= criteria.MaxPrice.Value);
    }

    if (criteria.CategoryId.HasValue)
    {
        query = query.Where(p => p.CategoryId == criteria.CategoryId.Value);
    }

    if (criteria.InStockOnly)
    {
        query = query.Where(p => p.Stock > 0);
    }

    return await query.ToListAsync();
}
```

## 字串操作

```csharp
// 包含
var products = await _context.Products
    .Where(p => p.Name.Contains("手機"))
    .ToListAsync();

// 開頭
var products = await _context.Products
    .Where(p => p.Name.StartsWith("iPhone"))
    .ToListAsync();

// 結尾
var products = await _context.Products
    .Where(p => p.Name.EndsWith("Pro"))
    .ToListAsync();

// 忽略大小寫
var products = await _context.Products
    .Where(p => p.Name.ToLower().Contains("iphone"))
    .ToListAsync();

// SQL LIKE
var products = await _context.Products
    .Where(p => EF.Functions.Like(p.Name, "%手機%"))
    .ToListAsync();
```

## 日期操作

```csharp
// 今天建立的產品
var today = DateTime.Today;
var products = await _context.Products
    .Where(p => p.CreatedAt >= today && p.CreatedAt < today.AddDays(1))
    .ToListAsync();

// 本月訂單
var startOfMonth = new DateTime(DateTime.Now.Year, DateTime.Now.Month, 1);
var orders = await _context.Orders
    .Where(o => o.OrderDate >= startOfMonth)
    .ToListAsync();

// 過去 7 天
var sevenDaysAgo = DateTime.Now.AddDays(-7);
var recentOrders = await _context.Orders
    .Where(o => o.OrderDate >= sevenDaysAgo)
    .ToListAsync();
```

## 複雜查詢範例

### 訂單統計

```csharp
public async Task<OrderStatistics> GetOrderStatisticsAsync(DateTime startDate, DateTime endDate)
{
    var statistics = await _context.Orders
        .Where(o => o.OrderDate >= startDate && o.OrderDate <= endDate)
        .GroupBy(o => 1) // 全部分為一組
        .Select(g => new OrderStatistics
        {
            TotalOrders = g.Count(),
            TotalAmount = g.Sum(o => o.TotalAmount),
            AverageAmount = g.Average(o => o.TotalAmount),
            MaxAmount = g.Max(o => o.TotalAmount),
            MinAmount = g.Min(o => o.TotalAmount)
        })
        .FirstOrDefaultAsync();

    return statistics ?? new OrderStatistics();
}
```

### 暢銷產品排行

```csharp
public async Task<List<ProductSalesDto>> GetTopSellingProductsAsync(int top = 10)
{
    var result = await _context.OrderItems
        .GroupBy(oi => new
        {
            oi.ProductId,
            oi.Product.Name
        })
        .Select(g => new ProductSalesDto
        {
            ProductId = g.Key.ProductId,
            ProductName = g.Key.Name,
            TotalQuantity = g.Sum(oi => oi.Quantity),
            TotalAmount = g.Sum(oi => oi.Quantity * oi.UnitPrice)
        })
        .OrderByDescending(x => x.TotalQuantity)
        .Take(top)
        .ToListAsync();

    return result;
}
```

### 客戶購買分析

```csharp
public async Task<List<CustomerAnalysisDto>> GetCustomerAnalysisAsync()
{
    var result = await _context.Orders
        .GroupBy(o => o.CustomerId)
        .Select(g => new CustomerAnalysisDto
        {
            CustomerId = g.Key,
            OrderCount = g.Count(),
            TotalSpent = g.Sum(o => o.TotalAmount),
            AverageOrderValue = g.Average(o => o.TotalAmount),
            FirstOrderDate = g.Min(o => o.OrderDate),
            LastOrderDate = g.Max(o => o.OrderDate)
        })
        .Where(c => c.OrderCount >= 3) // 至少 3 筆訂單
        .OrderByDescending(c => c.TotalSpent)
        .ToListAsync();

    return result;
}
```

## LINQ 效能考量

### 避免 N+1 查詢

```csharp
// 錯誤：N+1 查詢
var products = await _context.Products.ToListAsync();
foreach (var product in products)
{
    var category = await _context.Categories
        .FindAsync(product.CategoryId); // 每次都查詢資料庫
}

// 正確：使用 Include
var products = await _context.Products
    .Include(p => p.Category)
    .ToListAsync();
```

### 投影減少資料量

```csharp
// 錯誤：查詢整個實體
var products = await _context.Products
    .Where(p => p.CategoryId == 1)
    .ToListAsync();
var names = products.Select(p => p.Name).ToList();

// 正確：直接投影需要的欄位
var names = await _context.Products
    .Where(p => p.CategoryId == 1)
    .Select(p => p.Name)
    .ToListAsync();
```

### 使用 AsNoTracking

```csharp
// 唯讀查詢使用 AsNoTracking
var products = await _context.Products
    .AsNoTracking()
    .Where(p => p.Price > 1000)
    .ToListAsync();
```

## 小結

LINQ 相較於 Java Stream API：
- 語法更簡潔直覺
- 支援查詢語法（類似 SQL）
- 與 EF Core 整合完美（自動轉換為 SQL）
- 延遲執行（Deferred Execution）
- 功能更強大（如分組、聯接）

LINQ 是 .NET 開發的核心技能，熟練使用可以大幅提升開發效率。

下週將探討 EF Core 的進階功能：關聯配置與 Migration。
