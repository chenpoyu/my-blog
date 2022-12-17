---
layout: post
title: "ASP.NET Core 設定管理與環境配置"
date: 2020-06-30 15:05:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Configuration, Environment]
---

這週研究 ASP.NET Core 的設定管理系統，它比 Spring Boot 的 properties 更靈活。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Spring Boot 配置回顧

在 Spring Boot 中，我們使用 `application.properties` 或 `application.yml`：

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password

app.name=MyShop
app.version=1.0.0
```

```java
@Value("${app.name}")
private String appName;

@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private String version;
    // getters and setters
}
```

## ASP.NET Core 配置來源

ASP.NET Core 支援多種配置來源：
1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. 環境變數
4. 命令列參數
5. User Secrets (開發環境)
6. Azure Key Vault (生產環境)

優先順序：命令列 > 環境變數 > appsettings.{Environment}.json > appsettings.json

## appsettings.json 基本結構

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyShopDb;User Id=sa;Password=YourPassword123;",
    "RedisConnection": "localhost:6379"
  },
  "AppSettings": {
    "ApplicationName": "MyShop",
    "Version": "1.0.0",
    "EnableCache": true,
    "CacheExpirationMinutes": 30,
    "MaxUploadSizeMB": 10
  },
  "EmailSettings": {
    "SmtpServer": "smtp.gmail.com",
    "SmtpPort": 587,
    "FromEmail": "noreply@myshop.com",
    "FromName": "MyShop"
  }
}
```

## 讀取設定

### 方式一：使用 IConfiguration

```csharp
public class ProductService
{
    private readonly IConfiguration _configuration;

    public ProductService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void DoSomething()
    {
        // 讀取簡單值
        var appName = _configuration["AppSettings:ApplicationName"];
        var version = _configuration["AppSettings:Version"];
        
        // 讀取連線字串
        var connectionString = _configuration.GetConnectionString("DefaultConnection");
        
        // 讀取並轉換型別
        var enableCache = _configuration.GetValue<bool>("AppSettings:EnableCache");
        var cacheExpiration = _configuration.GetValue<int>("AppSettings:CacheExpirationMinutes");
        
        // 使用預設值
        var maxSize = _configuration.GetValue<int>("AppSettings:MaxUploadSizeMB", 5);
    }
}
```

### 方式二：Options Pattern (強烈建議)

定義設定類別：

```csharp
public class AppSettings
{
    public string ApplicationName { get; set; }
    public string Version { get; set; }
    public bool EnableCache { get; set; }
    public int CacheExpirationMinutes { get; set; }
    public int MaxUploadSizeMB { get; set; }
}

public class EmailSettings
{
    public string SmtpServer { get; set; }
    public int SmtpPort { get; set; }
    public string FromEmail { get; set; }
    public string FromName { get; set; }
    public string Username { get; set; }
    public string Password { get; set; }
}
```

註冊設定：

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    // 綁定設定
    services.Configure<AppSettings>(
        Configuration.GetSection("AppSettings"));
    
    services.Configure<EmailSettings>(
        Configuration.GetSection("EmailSettings"));
    
    services.AddControllers();
}
```

使用設定：

```csharp
public class ProductService
{
    private readonly AppSettings _appSettings;
    private readonly EmailSettings _emailSettings;

    public ProductService(
        IOptions<AppSettings> appSettings,
        IOptions<EmailSettings> emailSettings)
    {
        _appSettings = appSettings.Value;
        _emailSettings = emailSettings.Value;
    }

    public void DoSomething()
    {
        var appName = _appSettings.ApplicationName;
        var enableCache = _appSettings.EnableCache;
        
        if (_appSettings.EnableCache)
        {
            // 快取邏輯
        }
    }
}
```

Spring Boot 對應：

```java
@ConfigurationProperties(prefix = "app-settings")
@Component
public class AppSettings {
    private String applicationName;
    private String version;
    private boolean enableCache;
    // getters and setters
}

@Autowired
private AppSettings appSettings;
```

## Options Pattern 的三種介面

### IOptions<T> - 單例模式

```csharp
public class MyService
{
    private readonly AppSettings _settings;

    public MyService(IOptions<AppSettings> options)
    {
        _settings = options.Value;
        // 應用程式啟動時讀取一次，不會重新載入
    }
}
```

### IOptionsSnapshot<T> - Scoped 模式

```csharp
public class MyService
{
    private readonly AppSettings _settings;

