---
layout: post
title: "Entity Framework Core 進階技巧"
date: 2020-10-27 14:45:00 +0800
categories: [框架, .NET]
tags: [.NET Core, EF Core, ORM, Database]
---

這週深入探討 Entity Framework Core 的進階功能，包含 Query Filters、Shadow Properties、Table Splitting 等實用技巧。

> 本文環境：**.NET Core 3.1 LTS** + **EF Core 3.1**

## Global Query Filters

### 實作軟刪除

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 設定全域查詢過濾器
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => !p.IsDeleted);

        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => !o.IsDeleted);
    }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}

// 使用
public class ProductService
{
    public async Task<List<Product>> GetProductsAsync()
    {
        // 自動過濾已刪除的產品
        return await _context.Products.ToListAsync();
    }

    public async Task<List<Product>> GetAllIncludingDeletedAsync()
    {
        // 忽略查詢過濾器
        return await _context.Products
            .IgnoreQueryFilters()
            .ToListAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product != null)
        {
            product.IsDeleted = true;
            product.DeletedAt = DateTime.UtcNow;
            await _context.SaveChangesAsync();
        }
    }
}
```

### 多租戶過濾

```csharp
public interface ITenantProvider
{
    int GetCurrentTenantId();
}

public class ApplicationDbContext : DbContext
{
    private readonly ITenantProvider _tenantProvider;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        ITenantProvider tenantProvider)
        : base(options)
    {
        _tenantProvider = tenantProvider;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 多租戶過濾
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantProvider.GetCurrentTenantId());

        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenantProvider.GetCurrentTenantId());
    }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int TenantId { get; set; }
}
```

## Shadow Properties

### 使用隱藏屬性

```csharp
public class ApplicationDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 定義 Shadow Properties
        modelBuilder.Entity<Product>()
            .Property<DateTime>("CreatedAt");

        modelBuilder.Entity<Product>()
            .Property<DateTime>("UpdatedAt");

        modelBuilder.Entity<Product>()
            .Property<string>("CreatedBy");

        modelBuilder.Entity<Product>()
            .Property<string>("UpdatedBy");
    }

    public override Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        var entries = ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified);

        foreach (var entry in entries)
        {
            var now = DateTime.UtcNow;
            var currentUser = "system"; // 從認證取得

            if (entry.State == EntityState.Added)
            {
                entry.Property("CreatedAt").CurrentValue = now;
                entry.Property("CreatedBy").CurrentValue = currentUser;
            }

            entry.Property("UpdatedAt").CurrentValue = now;
            entry.Property("UpdatedBy").CurrentValue = currentUser;
        }

        return base.SaveChangesAsync(cancellationToken);
    }
}

// 查詢 Shadow Properties
public class ProductService
{
    public async Task<List<ProductAudit>> GetProductAuditAsync()
    {
        return await _context.Products
            .Select(p => new ProductAudit
            {
                Id = p.Id,
                Name = p.Name,
                CreatedAt = EF.Property<DateTime>(p, "CreatedAt"),
                UpdatedAt = EF.Property<DateTime>(p, "UpdatedAt"),
                CreatedBy = EF.Property<string>(p, "CreatedBy"),
                UpdatedBy = EF.Property<string>(p, "UpdatedBy")
            })
            .ToListAsync();
    }
}
```

## Owned Entity Types

### 值物件

```csharp
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public Address ShippingAddress { get; set; }
    public Address BillingAddress { get; set; }
}

public class ApplicationDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Customer>().OwnsOne(c => c.ShippingAddress, sa =>
        {
            sa.Property(a => a.Street).HasColumnName("ShippingStreet");
            sa.Property(a => a.City).HasColumnName("ShippingCity");
            sa.Property(a => a.State).HasColumnName("ShippingState");
            sa.Property(a => a.ZipCode).HasColumnName("ShippingZipCode");
        });

        modelBuilder.Entity<Customer>().OwnsOne(c => c.BillingAddress, ba =>
        {
            ba.Property(a => a.Street).HasColumnName("BillingStreet");
            ba.Property(a => a.City).HasColumnName("BillingCity");
            ba.Property(a => a.State).HasColumnName("BillingState");
            ba.Property(a => a.ZipCode).HasColumnName("BillingZipCode");
        });
    }
}

