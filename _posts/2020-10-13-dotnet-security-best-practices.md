---
layout: post
title: ".NET Core 安全性最佳實踐"
date: 2020-10-13 14:20:00 +0800
categories: [框架, .NET]
tags: [.NET Core, Security, Authentication, HTTPS]
---

這週研究 .NET Core 應用程式的安全性最佳實踐，包含認證、授權、資料保護和常見安全漏洞防護。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## HTTPS 與 TLS

### 強制使用 HTTPS

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddHttpsRedirection(options =>
    {
        options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
        options.HttpsPort = 443;
    });

    services.AddHsts(options =>
    {
        options.Preload = true;
        options.IncludeSubDomains = true;
        options.MaxAge = TimeSpan.FromDays(365);
    });

    services.AddControllers();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (!env.IsDevelopment())
    {
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    
    app.UseRouting();
    app.UseAuthentication();
    app.UseAuthorization();
    
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### 設定 TLS 版本

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
            webBuilder.ConfigureKestrel(options =>
            {
                options.ConfigureHttpsDefaults(httpsOptions =>
                {
                    httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
                });
            });
        });
```

## 資料保護

### 使用 Data Protection API

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .SetApplicationName("MyApp")
        .PersistKeysToFileSystem(new DirectoryInfo(@"/var/dpkeys"))
        .SetDefaultKeyLifetime(TimeSpan.FromDays(90));

    services.AddControllers();
}
```

### 加密敏感資料

```csharp
public class DataProtectionService
{
    private readonly IDataProtector _protector;

    public DataProtectionService(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("MyApp.Sensitive");
    }

    public string Protect(string plainText)
    {
        return _protector.Protect(plainText);
    }

    public string Unprotect(string cipherText)
    {
        return _protector.Unprotect(cipherText);
    }

    // 有時效性的保護
    public string ProtectWithExpiry(string plainText, TimeSpan lifetime)
    {
        var timeLimitedProtector = _protector.ToTimeLimitedDataProtector();
        return timeLimitedProtector.Protect(plainText, lifetime);
    }

    public string UnprotectWithExpiry(string cipherText)
    {
        var timeLimitedProtector = _protector.ToTimeLimitedDataProtector();
        return timeLimitedProtector.Unprotect(cipherText, out var expiration);
    }
}

// 使用
public class UserService
{
    private readonly DataProtectionService _dataProtection;

    public async Task CreateUserAsync(CreateUserRequest request)
    {
        var user = new User
        {
            Email = request.Email,
            // 加密信用卡號
            CreditCardNumber = _dataProtection.Protect(request.CreditCardNumber)
        };

        await _repository.CreateAsync(user);
    }

    public async Task<UserDto> GetUserAsync(int userId)
    {
        var user = await _repository.GetByIdAsync(userId);

        return new UserDto
        {
            Id = user.Id,
            Email = user.Email,
            // 解密信用卡號
            CreditCardNumber = _dataProtection.Unprotect(user.CreditCardNumber)
        };
    }
}
```

## 密碼處理

### 使用 Identity 密碼雜湊

```csharp
public class PasswordHasher
{
    private readonly IPasswordHasher<User> _passwordHasher;

    public PasswordHasher()
    {
        _passwordHasher = new PasswordHasher<User>();
    }

    public string HashPassword(string password)
    {
        var user = new User();
        return _passwordHasher.HashPassword(user, password);
    }

    public bool VerifyPassword(string hashedPassword, string providedPassword)
    {
        var user = new User();
        var result = _passwordHasher.VerifyHashedPassword(
            user, hashedPassword, providedPassword);

        return result == PasswordVerificationResult.Success ||
               result == PasswordVerificationResult.SuccessRehashNeeded;
    }
}
```

### 自訂密碼強度驗證

