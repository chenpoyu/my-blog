---
layout: post
title: "ASP.NET Core Middleware 管道機制"
date: 2020-05-26 14:15:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Middleware, Pipeline]
---

這週研究 ASP.NET Core 的 Middleware 管道機制，它類似 Java 的 Servlet Filter，但設計更加靈活。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Java Servlet Filter 回顧

在 Spring Boot 中，我們使用 Filter 處理請求：

```java
@Component
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, 
                        ServletResponse response, 
                        FilterChain chain) 
            throws IOException, ServletException {
        // 前置處理
        System.out.println("Request received");
        
        // 呼叫下一個 Filter
        chain.doFilter(request, response);
        
        // 後置處理
        System.out.println("Response sent");
    }
}
```

## ASP.NET Core Middleware 概念

Middleware 是組成請求管道的元件，每個 Middleware：
1. 可以在下一個 Middleware 執行前後處理請求
2. 可以決定是否將請求傳遞給下一個 Middleware
3. 按照註冊順序執行

請求管道的流程：
```
Request → MW1 → MW2 → MW3 → Endpoint
            ↓      ↓      ↓        ↓
Response ← MW1 ← MW2 ← MW3 ← Handler
```

## 內建 Middleware

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // 1. 異常處理
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    // 2. HTTPS 重導向
    app.UseHttpsRedirection();

    // 3. 靜態檔案
    app.UseStaticFiles();

    // 4. 路由
    app.UseRouting();

    // 5. 認證
    app.UseAuthentication();

    // 6. 授權
    app.UseAuthorization();

    // 7. 端點
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

順序很重要！錯誤的順序會導致功能失效。

## 自訂 Middleware - 方式一：Use 方法

```csharp
public void Configure(IApplicationBuilder app)
{
    // 使用 Lambda 建立 Middleware
    app.Use(async (context, next) =>
    {
        // 請求處理前
        var start = DateTime.UtcNow;
        var path = context.Request.Path;
        
        Console.WriteLine($"請求開始: {path}");

        // 呼叫下一個 Middleware
        await next.Invoke();

        // 請求處理後
        var duration = DateTime.UtcNow - start;
        Console.WriteLine($"請求結束: {path}, 耗時: {duration.TotalMilliseconds}ms");
    });

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

## 自訂 Middleware - 方式二：類別

### 建立 Middleware 類別

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
        var method = context.Request.Method;
        var path = context.Request.Path;
        var queryString = context.Request.QueryString;
        
        _logger.LogInformation(
            "[{RequestId}] 請求: {Method} {Path}{Query}",
            requestId, method, path, queryString);

        var start = DateTime.UtcNow;

        try
        {
            await _next(context);
            
            var duration = DateTime.UtcNow - start;
            var statusCode = context.Response.StatusCode;
            
            _logger.LogInformation(
                "[{RequestId}] 回應: {StatusCode}, 耗時: {Duration}ms",
                requestId, statusCode, duration.TotalMilliseconds);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "[{RequestId}] 發生錯誤", requestId);
            throw;
        }
    }
}
```

### 註冊 Middleware

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseMiddleware<RequestLoggingMiddleware>();
    
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### 建立擴充方法（慣例）

