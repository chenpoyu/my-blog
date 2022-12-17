---
layout: post
title: "Entity Framework Core 基礎入門"
date: 2020-06-09 14:20:00 +0800
categories: [框架, .NET]
tags: [EF Core, ORM, Database]
---

這週開始學習 Entity Framework Core（EF Core），它是 .NET 的 ORM 框架，類似 Java 的 Hibernate/JPA。

> 本文環境：**EF Core 3.1** + **.NET Core 3.1 LTS** + **SQL Server**

## Hibernate/JPA 回顧

在 Java 中，我們這樣定義實體：

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String name;
    
    private BigDecimal price;
    
    // getters and setters
}

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByName(String name);
}
```

## EF Core 基本概念

EF Core 的對應概念：
- **Entity** = JPA Entity
- **DbContext** = JPA EntityManager
- **DbSet<T>** = JPA Repository
- **Migration** = Flyway/Liquibase

## 安裝套件

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

對應 Maven 依賴：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

## 定義實體類別

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    
    // 導覽屬性
    public int CategoryId { get; set; }
    public Category Category { get; set; }
    
    public ICollection<OrderItem> OrderItems { get; set; }
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    public ICollection<Product> Products { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    
    public Order Order { get; set; }
    public Product Product { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal TotalAmount { get; set; }
    
    public ICollection<OrderItem> OrderItems { get; set; }
}
```

相較於 JPA，EF Core 不需要額外的註解，透過慣例自動對應。

## 建立 DbContext

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // 稍後會使用到
    }
}
```

對應 Spring Data JPA：
```java
public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

但 DbContext 更像 JPA 的 EntityManager，結合了 Repository 和 Session 的概念。

## 註冊 DbContext

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(
            Configuration.GetConnectionString("DefaultConnection")));
    
    services.AddControllers();
}
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyShopDb;User Id=sa;Password=YourPassword123;"
  }
}
```

Spring Boot 對應：
```properties
spring.datasource.url=jdbc:sqlserver://localhost;databaseName=MyShopDb
spring.datasource.username=sa
spring.datasource.password=YourPassword123
```

## 基本 CRUD 操作

### Create

```csharp
public class ProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Product> CreateProductAsync(Product product)
    {
        product.CreatedAt = DateTime.UtcNow;
        
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        
        return product;
    }
}
```

JPA 對應：
```java
@Service
public class ProductService {
    @Autowired
    private ProductRepository repository;
    
    public Product createProduct(Product product) {
        product.setCreatedAt(LocalDateTime.now());
        return repository.save(product);
    }
}
```

### Read

```csharp
// 取得單一筆
public async Task<Product> GetProductAsync(int id)
{
    return await _context.Products.FindAsync(id);
}

// 取得所有
public async Task<List<Product>> GetAllProductsAsync()
{
    return await _context.Products.ToListAsync();
}

// 條件查詢
public async Task<List<Product>> GetProductsByCategoryAsync(int categoryId)
{
    return await _context.Products
        .Where(p => p.CategoryId == categoryId)
        .ToListAsync();
}

// 單一結果
public async Task<Product> GetProductByNameAsync(string name)
{
    return await _context.Products
        .FirstOrDefaultAsync(p => p.Name == name);
}
```

JPA 對應：
```java
// 取得單一筆
Optional<Product> product = repository.findById(id);

// 取得所有
List<Product> products = repository.findAll();

// 條件查詢
List<Product> products = repository.findByCategoryId(categoryId);

// 單一結果
Product product = repository.findByName(name);
```

### Update

```csharp
public async Task<Product> UpdateProductAsync(Product product)
{
    product.UpdatedAt = DateTime.UtcNow;
    
    _context.Products.Update(product);
    await _context.SaveChangesAsync();
    
    return product;
}

// 或者只更新特定欄位
public async Task UpdateProductPriceAsync(int id, decimal newPrice)
{
    var product = await _context.Products.FindAsync(id);
    if (product != null)
    {
        product.Price = newPrice;
        product.UpdatedAt = DateTime.UtcNow;
        
        await _context.SaveChangesAsync();
    }
}
```

JPA 對應：
```java
public Product updateProduct(Product product) {
    product.setUpdatedAt(LocalDateTime.now());
    return repository.save(product);
}
```

### Delete

```csharp
public async Task DeleteProductAsync(int id)
{
    var product = await _context.Products.FindAsync(id);
    if (product != null)
    {
        _context.Products.Remove(product);
        await _context.SaveChangesAsync();
    }
}

