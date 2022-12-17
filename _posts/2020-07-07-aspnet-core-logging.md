---
layout: post
title: "ASP.NET Core 日誌記錄系統"
date: 2020-07-07 14:45:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Logging, Serilog]
---

這週研究 ASP.NET Core 的日誌系統，它內建完整的日誌框架，類似 Java 的 SLF4J + Logback。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Java 日誌回顧

在 Spring Boot 中，我們這樣記錄日誌：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class ProductService {
    private static final Logger logger = LoggerFactory.getLogger(ProductService.class);
    
    public Product getProduct(Long id) {
        logger.debug("查詢產品: {}", id);
        
        Product product = repository.findById(id).orElse(null);
        
        if (product == null) {
            logger.warn("找不到產品: {}", id);
        } else {
            logger.info("成功取得產品: {}, 名稱: {}", id, product.getName());
        }
        
        return product;
    }
}
```

## ASP.NET Core 日誌基礎

### 使用 ILogger

```csharp
public class ProductService
{
    private readonly ILogger<ProductService> _logger;
    private readonly ApplicationDbContext _context;

    public ProductService(
        ILogger<ProductService> logger,
        ApplicationDbContext context)
    {
        _logger = logger;
        _context = context;
    }

    public async Task<Product> GetProductAsync(int id)
    {
        _logger.LogDebug("查詢產品: {ProductId}", id);
        
        var product = await _context.Products.FindAsync(id);
        
        if (product == null)
        {
            _logger.LogWarning("找不到產品: {ProductId}", id);
        }
        else
        {
            _logger.LogInformation(
                "成功取得產品: {ProductId}, 名稱: {ProductName}",
                id, product.Name);
        }
        
        return product;
    }
}
```

注意：使用結構化日誌，參數以 `{PropertyName}` 形式標記，而非字串插值。

## 日誌等級

ASP.NET Core 定義六個日誌等級：

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(Order order)
    {
        // Trace: 最詳細的訊息，通常僅用於開發
        _logger.LogTrace("開始建立訂單流程");
        
        // Debug: 除錯訊息
        _logger.LogDebug("訂單資料: {@Order}", order);
        
        // Information: 一般資訊訊息
        _logger.LogInformation("建立訂單: {OrderNumber}", order.OrderNumber);
        
        try
        {
            await SaveOrderAsync(order);
            return order;
        }
        catch (InvalidOperationException ex)
        {
            // Warning: 警告訊息，不影響執行
            _logger.LogWarning(ex,
                "庫存不足，訂單: {OrderNumber}, 產品: {ProductId}",
                order.OrderNumber, ex.Data["ProductId"]);
            throw;
        }
        catch (Exception ex)
        {
            // Error: 錯誤訊息，影響功能執行
            _logger.LogError(ex,
                "建立訂單失敗: {OrderNumber}",
                order.OrderNumber);
            throw;
        }
        finally
        {
            // Critical: 嚴重錯誤，可能導致應用程式崩潰
            if (!await IsSystemHealthyAsync())
            {
                _logger.LogCritical("系統健康狀態異常");
            }
        }
    }
}
```

等級對應：

| .NET Core | Java (SLF4J) | 說明 |
|-----------|--------------|------|
| Trace | TRACE | 最詳細 |
| Debug | DEBUG | 除錯 |
| Information | INFO | 一般資訊 |
| Warning | WARN | 警告 |
| Error | ERROR | 錯誤 |
| Critical | FATAL | 嚴重錯誤 |

## 設定日誌等級

### appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "Microsoft.EntityFrameworkCore": "Warning",
      "MyShopApi": "Debug",
      "MyShopApi.Services.ProductService": "Trace"
    }
  }
}
```

### 環境特定設定

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}

// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Warning",
      "MyShopApi": "Information"
    }
  }
}
```

## 日誌提供者

ASP.NET Core 支援多個日誌提供者：