```csharp
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }
}

// 使用
public void Configure(IApplicationBuilder app)
{
    app.UseRequestLogging();
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

## 實際案例：API Key 驗證

```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private const string ApiKeyHeaderName = "X-Api-Key";

    public ApiKeyMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(
        HttpContext context,
        IConfiguration configuration)
    {
        // 檢查是否為需要驗證的路徑
        if (!context.Request.Path.StartsWithSegments("/api"))
        {
            await _next(context);
            return;
        }

        // 取得 Header 中的 API Key
        if (!context.Request.Headers.TryGetValue(
            ApiKeyHeaderName, out var extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("API Key 遺失");
            return;
        }

        // 驗證 API Key
        var apiKey = configuration.GetValue<string>("ApiKey");
        if (!apiKey.Equals(extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("無效的 API Key");
            return;
        }

        await _next(context);
    }
}

// appsettings.json
{
  "ApiKey": "my-secret-key-12345"
}
```

對應的 Spring Boot 實作：

```java
@Component
public class ApiKeyFilter implements Filter {
    @Value("${api.key}")
    private String apiKey;

    @Override
    public void doFilter(ServletRequest request, 
                        ServletResponse response, 
                        FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        if (!httpRequest.getRequestURI().startsWith("/api")) {
            chain.doFilter(request, response);
            return;
        }

        String extractedApiKey = httpRequest.getHeader("X-Api-Key");
        
        if (extractedApiKey == null || !apiKey.equals(extractedApiKey)) {
            httpResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            httpResponse.getWriter().write("Invalid API Key");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

## 異常處理 Middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "發生未處理的異常");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static Task HandleExceptionAsync(
        HttpContext context, 
        Exception exception)
    {
        var code = exception switch
        {
            ArgumentNullException => StatusCodes.Status400BadRequest,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            KeyNotFoundException => StatusCodes.Status404NotFound,
            _ => StatusCodes.Status500InternalServerError
        };

        var result = JsonSerializer.Serialize(new
        {
            error = exception.Message,
            statusCode = code
        });

        context.Response.ContentType = "application/json";
        context.Response.StatusCode = code;

        return context.Response.WriteAsync(result);
    }
}
```

## 條件式 Middleware

```csharp
public void Configure(IApplicationBuilder app)
{
    // 只對特定路徑套用 Middleware
    app.UseWhen(
        context => context.Request.Path.StartsWithSegments("/api"),
        appBuilder =>
        {
            appBuilder.UseMiddleware<ApiKeyMiddleware>();
        });

    // MapWhen 會建立分支
    app.MapWhen(
        context => context.Request.Path.StartsWithSegments("/admin"),
        appBuilder =>
        {
            appBuilder.UseMiddleware<AdminAuthMiddleware>();
            appBuilder.Run(async context =>
            {
                await context.Response.WriteAsync("Admin Area");
            });
        });

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

## Run 與 Map

### Run - 終止管道

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello World!");
});

// Run 之後的 Middleware 不會被執行
app.UseRouting(); // 永遠不會執行
```

### Map - 分支管道

```csharp
app.Map("/health", appBuilder =>
{
    appBuilder.Run(async context =>
    {
        await context.Response.WriteAsync("Healthy");
    });
});

// /health 的請求不會執行到這裡
app.UseRouting();
```

## Middleware 與 Filter 的差異

| 特性 | ASP.NET Core Middleware | Spring Filter |
|------|------------------------|---------------|
| 執行時機 | 請求管道的任何階段 | Servlet 容器層級 |
| 順序控制 | 註冊順序 | @Order 或 FilterRegistrationBean |
| 短路 | 直接不呼叫 next | 不呼叫 chain.doFilter() |
| 依賴注入 | 建構子注入 | @Autowired |
| 存取服務 | 透過 HttpContext.RequestServices | 透過 ApplicationContext |

## 常見 Middleware 順序

```csharp
public void Configure(IApplicationBuilder app)
{
    // 1. 異常處理（最外層）
    app.UseExceptionHandler("/error");
    app.UseHsts();

    // 2. HTTPS
    app.UseHttpsRedirection();

    // 3. 靜態檔案
    app.UseStaticFiles();

    // 4. Cookie Policy
    app.UseCookiePolicy();

    // 5. 路由
    app.UseRouting();

    // 6. CORS
    app.UseCors();

    // 7. 認證與授權
    app.UseAuthentication();
    app.UseAuthorization();

    // 8. 自訂 Middleware
    app.UseRequestLogging();

    // 9. Session
    app.UseSession();

    // 10. 端點（最內層）
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

## 小結

ASP.NET Core 的 Middleware 機制：
- 比 Servlet Filter 更靈活
- 順序控制直觀
- 易於測試和組合
- 效能優異（直接操作 HttpContext）

Middleware 是 ASP.NET Core 架構的核心，理解其運作原理對後續學習其他功能至關重要。

下週將探討 ASP.NET Core 的路由機制。
