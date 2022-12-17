---
layout: post
title: ".NET Core 初探：從 Java 到 .NET 的轉換"
date: 2020-05-12 14:35:00 +0800
categories: [框架, .NET]
tags: [.NET Core, C#, 跨平台]
---

專案需要開始使用 .NET Core 技術棧，作為 Java 開發者，需要重新學習一個完全不同的生態系統。本文記錄初次接觸 .NET Core 的基本認知。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0** + **Visual Studio Code**

## .NET Core 是什麼

.NET Core 是微軟推出的開源、跨平台框架，可以在 Windows、macOS、Linux 上執行。與傳統的 .NET Framework 不同，它不再綁定 Windows 系統。

從 Java 的角度來看：
- **.NET Core** ≈ **JVM + Spring Boot**
- **C#** ≈ **Java**
- **NuGet** ≈ **Maven/Gradle**
- **ASP.NET Core** ≈ **Spring MVC**

## 環境建置

### 安裝 .NET SDK

```bash
# macOS (使用 Homebrew)
brew install --cask dotnet-sdk

# 驗證安裝
dotnet --version
# 輸出: 3.1.x
```

### 建立第一個專案

```bash
# 建立 Web API 專案
dotnet new webapi -n MyShopApi
cd MyShopApi

# 執行專案
dotnet run
```

Java 的 `mvn spring-boot:run` 對應到 `dotnet run`。

## 專案結構

```
MyShopApi/
├── Controllers/
│   └── WeatherForecastController.cs
├── Properties/
│   └── launchSettings.json
├── appsettings.json
├── appsettings.Development.json
├── Program.cs
├── Startup.cs
├── MyShopApi.csproj
└── WeatherForecast.cs
```

對比 Spring Boot：
- `Program.cs` = `Application.java` (main 方法)
- `Startup.cs` = `@Configuration` 類別
- `appsettings.json` = `application.properties`
- `.csproj` = `pom.xml`

## Program.cs - 程式入口

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

這相當於 Spring Boot 的：

```java
@SpringBootApplication
public class MyShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyShopApplication.class, args);
    }
}
```

## Startup.cs - 配置中心

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseRouting();
        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

- `ConfigureServices` = Spring 的 `@Bean` 定義
- `Configure` = Spring 的 Middleware/Filter 配置

## 第一個 Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new { message = "Hello from .NET Core" });
    }

    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        return Ok(new { id, name = $"Product {id}" });
    }
}
```

對應的 Spring Controller：

```java
@RestController
@RequestMapping("/api/products")
public class ProductsController {
    
    @GetMapping
    public ResponseEntity<?> getAll() {
        return ResponseEntity.ok(Map.of("message", "Hello from Spring"));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<?> getById(@PathVariable int id) {
        return ResponseEntity.ok(Map.of("id", id, "name", "Product " + id));
    }
}
```

## C# 與 Java 的語法差異

### 屬性 (Property)

C# 有原生的 Property 支援：

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}
```

Java 需要手寫 getter/setter 或用 Lombok：

```java
public class Product {
    private int id;
    private String name;
    private BigDecimal price;
    
    // getter/setter...
}
```

### 命名慣例

| 項目 | Java | C# |
|------|------|-----|
| 類別 | PascalCase | PascalCase |
| 方法 | camelCase | PascalCase |
| 變數 | camelCase | camelCase |
| 常數 | UPPER_CASE | PascalCase |

### 型別

| Java | C# |
|------|-----|
| String | string |
| boolean | bool |
| int | int |
| List<T> | List<T> |
| Map<K,V> | Dictionary<K,V> |

## 依賴注入

.NET Core 內建 DI 容器，不需要額外引入 Spring：

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<IProductService, ProductService>();
    services.AddScoped<IProductRepository, ProductRepository>();
}

// Controller
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }
}
```

生命週期對應：
- `AddTransient` = Spring `@Scope("prototype")`
- `AddScoped` = 每個 HTTP 請求一個實例
- `AddSingleton` = Spring `@Scope("singleton")`

## 套件管理

```bash
# 安裝套件 (類似 Maven dependency)
dotnet add package Newtonsoft.Json

# 還原套件
dotnet restore

# 清理並重建
dotnet clean
dotnet build
```

`.csproj` 檔案：

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Newtonsoft.Json" Version="13.0.1" />
  </ItemGroup>
</Project>
```

## 執行與部署

```bash
# 開發模式執行
dotnet run

# 發布 Release 版本
dotnet publish -c Release -o ./publish

# 執行發布的應用
cd publish
dotnet MyShopApi.dll
```

Java 的 `java -jar app.jar` 對應到 `dotnet app.dll`。

## 小結

初步接觸 .NET Core，發現與 Java/Spring 有許多相似之處：
- 都使用依賴注入
- 都有 MVC 架構
- 都支援 RESTful API
- 都有完整的生態系統

主要差異在於語言特性和工具鏈，但核心觀念是相通的。接下來會深入學習 ASP.NET Core 的各個元件。