### Console Provider (預設)

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureLogging(logging =>
        {
            logging.ClearProviders();
            logging.AddConsole();
            logging.AddDebug();
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

### File Provider (需要第三方套件)

使用 Serilog：

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Console
```

```csharp
// Program.cs
public static void Main(string[] args)
{
    Log.Logger = new LoggerConfiguration()
        .MinimumLevel.Debug()
        .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.File(
            "logs/myapp-.txt",
            rollingInterval: RollingInterval.Day,
            outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
        .CreateLogger();

    try
    {
        Log.Information("啟動應用程式");
        CreateHostBuilder(args).Build().Run();
    }
    catch (Exception ex)
    {
        Log.Fatal(ex, "應用程式啟動失敗");
    }
    finally
    {
        Log.CloseAndFlush();
    }
}

public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseSerilog()
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

Java Logback 對應：

```xml
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/myapp.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/myapp-%d{yyyy-MM-dd}.log</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
</configuration>
```

## 結構化日誌

### 物件序列化

```csharp
public class ProductService
{
    public async Task UpdateProductAsync(Product product)
    {
        // 使用 @ 符號序列化物件
        _logger.LogInformation("更新產品: {@Product}", product);
        
        // 輸出: {"Product": {"Id": 1, "Name": "iPhone", "Price": 25900}}
    }
}
```

### 自訂屬性

```csharp
using (var scope = _logger.BeginScope(new Dictionary<string, object>
{
    ["UserId"] = userId,
    ["OrderId"] = orderId,
    ["CorrelationId"] = correlationId
}))
{
    _logger.LogInformation("處理訂單");
    // 所有在這個 scope 內的日誌都會包含這些屬性
    
    await ProcessOrderAsync(orderId);
}
```

### Serilog 進階設定

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .Enrich.WithProperty("Application", "MyShopApi")
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.File(
        "logs/myapp-.json",
        rollingInterval: RollingInterval.Day,
        formatter: new JsonFormatter())
    .CreateLogger();
```

## 日誌過濾

### 依照類別過濾

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureLogging(logging =>
        {
            logging.AddFilter("System", LogLevel.Warning);
            logging.AddFilter("Microsoft", LogLevel.Warning);
            logging.AddFilter("Microsoft.EntityFrameworkCore", LogLevel.Information);
            logging.AddFilter("MyShopApi.Services", LogLevel.Debug);
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

### 自訂過濾器

```csharp
logging.AddFilter((provider, category, logLevel) =>
{
    // 只記錄 Warning 以上的 Microsoft 日誌
    if (category.StartsWith("Microsoft") && logLevel < LogLevel.Warning)
    {
        return false;
    }
    
    // 記錄所有自己的應用程式日誌
    if (category.StartsWith("MyShopApi"))
    {
        return true;
    }
    
    // 預設只記錄 Information 以上
    return logLevel >= LogLevel.Information;
});
```

## HTTP 請求日誌

### 使用 Middleware

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var requestId = Guid.NewGuid().ToString();
        
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["RequestId"] = requestId,
            ["RequestPath"] = context.Request.Path,
            ["RequestMethod"] = context.Request.Method
        }))
        {
            _logger.LogInformation(
                "HTTP {Method} {Path} 開始",
                context.Request.Method,
                context.Request.Path);

            var sw = Stopwatch.StartNew();

            try
            {
                await _next(context);
            }
            finally
            {
                sw.Stop();

                _logger.LogInformation(
                    "HTTP {Method} {Path} 完成，狀態碼: {StatusCode}，耗時: {Elapsed}ms",
                    context.Request.Method,
                    context.Request.Path,
                    context.Response.StatusCode,
                    sw.ElapsedMilliseconds);
            }
        }
    }
}
```

### 使用 Serilog 的 RequestLogging

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = 
            "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
        
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"]);
            
            if (httpContext.User.Identity?.IsAuthenticated == true)
            {
                diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
            }
        };
    });

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

## 效能考量

### 使用 Log 方法變數