```csharp
public class PasswordValidator
{
    public ValidationResult Validate(string password)
    {
        var errors = new List<string>();

        if (password.Length < 12)
        {
            errors.Add("密碼長度至少需要 12 個字元");
        }

        if (!password.Any(char.IsUpper))
        {
            errors.Add("密碼必須包含至少一個大寫字母");
        }

        if (!password.Any(char.IsLower))
        {
            errors.Add("密碼必須包含至少一個小寫字母");
        }

        if (!password.Any(char.IsDigit))
        {
            errors.Add("密碼必須包含至少一個數字");
        }

        if (!password.Any(c => "!@#$%^&*()_+-=[]{}|;:,.<>?".Contains(c)))
        {
            errors.Add("密碼必須包含至少一個特殊字元");
        }

        // 檢查常見弱密碼
        var commonPasswords = new[] { "password", "123456", "qwerty" };
        if (commonPasswords.Any(p => password.ToLower().Contains(p)))
        {
            errors.Add("密碼過於簡單，請使用更複雜的密碼");
        }

        return errors.Any()
            ? ValidationResult.Failure(errors)
            : ValidationResult.Success();
    }
}
```

## SQL Injection 防護

### 使用參數化查詢

```csharp
// 錯誤示範（易受 SQL Injection 攻擊）
public async Task<List<Product>> SearchBad(string keyword)
{
    var sql = $"SELECT * FROM Products WHERE Name LIKE '%{keyword}%'";
    return await _context.Products.FromSqlRaw(sql).ToListAsync();
}

// 正確做法
public async Task<List<Product>> SearchGood(string keyword)
{
    return await _context.Products
        .Where(p => EF.Functions.Like(p.Name, $"%{keyword}%"))
        .ToListAsync();
}

// 或使用參數化查詢
public async Task<List<Product>> SearchWithParameter(string keyword)
{
    return await _context.Products
        .FromSqlRaw("SELECT * FROM Products WHERE Name LIKE {0}", $"%{keyword}%")
        .ToListAsync();
}
```

## XSS 防護

### 輸出編碼

```csharp
using Microsoft.AspNetCore.Html;
using System.Text.Encodings.Web;

public class XssProtection
{
    private readonly HtmlEncoder _htmlEncoder;
    private readonly JavaScriptEncoder _jsEncoder;

    public XssProtection(HtmlEncoder htmlEncoder, JavaScriptEncoder jsEncoder)
    {
        _htmlEncoder = htmlEncoder;
        _jsEncoder = jsEncoder;
    }

    public string EncodeForHtml(string input)
    {
        return _htmlEncoder.Encode(input);
    }

    public string EncodeForJavaScript(string input)
    {
        return _jsEncoder.Encode(input);
    }

    public IHtmlContent SanitizeHtml(string input)
    {
        // 使用 HtmlSanitizer 套件
        var sanitizer = new HtmlSanitizer();
        sanitizer.AllowedTags.Add("p");
        sanitizer.AllowedTags.Add("br");
        sanitizer.AllowedTags.Add("strong");
        sanitizer.AllowedTags.Add("em");

        var sanitized = sanitizer.Sanitize(input);
        return new HtmlString(sanitized);
    }
}
```

### Content Security Policy

```csharp
// Startup.cs
public void Configure(IApplicationBuilder app)
{
    app.Use(async (context, next) =>
    {
        context.Response.Headers.Add("Content-Security-Policy",
            "default-src 'self'; " +
            "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data: https:; " +
            "font-src 'self' data:; " +
            "connect-src 'self' https://api.example.com;");

        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
        context.Response.Headers.Add("X-Frame-Options", "SAMEORIGIN");
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
        context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");

        await next();
    });

    // ... 其他 middleware
}
```

## CSRF 防護

### 使用 AntiForgery Token

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddAntiforgery(options =>
    {
        options.HeaderName = "X-CSRF-TOKEN";
        options.Cookie.Name = "CSRF-TOKEN";
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    });

    services.AddControllers();
}

// Controller
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest request)
    {
        var product = await _productService.CreateAsync(request);
        return Ok(product);
    }
}

// 取得 CSRF Token 端點
[HttpGet("csrf-token")]
public IActionResult GetCsrfToken()
{
    var tokens = _antiforgery.GetAndStoreTokens(HttpContext);
    return Ok(new { token = tokens.RequestToken });
}
```

## 速率限制

### 使用 AspNetCoreRateLimit

```bash
dotnet add package AspNetCoreRateLimit
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    // 載入設定
    services.AddOptions();
    services.AddMemoryCache();

    services.Configure<IpRateLimitOptions>(Configuration.GetSection("IpRateLimiting"));
    services.Configure<IpRateLimitPolicies>(Configuration.GetSection("IpRateLimitPolicies"));

    services.AddSingleton<IIpPolicyStore, MemoryCacheIpPolicyStore>();
    services.AddSingleton<IRateLimitCounterStore, MemoryCacheRateLimitCounterStore>();
    services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();
    services.AddSingleton<IProcessingStrategy, AsyncKeyLockProcessingStrategy>();

    services.AddControllers();
}