// 使用
public class CustomerService
{
    public async Task CreateCustomerAsync(CreateCustomerRequest request)
    {
        var customer = new Customer
        {
            Name = request.Name,
            ShippingAddress = new Address
            {
                Street = request.ShippingStreet,
                City = request.ShippingCity,
                State = request.ShippingState,
                ZipCode = request.ShippingZipCode
            },
            BillingAddress = new Address
            {
                Street = request.BillingStreet,
                City = request.BillingCity,
                State = request.BillingState,
                ZipCode = request.BillingZipCode
            }
        };

        _context.Customers.Add(customer);
        await _context.SaveChangesAsync();
    }
}
```

## Table Splitting

### 將一個資料表映射到多個實體

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal TotalAmount { get; set; }
    public OrderDetails Details { get; set; }
}

public class OrderDetails
{
    public int OrderId { get; set; }
    public string ShippingAddress { get; set; }
    public string PaymentMethod { get; set; }
    public string Notes { get; set; }
}

public class ApplicationDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>().ToTable("Orders");
        modelBuilder.Entity<OrderDetails>().ToTable("Orders");

        modelBuilder.Entity<Order>()
            .HasOne(o => o.Details)
            .WithOne()
            .HasForeignKey<OrderDetails>(d => d.OrderId);

        modelBuilder.Entity<OrderDetails>()
            .Property(d => d.OrderId)
            .ValueGeneratedNever();
    }
}

// 查詢
public async Task<Order> GetOrderAsync(int id)
{
    return await _context.Orders
        .Include(o => o.Details)
        .FirstOrDefaultAsync(o => o.Id == id);
}
```

## Interceptors

### 自訂 Command Interceptor

```csharp
public class AuditCommandInterceptor : DbCommandInterceptor
{
    private readonly ILogger<AuditCommandInterceptor> _logger;

    public AuditCommandInterceptor(ILogger<AuditCommandInterceptor> logger)
    {
        _logger = logger;
    }

    public override InterceptionResult<int> NonQueryExecuting(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<int> result)
    {
        _logger.LogInformation(
            "執行 SQL: {CommandText}",
            command.CommandText);

        return base.NonQueryExecuting(command, eventData, result);
    }

    public override async ValueTask<InterceptionResult<int>> NonQueryExecutingAsync(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation(
            "執行 SQL: {CommandText}",
            command.CommandText);

        return await base.NonQueryExecutingAsync(
            command, eventData, result, cancellationToken);
    }
}

// 註冊
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<ApplicationDbContext>((sp, options) =>
    {
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"))
            .AddInterceptors(sp.GetRequiredService<AuditCommandInterceptor>());
    });

    services.AddScoped<AuditCommandInterceptor>();
}
```

### Connection Interceptor

```csharp
public class ConnectionInterceptor : DbConnectionInterceptor
{
    public override InterceptionResult ConnectionOpening(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result)
    {
        Console.WriteLine($"開啟連線: {connection.Database}");
        return base.ConnectionOpening(connection, eventData, result);
    }

    public override void ConnectionClosed(
        DbConnection connection,
        ConnectionEndEventData eventData)
    {
        Console.WriteLine($"關閉連線: {connection.Database}");
        base.ConnectionClosed(connection, eventData);
    }
}
```

## Temporal Tables

### 使用時態表追蹤歷史

```csharp
public class ApplicationDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>()
            .ToTable("Products", b => b.IsTemporal(temporal =>
            {
                temporal.HasPeriodStart("ValidFrom");
                temporal.HasPeriodEnd("ValidTo");
                temporal.UseHistoryTable("ProductsHistory");
            }));
    }
}

// 查詢歷史資料
public class ProductService
{
    // 查詢特定時間的資料
    public async Task<Product> GetProductAsOfAsync(int id, DateTime dateTime)
    {
        return await _context.Products
            .TemporalAsOf(dateTime)
            .FirstOrDefaultAsync(p => p.Id == id);
    }

    // 查詢時間範圍內的所有變更
    public async Task<List<Product>> GetProductHistoryAsync(
        int id,
        DateTime from,
        DateTime to)
    {
        return await _context.Products
            .TemporalBetween(from, to)
            .Where(p => p.Id == id)
            .OrderBy(p => EF.Property<DateTime>(p, "ValidFrom"))
            .ToListAsync();
    }

    // 查詢所有歷史版本
    public async Task<List<Product>> GetAllProductVersionsAsync(int id)
    {
        return await _context.Products
            .TemporalAll()
            .Where(p => p.Id == id)
            .OrderBy(p => EF.Property<DateTime>(p, "ValidFrom"))
            .ToListAsync();
    }
}
```

## Compiled Queries

### 提升查詢效能

```csharp
public class ProductQueries
{
    private static readonly Func<ApplicationDbContext, int, Task<Product>> 
        _getProductById = EF.CompileAsyncQuery(
            (ApplicationDbContext context, int id) =>
                context.Products.FirstOrDefault(p => p.Id == id));

    private static readonly Func<ApplicationDbContext, string, IAsyncEnumerable<Product>>
        _searchProducts = EF.CompileAsyncQuery(
            (ApplicationDbContext context, string keyword) =>
                context.Products.Where(p => p.Name.Contains(keyword)));

    public async Task<Product> GetProductByIdAsync(int id)
    {
        return await _getProductById(_context, id);
    }

    public IAsyncEnumerable<Product> SearchProductsAsync(string keyword)
    {
        return _searchProducts(_context, keyword);
    }
}
```

