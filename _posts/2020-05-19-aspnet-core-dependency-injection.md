---
layout: post
title: "ASP.NET Core 依賴注入深入探討"
date: 2020-05-19 15:20:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, DI, 依賴注入]
---

上週初步接觸了 .NET Core 的依賴注入，本週深入研究 ASP.NET Core 內建的 DI 容器運作機制。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Spring 的依賴注入回顧

在 Spring 中，我們這樣註冊 Bean：

```java
@Service
public class ProductService {
    private final ProductRepository repository;
    
    @Autowired
    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }
}
```

或使用 Java Config：

```java
@Configuration
public class AppConfig {
    @Bean
    public ProductService productService(ProductRepository repository) {
        return new ProductService(repository);
    }
}
```

## ASP.NET Core 的服務註冊

ASP.NET Core 在 `Startup.ConfigureServices` 中註冊服務：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // 註冊介面與實作
    services.AddScoped<IProductService, ProductService>();
    services.AddScoped<IProductRepository, ProductRepository>();
    
    // 註冊具體類別
    services.AddSingleton<EmailService>();
    
    // 使用 Factory 註冊
    services.AddTransient<IOrderService>(provider => 
    {
        var repository = provider.GetRequiredService<IOrderRepository>();
        return new OrderService(repository);
    });
}
```

## 三種生命週期

### Transient (每次注入都建立新實例)

```csharp
services.AddTransient<ITransientService, TransientService>();
```

相當於 Spring 的：
```java
@Scope("prototype")
@Service
public class TransientService { }
```

使用時機：
- 輕量級、無狀態的服務
- 每次呼叫需要獨立實例

### Scoped (每個請求一個實例)

```csharp
services.AddScoped<IScopedService, ScopedService>();
```

Spring 沒有完全對應的概念，但類似 Request Scope：
```java
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Service
public class ScopedService { }
```

使用時機：
- 資料庫 Context (Entity Framework Core)
- 需要在同一請求中共享狀態的服務

### Singleton (整個應用程式生命週期只有一個實例)

```csharp
services.AddSingleton<ISingletonService, SingletonService>();
```

相當於 Spring 預設的 Singleton：
```java
@Service
public class SingletonService { }
```

使用時機：
- 配置物件
- 快取服務
- 無狀態的工具類別

## 實際案例：電商服務架構

### 定義介面

```csharp
public interface IProductRepository
{
    Task<Product> GetByIdAsync(int id);
    Task<List<Product>> GetAllAsync();
    Task AddAsync(Product product);
}

public interface IProductService
{
    Task<ProductDto> GetProductAsync(int id);
    Task<List<ProductDto>> GetAllProductsAsync();
}

public interface ICacheService
{
    T Get<T>(string key);
    void Set<T>(string key, T value, TimeSpan expiration);
}
```

### 實作類別

```csharp
public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;

    public ProductRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Product> GetByIdAsync(int id)
    {
        return await _context.Products.FindAsync(id);
    }

    public async Task<List<Product>> GetAllAsync()
    {
        return await _context.Products.ToListAsync();
    }

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }
}

public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly ICacheService _cache;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IProductRepository repository,
        ICacheService cache,
        ILogger<ProductService> logger)
    {
        _repository = repository;
        _cache = cache;
        _logger = logger;
    }

    public async Task<ProductDto> GetProductAsync(int id)
    {
        var cacheKey = $"product_{id}";
        var cached = _cache.Get<ProductDto>(cacheKey);
        
        if (cached != null)
        {
            _logger.LogInformation("從快取取得產品 {ProductId}", id);
            return cached;
        }

        var product = await _repository.GetByIdAsync(id);
        var dto = MapToDto(product);
        
        _cache.Set(cacheKey, dto, TimeSpan.FromMinutes(10));
        
        return dto;
    }

    private ProductDto MapToDto(Product product)
    {
        return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price
        };
    }
}