    public MyService(IOptionsSnapshot<AppSettings> options)
    {
        _settings = options.Value;
        // 每個請求重新讀取，適合會變動的設定
    }
}
```

### IOptionsMonitor<T> - 即時監控

```csharp
public class MyService
{
    private readonly IOptionsMonitor<AppSettings> _options;

    public MyService(IOptionsMonitor<AppSettings> options)
    {
        _options = options;
        
        // 監聽設定變更
        _options.OnChange(settings =>
        {
            Console.WriteLine($"設定已變更: {settings.ApplicationName}");
        });
    }

    public void DoSomething()
    {
        // 即時取得最新設定
        var currentSettings = _options.CurrentValue;
    }
}
```

## 環境特定設定

### 設定檔案

```
appsettings.json                    # 基礎設定
appsettings.Development.json        # 開發環境
appsettings.Staging.json           # 測試環境
appsettings.Production.json        # 生產環境
```

### appsettings.Development.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Debug"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyShopDb_Dev;User Id=sa;Password=DevPassword123;"
  },
  "AppSettings": {
    "EnableCache": false,
    "DetailedErrors": true
  }
}
```

### appsettings.Production.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-server;Database=MyShopDb;User Id=sa;Password=ProdPassword123;"
  },
  "AppSettings": {
    "EnableCache": true,
    "DetailedErrors": false
  }
}
```

### 設定環境變數

```bash
# Windows
set ASPNETCORE_ENVIRONMENT=Development
set ASPNETCORE_ENVIRONMENT=Production

# macOS/Linux
export ASPNETCORE_ENVIRONMENT=Development
export ASPNETCORE_ENVIRONMENT=Production

# launchSettings.json (開發時使用)
{
  "profiles": {
    "MyShopApi": {
      "commandName": "Project",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      },
      "applicationUrl": "https://localhost:5001;http://localhost:5000"
    }
  }
}
```

### 程式碼中檢查環境

```csharp
public class Startup
{
    private readonly IWebHostEnvironment _env;

    public Startup(IConfiguration configuration, IWebHostEnvironment env)
    {
        Configuration = configuration;
        _env = env;
    }

    public void Configure(IApplicationBuilder app)
    {
        if (_env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
            app.UseSwagger();
            app.UseSwaggerUI();
        }
        else if (_env.IsStaging())
        {
            app.UseExceptionHandler("/error");
        }
        else if (_env.IsProduction())
        {
            app.UseExceptionHandler("/error");
            app.UseHsts();
        }

        // 自訂環境
        if (_env.IsEnvironment("Testing"))
        {
            // 測試環境特定邏輯
        }
    }
}
```

Spring Boot 對應：

```java
@Profile("development")
@Configuration
public class DevelopmentConfig {
    // 開發環境配置
}

@Profile("production")
@Configuration
public class ProductionConfig {
    // 生產環境配置
}
```

## 使用環境變數覆寫設定

環境變數使用雙底線 `__` 表示階層：

```bash
# 覆寫 ConnectionStrings:DefaultConnection
export ConnectionStrings__DefaultConnection="Server=prod;Database=MyDb"

# 覆寫 AppSettings:EnableCache
export AppSettings__EnableCache=true

# 覆寫巢狀設定
export EmailSettings__SmtpServer="smtp.example.com"
export EmailSettings__SmtpPort=587
```

## User Secrets (開發環境)

不要將敏感資訊（密碼、API Key）提交到版控。

### 初始化 User Secrets

```bash
dotnet user-secrets init
```

這會在 `.csproj` 加入：

```xml
<PropertyGroup>
  <UserSecretsId>a1b2c3d4-e5f6-1234-5678-90abcdef1234</UserSecretsId>
</PropertyGroup>
```

### 設定 Secrets

```bash
# 設定值
dotnet user-secrets set "EmailSettings:Password" "mypassword123"
dotnet user-secrets set "ApiKeys:GoogleMaps" "AIzaSyXXXXXXXXXXXXXXXXXXXXXX"

# 列出所有 secrets
dotnet user-secrets list

# 清除特定 secret
dotnet user-secrets remove "EmailSettings:Password"

# 清除所有 secrets
dotnet user-secrets clear
```

Secrets 儲存在使用者目錄：
- Windows: `%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json`
- macOS/Linux: `~/.microsoft/usersecrets/<user_secrets_id>/secrets.json`

程式碼中直接使用，無需修改：

```csharp
var emailPassword = _configuration["EmailSettings:Password"];
var apiKey = _configuration["ApiKeys:GoogleMaps"];
```

## 設定驗證

### 使用 Data Annotations

```csharp
public class AppSettings
{
    [Required]
    [MinLength(3)]
    public string ApplicationName { get; set; }
    