```csharp
public class ProductService
{
    private readonly ILogger<ProductService> _logger;
    
    // 定義 Log 方法（避免每次呼叫時建立委派）
    private static readonly Action<ILogger, int, Exception> _getProductLog =
        LoggerMessage.Define<int>(
            LogLevel.Information,
            new EventId(1, nameof(GetProduct)),
            "取得產品: {ProductId}");

    public async Task<Product> GetProduct(int id)
    {
        _getProductLog(_logger, id, null);
        
        return await _context.Products.FindAsync(id);
    }
}
```

### 檢查是否啟用

```csharp
// 避免不必要的字串操作
if (_logger.IsEnabled(LogLevel.Debug))
{
    var detailedInfo = GenerateDetailedInfo(); // 昂貴的操作
    _logger.LogDebug("詳細資訊: {Info}", detailedInfo);
}
```

## 日誌最佳實踐

### 1. 使用結構化日誌

```csharp
// 好的做法
_logger.LogInformation("訂單建立成功: {OrderId}, 金額: {Amount}", orderId, amount);

// 避免字串插值
_logger.LogInformation($"訂單建立成功: {orderId}, 金額: {amount}");
```

### 2. 記錄重要業務事件

```csharp
public async Task<Order> CreateOrderAsync(Order order)
{
    _logger.LogInformation(
        "訂單建立: {@Order}, 使用者: {UserId}",
        order, GetCurrentUserId());
    
    var savedOrder = await _context.Orders.AddAsync(order);
    await _context.SaveChangesAsync();
    
    _logger.LogInformation(
        "訂單儲存成功: {OrderId}, 編號: {OrderNumber}",
        savedOrder.Id, savedOrder.OrderNumber);
    
    return savedOrder;
}
```

### 3. 記錄異常但不重複

```csharp
public async Task<Product> GetProductAsync(int id)
{
    try
    {
        return await _context.Products.FindAsync(id);
    }
    catch (Exception ex)
    {
        // 在最外層記錄異常
        _logger.LogError(ex, "取得產品失敗: {ProductId}", id);
        throw; // 重新拋出，不在上層再次記錄
    }
}
```

### 4. 使用 Scope 關聯日誌

```csharp
using (_logger.BeginScope("OrderId: {OrderId}", orderId))
{
    _logger.LogInformation("驗證訂單");
    ValidateOrder(orderId);
    
    _logger.LogInformation("計算金額");
    CalculateTotal(orderId);
    
    _logger.LogInformation("儲存訂單");
    SaveOrder(orderId);
}
// 所有日誌都會包含 OrderId
```

### 5. 不要記錄敏感資訊

```csharp
public class User
{
    public string Email { get; set; }
    
    [JsonIgnore]
    [NotLogged]
    public string Password { get; set; }
    
    public string CreditCard { get; set; }
}

// 記錄前移除敏感資訊
_logger.LogInformation(
    "使用者註冊: {Email}",
    user.Email); // 不要記錄密碼
```

## 集中式日誌

### 使用 Seq

```bash
dotnet add package Serilog.Sinks.Seq
```

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();
```

### 使用 Application Insights

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddApplicationInsightsTelemetry(
        Configuration["ApplicationInsights:InstrumentationKey"]);
}
```

### 使用 ELK Stack

```bash
dotnet add package Serilog.Sinks.Elasticsearch
```

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
    {
        AutoRegisterTemplate = true,
        IndexFormat = "myapp-logs-{0:yyyy.MM.dd}"
    })
    .CreateLogger();
```

Java 對應使用 Logstash + Elasticsearch + Kibana。

## 小結

ASP.NET Core 的日誌系統：
- 內建完整的日誌框架
- 支援結構化日誌
- 多個日誌提供者
- Serilog 提供強大功能
- 效能優異

相較於 Java (SLF4J + Logback)：
- 內建依賴注入
- 結構化日誌更直覺
- Serilog 比 Logback 更靈活
- 設定更簡潔

下週將探討 ASP.NET Core 的認證與授權。