public class MemoryCacheService : ICacheService
{
    private readonly IMemoryCache _cache;

    public MemoryCacheService(IMemoryCache cache)
    {
        _cache = cache;
    }

    public T Get<T>(string key)
    {
        return _cache.TryGetValue(key, out T value) ? value : default;
    }

    public void Set<T>(string key, T value, TimeSpan expiration)
    {
        _cache.Set(key, value, expiration);
    }
}
```

### 註冊服務

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // DbContext 使用 Scoped
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("Default")));

    // Repository 使用 Scoped (配合 DbContext)
    services.AddScoped<IProductRepository, ProductRepository>();

    // Service 使用 Scoped
    services.AddScoped<IProductService, ProductService>();

    // Cache 使用 Singleton (全域共享)
    services.AddMemoryCache();
    services.AddSingleton<ICacheService, MemoryCacheService>();

    services.AddControllers();
}
```

### Controller 使用

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var products = await _productService.GetAllProductsAsync();
        return Ok(products);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var product = await _productService.GetProductAsync(id);
        if (product == null)
            return NotFound();
        
        return Ok(product);
    }
}
```

## 手動解析服務

有時需要手動從 DI 容器取得服務：

```csharp
public class MyBackgroundService
{
    private readonly IServiceProvider _serviceProvider;

    public MyBackgroundService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void DoWork()
    {
        // 建立新的 Scope
        using (var scope = _serviceProvider.CreateScope())
        {
            var productService = scope.ServiceProvider
                .GetRequiredService<IProductService>();
            
            // 使用服務...
        }
    }
}
```

這類似 Spring 的：
```java
@Autowired
private ApplicationContext context;

public void doWork() {
    ProductService service = context.getBean(ProductService.class);
}
```

## 第三方 DI 容器

雖然內建 DI 容器已經足夠強大，但也可以替換成其他容器：

```csharp
// 使用 Autofac
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
}

public void ConfigureContainer(ContainerBuilder builder)
{
    builder.RegisterType<ProductRepository>()
        .As<IProductRepository>()
        .InstancePerLifetimeScope();
    
    builder.RegisterType<ProductService>()
        .As<IProductService>()
        .InstancePerLifetimeScope();
}
```

這類似 Spring 使用 Guice 或其他 DI 框架。

## 生命週期陷阱

### 不要在 Singleton 中注入 Scoped 服務

```csharp
// 錯誤示範
services.AddSingleton<MySingletonService>(); // 注入了 Scoped 的 DbContext

public class MySingletonService
{
    private readonly ApplicationDbContext _context; // Scoped
    
    public MySingletonService(ApplicationDbContext context)
    {
        _context = context; // 危險！
    }
}
```

這會導致 DbContext 的生命週期被延長，引發執行緒安全問題。

正確做法：

```csharp
public class MySingletonService
{
    private readonly IServiceProvider _serviceProvider;
    
    public MySingletonService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public void DoWork()
    {
        using (var scope = _serviceProvider.CreateScope())
        {
            var context = scope.ServiceProvider
                .GetRequiredService<ApplicationDbContext>();
            // 使用 context
        }
    }
}
```

## 與 Spring 的對比總結

| 功能 | Spring | ASP.NET Core |
|------|--------|--------------|
| 註冊方式 | @Service, @Component | AddScoped/Transient/Singleton |
| 自動掃描 | @ComponentScan | 無，需手動註冊 |
| 生命週期 | Singleton, Prototype | Singleton, Scoped, Transient |
| 建構子注入 | @Autowired | 預設支援 |
| 手動取得 | ApplicationContext | IServiceProvider |

## 小結

ASP.NET Core 的 DI 系統設計簡潔，但功能完整。相較於 Spring：
- 需要手動註冊每個服務（無自動掃描）
- 生命週期管理更明確
- Scoped 生命週期特別適合 Web 應用
- 內建容器已能滿足大部分需求

下週將探討 ASP.NET Core 的 Middleware 機制。