public void Configure(IApplicationBuilder app)
{
    app.UseIpRateLimiting();

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

```json
// appsettings.json
{
  "IpRateLimiting": {
    "EnableEndpointRateLimiting": true,
    "StackBlockedRequests": false,
    "RealIpHeader": "X-Real-IP",
    "ClientIdHeader": "X-ClientId",
    "HttpStatusCode": 429,
    "GeneralRules": [
      {
        "Endpoint": "*",
        "Period": "1s",
        "Limit": 10
      },
      {
        "Endpoint": "*",
        "Period": "1m",
        "Limit": 100
      }
    ]
  },
  "IpRateLimitPolicies": {
    "IpRules": [
      {
        "Ip": "127.0.0.1",
        "Rules": [
          {
            "Endpoint": "*",
            "Period": "1s",
            "Limit": 1000
          }
        ]
      }
    ]
  }
}
```

## 敏感資料遮罩

### 日誌中的敏感資料

```csharp
public class SensitiveDataMasker
{
    public string MaskCreditCard(string creditCard)
    {
        if (string.IsNullOrEmpty(creditCard) || creditCard.Length < 4)
        {
            return "****";
        }

        return "****-****-****-" + creditCard.Substring(creditCard.Length - 4);
    }

    public string MaskEmail(string email)
    {
        if (string.IsNullOrEmpty(email) || !email.Contains("@"))
        {
            return "****@****.com";
        }

        var parts = email.Split('@');
        var username = parts[0];
        var domain = parts[1];

        var maskedUsername = username.Length <= 2
            ? "**"
            : username.Substring(0, 2) + new string('*', username.Length - 2);

        return $"{maskedUsername}@{domain}";
    }

    public string MaskPhone(string phone)
    {
        if (string.IsNullOrEmpty(phone) || phone.Length < 4)
        {
            return "****";
        }

        return new string('*', phone.Length - 4) + phone.Substring(phone.Length - 4);
    }
}

// 在日誌中使用
_logger.LogInformation(
    "使用者 {UserId} 更新信用卡: {CreditCard}",
    userId,
    _masker.MaskCreditCard(creditCard));
```

## 安全標頭

### 使用 NWebsec

```bash
dotnet add package NWebsec.AspNetCore.Middleware
```

```csharp
// Startup.cs
public void Configure(IApplicationBuilder app)
{
    app.UseXContentTypeOptions();
    app.UseReferrerPolicy(opts => opts.StrictOriginWhenCrossOrigin());
    app.UseXXssProtection(options => options.EnabledWithBlockMode());
    app.UseXfo(options => options.SameOrigin());

    app.UseCsp(options => options
        .DefaultSources(s => s.Self())
        .ScriptSources(s => s.Self().CustomSources("https://cdn.jsdelivr.net"))
        .StyleSources(s => s.Self().UnsafeInline())
        .ImageSources(s => s.Self().CustomSources("https:", "data:"))
    );

    app.UseRedirectValidation(opts =>
    {
        opts.AllowedDestinations("https://myapp.com");
        opts.AllowSameHostRedirectsToHttps();
    });

    // ... 其他 middleware
}
```

## 機密資訊管理

### 使用 User Secrets

```bash
# 初始化 User Secrets
dotnet user-secrets init

# 設定機密資訊
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=...;Database=...;Password=..."
dotnet user-secrets set "JwtSettings:SecretKey" "your-secret-key-here"
```

### 使用 Azure Key Vault

```bash
dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
dotnet add package Azure.Identity
```

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((context, config) =>
        {
            if (!context.HostingEnvironment.IsDevelopment())
            {
                var builtConfig = config.Build();
                var keyVaultEndpoint = new Uri(builtConfig["KeyVaultEndpoint"]);

                config.AddAzureKeyVault(
                    keyVaultEndpoint,
                    new DefaultAzureCredential());
            }
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

## API 金鑰驗證

### 實作 API Key 中間件

```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;

    public ApiKeyMiddleware(RequestDelegate next, IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 略過健康檢查端點
        if (context.Request.Path.StartsWithSegments("/health"))
        {
            await _next(context);
            return;
        }

        if (!context.Request.Headers.TryGetValue("X-API-Key", out var extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("API Key 缺失");
            return;
        }

        var apiKeys = _configuration.GetSection("ApiKeys").Get<List<string>>();

        if (!apiKeys.Contains(extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("無效的 API Key");
            return;
        }

        await _next(context);
    }
}

// 註冊
app.UseMiddleware<ApiKeyMiddleware>();
```

## 輸入驗證

### 防止惡意輸入

```csharp
public class InputValidator
{
    private static readonly Regex FileNameRegex = new Regex(@"^[a-zA-Z0-9_\-\.]+$");
    private static readonly Regex EmailRegex = new Regex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$");

    public bool IsValidFileName(string fileName)
    {
        if (string.IsNullOrWhiteSpace(fileName))
        {
            return false;
        }

        // 檢查檔名格式
        if (!FileNameRegex.IsMatch(fileName))
        {
            return false;
        }

        // 檢查危險的副檔名
        var dangerousExtensions = new[] { ".exe", ".bat", ".cmd", ".sh", ".ps1" };
        return !dangerousExtensions.Any(ext =>
            fileName.EndsWith(ext, StringComparison.OrdinalIgnoreCase));
    }

    public bool IsValidEmail(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
        {
            return false;
        }

        return EmailRegex.IsMatch(email) && email.Length <= 254;
    }

    public string SanitizePath(string path)
    {
        // 移除路徑遍歷攻擊
        path = path.Replace("..", string.Empty);
        path = path.Replace("./", string.Empty);
        path = path.Replace("..\\", string.Empty);

        return Path.GetFullPath(path);
    }
}
```

## 安全檢查清單

### 部署前檢查

```markdown
## 認證與授權
- [ ] 使用強密碼策略
- [ ] 實作帳號鎖定機制
- [ ] 使用 HTTPS 傳輸認證資訊
- [ ] JWT Token 設定適當的過期時間
- [ ] 實作 Refresh Token 機制

## 資料保護
- [ ] 敏感資料加密
- [ ] 密碼使用雜湊演算法
- [ ] 啟用 Data Protection
- [ ] 設定適當的 Cookie 屬性

## 輸入驗證
- [ ] 驗證所有使用者輸入
- [ ] 使用參數化查詢防止 SQL Injection
- [ ] 實作檔案上傳驗證
- [ ] 限制請求大小

## 輸出編碼
- [ ] HTML 輸出編碼
- [ ] JavaScript 輸出編碼
- [ ] 設定 Content Security Policy
- [ ] 實作 CSRF 防護

## 安全標頭
- [ ] 設定 X-Content-Type-Options
- [ ] 設定 X-Frame-Options
- [ ] 設定 X-XSS-Protection
- [ ] 設定 Strict-Transport-Security

## 速率限制
- [ ] 實作 API 速率限制
- [ ] 防止暴力破解攻擊
- [ ] 設定請求超時

## 錯誤處理
- [ ] 不在錯誤訊息中洩漏敏感資訊
- [ ] 記錄安全相關事件
- [ ] 實作適當的錯誤頁面

## 依賴管理
- [ ] 定期更新套件
- [ ] 掃描已知漏洞
- [ ] 移除未使用的套件

## 監控與日誌
- [ ] 記錄安全事件
- [ ] 設定警報機制
- [ ] 定期審查日誌
```

## 小結

.NET Core 安全性最佳實踐：
- 強制使用 HTTPS 和 TLS 1.2+
- 使用 Data Protection API 保護敏感資料
- 防護常見攻擊（SQL Injection、XSS、CSRF）
- 實作速率限制和輸入驗證
- 設定安全標頭和 CSP

相較於 Spring Boot：
- Data Protection API 比 Jasypt 更完整
- 安全標頭設定更靈活
- AspNetCoreRateLimit 比 Bucket4j 更簡單
- User Secrets 和 Key Vault 整合更順暢

下週將探討 .NET Core 中的非同步程式設計模式。