    [Required]
    [RegularExpression(@"^\d+\.\d+\.\d+$")]
    public string Version { get; set; }
    
    [Range(1, 1440)]
    public int CacheExpirationMinutes { get; set; }
    
    [Range(1, 100)]
    public int MaxUploadSizeMB { get; set; }
}
```

### 註冊驗證

```csharp
services.AddOptions<AppSettings>()
    .Bind(Configuration.GetSection("AppSettings"))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // 啟動時驗證
```

### 自訂驗證

```csharp
services.AddOptions<EmailSettings>()
    .Bind(Configuration.GetSection("EmailSettings"))
    .Validate(settings =>
    {
        if (string.IsNullOrEmpty(settings.SmtpServer))
            return false;
        
        if (settings.SmtpPort < 1 || settings.SmtpPort > 65535)
            return false;
        
        return true;
    }, "Email settings are invalid")
    .ValidateOnStart();
```

## 設定重新載入

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((context, config) =>
        {
            config.AddJsonFile("appsettings.json",
                optional: false,
                reloadOnChange: true);  // 啟用重新載入
            
            config.AddJsonFile(
                $"appsettings.{context.HostingEnvironment.EnvironmentName}.json",
                optional: true,
                reloadOnChange: true);
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

監聽變更：

```csharp
public class MyService
{
    public MyService(IOptionsMonitor<AppSettings> optionsMonitor)
    {
        optionsMonitor.OnChange(settings =>
        {
            Console.WriteLine($"設定已變更: EnableCache = {settings.EnableCache}");
        });
    }
}
```

## 從資料庫讀取設定

```csharp
public class DatabaseConfigurationSource : IConfigurationSource
{
    private readonly string _connectionString;

    public DatabaseConfigurationSource(string connectionString)
    {
        _connectionString = connectionString;
    }

    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        return new DatabaseConfigurationProvider(_connectionString);
    }
}

public class DatabaseConfigurationProvider : ConfigurationProvider
{
    private readonly string _connectionString;

    public DatabaseConfigurationProvider(string connectionString)
    {
        _connectionString = connectionString;
    }

    public override void Load()
    {
        using (var connection = new SqlConnection(_connectionString))
        {
            connection.Open();
            var command = new SqlCommand(
                "SELECT ConfigKey, ConfigValue FROM AppConfigs",
                connection);
            
            using (var reader = command.ExecuteReader())
            {
                while (reader.Read())
                {
                    var key = reader.GetString(0);
                    var value = reader.GetString(1);
                    Data[key] = value;
                }
            }
        }
    }
}

// 使用
config.Add(new DatabaseConfigurationSource(connectionString));
```

## 設定最佳實踐

### 1. 使用 Options Pattern

```csharp
// 好的做法
services.Configure<AppSettings>(Configuration.GetSection("AppSettings"));

// 避免直接使用 IConfiguration
public class MyService
{
    private readonly IConfiguration _config; // 不推薦
}
```

### 2. 分組相關設定

```json
{
  "Database": {
    "ConnectionString": "...",
    "CommandTimeout": 30,
    "MaxRetryCount": 3
  },
  "Cache": {
    "Provider": "Redis",
    "ExpirationMinutes": 30,
    "ConnectionString": "..."
  }
}
```

### 3. 使用強型別

```csharp
// 好的做法
private readonly AppSettings _settings;

// 避免魔術字串
var value = _configuration["AppSettings:SomeValue"]; // 不推薦
```

### 4. 敏感資訊不要提交

```json
// appsettings.json (提交到版控)
{
  "EmailSettings": {
    "SmtpServer": "smtp.gmail.com",
    "SmtpPort": 587,
    "FromEmail": "noreply@myshop.com"
    // 密碼使用 User Secrets 或環境變數
  }
}
```

## 小結

ASP.NET Core 的設定系統：
- 支援多種設定來源
- Options Pattern 提供強型別設定
- 環境特定設定分離清楚
- User Secrets 保護敏感資訊
- 支援設定驗證與重新載入

相較於 Spring Boot：
- 更靈活的設定來源組合
- 更好的型別安全
- 內建 Secrets 管理
- 設定階層更清晰

下週將探討 ASP.NET Core 的日誌記錄系統。