// 直接刪除（不需要先查詢）
public async Task DeleteProductDirectAsync(int id)
{
    var product = new Product { Id = id };
    _context.Products.Remove(product);
    await _context.SaveChangesAsync();
}
```

JPA 對應：
```java
public void deleteProduct(Long id) {
    repository.deleteById(id);
}
```

## 追蹤狀態

EF Core 會追蹤實體的狀態：

```csharp
var product = await _context.Products.FindAsync(1);
// 狀態: Unchanged

product.Price = 999;
// 狀態: Modified

_context.Products.Remove(product);
// 狀態: Deleted

var newProduct = new Product { Name = "New" };
_context.Products.Add(newProduct);
// 狀態: Added

await _context.SaveChangesAsync();
// 所有變更寫入資料庫
```

這類似 JPA 的 Persistence Context。

## 查詢不追蹤

對於唯讀查詢，可以停用追蹤以提升效能：

```csharp
// 不追蹤
var products = await _context.Products
    .AsNoTracking()
    .ToListAsync();

// 全域設定不追蹤
public class ApplicationDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseQueryTrackingBehavior(
            QueryTrackingBehavior.NoTracking);
    }
}
```

## 載入關聯資料

### Eager Loading (立即載入)

```csharp
// 使用 Include
var products = await _context.Products
    .Include(p => p.Category)
    .ToListAsync();

// 多層關聯
var orders = await _context.Orders
    .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
    .ToListAsync();
```

JPA 對應：
```java
@Query("SELECT p FROM Product p JOIN FETCH p.category")
List<Product> findAllWithCategory();
```

### Explicit Loading (明確載入)

```csharp
var product = await _context.Products.FindAsync(1);

// 載入關聯的 Category
await _context.Entry(product)
    .Reference(p => p.Category)
    .LoadAsync();

// 載入集合
await _context.Entry(product)
    .Collection(p => p.OrderItems)
    .LoadAsync();
```

### Lazy Loading (延遲載入)

需要安裝套件：
```bash
dotnet add package Microsoft.EntityFrameworkCore.Proxies
```

```csharp
// Startup.cs
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseLazyLoadingProxies());

// 導覽屬性必須是 virtual
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    public virtual Category Category { get; set; }
    public virtual ICollection<OrderItem> OrderItems { get; set; }
}

// 使用時自動載入
var product = await _context.Products.FindAsync(1);
var categoryName = product.Category.Name; // 自動查詢資料庫
```

JPA 預設就是 Lazy Loading：
```java
@Entity
public class Product {
    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;
}
```

## 原始 SQL 查詢

```csharp
// 查詢實體
var products = await _context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > {0}", 1000)
    .ToListAsync();

// 執行 SQL 指令
await _context.Database
    .ExecuteSqlRawAsync("UPDATE Products SET Stock = Stock - 1 WHERE Id = {0}", productId);

// 使用插值字串（自動參數化）
var minPrice = 1000;
var products = await _context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}")
    .ToListAsync();
```

JPA 對應：
```java
@Query(value = "SELECT * FROM products WHERE price > ?1", nativeQuery = true)
List<Product> findByPriceGreaterThan(BigDecimal price);
```

## 交易處理

```csharp
public async Task CreateOrderAsync(Order order)
{
    using (var transaction = await _context.Database.BeginTransactionAsync())
    {
        try
        {
            // 建立訂單
            _context.Orders.Add(order);
            await _context.SaveChangesAsync();
            
            // 扣除庫存
            foreach (var item in order.OrderItems)
            {
                var product = await _context.Products.FindAsync(item.ProductId);
                product.Stock -= item.Quantity;
            }
            await _context.SaveChangesAsync();
            
            // 提交交易
            await transaction.CommitAsync();
        }
        catch
        {
            // 回滾交易
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

JPA 對應：
```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);
    
    for (OrderItem item : order.getOrderItems()) {
        Product product = productRepository.findById(item.getProductId()).get();
        product.setStock(product.getStock() - item.getQuantity());
    }
}
```

## 小結

EF Core 與 JPA/Hibernate 的對比：

| 功能 | EF Core | JPA/Hibernate |
|------|---------|---------------|
| 實體定義 | POCO 類別 | @Entity 註解 |
| Context | DbContext | EntityManager |
| 查詢 | LINQ | JPQL/Criteria API |
| 慣例 | 內建 | 需要註解 |
| 追蹤 | 預設追蹤 | Persistence Context |
| 延遲載入 | 需要設定 | 預設開啟 |

EF Core 的設計更加簡潔，不需要太多註解。LINQ 查詢比 JPQL 更直覺。

下週將深入探討 LINQ 查詢語法。