## Bulk Operations

### 批次更新

```csharp
public class BulkOperations
{
    // 批次插入
    public async Task BulkInsertAsync(List<Product> products)
    {
        _context.Products.AddRange(products);
        await _context.SaveChangesAsync();
    }

    // 批次更新（使用 EF Plus）
    public async Task BulkUpdatePriceAsync(decimal percentage)
    {
        await _context.Products
            .Where(p => p.Category == "Electronics")
            .UpdateAsync(p => new Product
            {
                Price = p.Price * (1 + percentage / 100)
            });
    }

    // 批次刪除
    public async Task BulkDeleteAsync()
    {
        await _context.Products
            .Where(p => p.Stock == 0)
            .DeleteAsync();
    }

    // 使用原生 SQL
    public async Task BulkUpdateWithSqlAsync(decimal percentage)
    {
        await _context.Database.ExecuteSqlRawAsync(
            "UPDATE Products SET Price = Price * {0} WHERE Category = {1}",
            1 + percentage / 100,
            "Electronics");
    }
}
```

## Change Tracking

### 追蹤實體變更

```csharp
public class ChangeTrackingExample
{
    public async Task TrackChangesAsync()
    {
        var product = await _context.Products.FindAsync(1);
        product.Price = 100;

        // 取得變更資訊
        var entry = _context.Entry(product);

        foreach (var property in entry.Properties)
        {
            if (property.IsModified)
            {
                _logger.LogInformation(
                    "{PropertyName}: {OriginalValue} -> {CurrentValue}",
                    property.Metadata.Name,
                    property.OriginalValue,
                    property.CurrentValue);
            }
        }

        await _context.SaveChangesAsync();
    }

    public void DetachEntity()
    {
        var product = _context.Products.Find(1);
        _context.Entry(product).State = EntityState.Detached;
    }

    public async Task UpdateDetachedEntityAsync(Product product)
    {
        // 附加並標記為已修改
        _context.Attach(product);
        _context.Entry(product).State = EntityState.Modified;

        // 或只更新特定屬性
        _context.Entry(product).Property(p => p.Price).IsModified = true;
        _context.Entry(product).Property(p => p.Stock).IsModified = true;

        await _context.SaveChangesAsync();
    }
}
```

## Database Functions

### 對應資料庫函數

```csharp
public class ApplicationDbContext : DbContext
{
    [DbFunction("GetFullName", "dbo")]
    public static string GetFullName(string firstName, string lastName)
    {
        throw new NotSupportedException();
    }

    [DbFunction("CalculateDiscount", "dbo")]
    public static decimal CalculateDiscount(decimal price, int customerId)
    {
        throw new NotSupportedException();
    }
}

// 使用
public async Task<List<CustomerDto>> GetCustomersAsync()
{
    return await _context.Customers
        .Select(c => new CustomerDto
        {
            Id = c.Id,
            FullName = ApplicationDbContext.GetFullName(c.FirstName, c.LastName)
        })
        .ToListAsync();
}

public async Task<List<Product>> GetDiscountedProductsAsync(int customerId)
{
    return await _context.Products
        .Select(p => new Product
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            DiscountedPrice = ApplicationDbContext.CalculateDiscount(p.Price, customerId)
        })
        .ToListAsync();
}
```

## 效能診斷

### 使用 ToQueryString

```csharp
public class QueryDiagnostics
{
    public void DiagnoseQuery()
    {
        var query = _context.Products
            .Where(p => p.Price > 100)
            .OrderBy(p => p.Name)
            .Take(10);

        // 取得產生的 SQL
        var sql = query.ToQueryString();
        _logger.LogInformation("產生的 SQL: {Sql}", sql);
    }

    public async Task<List<Product>> GetProductsWithLoggingAsync()
    {
        var query = _context.Products
            .Where(p => p.Category == "Electronics");

        _logger.LogInformation("SQL: {Sql}", query.ToQueryString());

        return await query.ToListAsync();
    }
}
```

## 小結

EF Core 進階技巧：
- Global Query Filters 實作軟刪除和多租戶
- Shadow Properties 追蹤審計資訊
- Owned Entity Types 處理值物件
- Interceptors 攔截資料庫操作
- Compiled Queries 提升查詢效能

相較於 Hibernate：
- Query Filters 比 @Where 更靈活
- Shadow Properties 類似動態屬性
- Owned Types 比 @Embeddable 更強大
- Temporal Tables 原生支援時態資料

下週將探討 .NET Core 中的記憶體管理和效能優化。
