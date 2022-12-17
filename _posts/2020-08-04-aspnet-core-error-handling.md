---
layout: post
title: "ASP.NET Core 錯誤處理與例外管理"
date: 2020-08-04 15:10:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Exception Handling, Error Handling]
---

這週研究 ASP.NET Core 的錯誤處理機制，包含全域例外處理和錯誤回應格式化。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Spring Boot 錯誤處理回顧

在 Spring Boot 中，我們使用 `@ControllerAdvice` 處理全域例外：

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<?> handleNotFound(EntityNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(Map.of("error", ex.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handleGeneral(Exception ex) {
        return ResponseEntity.status(500)
            .body(Map.of("error", "Internal server error"));
    }
}
```

## 開發環境錯誤頁面

```csharp
// Startup.cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        // 開發環境：顯示詳細錯誤
        app.UseDeveloperExceptionPage();
    }
    else
    {
        // 生產環境：使用錯誤處理
        app.UseExceptionHandler("/error");
        app.UseHsts();
    }

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

`UseDeveloperExceptionPage` 會顯示：
- 例外詳細資訊
- 堆疊追蹤
- 查詢字串
- Cookies
- Headers
- Routing

## 全域例外處理 Middleware

### 自訂 Middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IWebHostEnvironment _env;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger,
        IWebHostEnvironment env)
    {
        _next = next;
        _logger = logger;
        _env = env;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "發生未處理的例外");
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";

        var response = exception switch
        {
            ApplicationException appEx => new ErrorResponse
            {
                StatusCode = StatusCodes.Status400BadRequest,
                Message = appEx.Message,
                Details = _env.IsDevelopment() ? appEx.StackTrace : null
            },
            KeyNotFoundException => new ErrorResponse
            {
                StatusCode = StatusCodes.Status404NotFound,
                Message = "找不到請求的資源"
            },
            UnauthorizedAccessException => new ErrorResponse
            {
                StatusCode = StatusCodes.Status401Unauthorized,
                Message = "未授權的存取"
            },
            _ => new ErrorResponse
            {
                StatusCode = StatusCodes.Status500InternalServerError,
                Message = _env.IsDevelopment() 
                    ? exception.Message 
                    : "伺服器發生錯誤",
                Details = _env.IsDevelopment() ? exception.StackTrace : null
            }
        };

        context.Response.StatusCode = response.StatusCode;
        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }
}

public class ErrorResponse
{
    public int StatusCode { get; set; }
    public string Message { get; set; }
    public string Details { get; set; }
}

// 註冊
public void Configure(IApplicationBuilder app)
{
    app.UseMiddleware<ExceptionHandlingMiddleware>();
    
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### 使用 UseExceptionHandler

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseExceptionHandler(errorApp =>
    {
        errorApp.Run(async context =>
        {
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";

            var exceptionHandlerPathFeature =
                context.Features.Get<IExceptionHandlerPathFeature>();

            var exception = exceptionHandlerPathFeature?.Error;

            var response = new ErrorResponse
            {
                StatusCode = context.Response.StatusCode,
                Message = "發生錯誤",
                Path = exceptionHandlerPathFeature?.Path
            };

            if (_env.IsDevelopment())
            {
                response.Details = exception?.ToString();
            }

            await context.Response.WriteAsync(
                JsonSerializer.Serialize(response));
        });
    });

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

## 使用 Problem Details (RFC 7807)

### 安裝套件

```bash
dotnet add package Hellang.Middleware.ProblemDetails
```

### 設定

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddProblemDetails(options =>
    {
        // 自訂例外對應
        options.Map<EntityNotFoundException>(ex => new ProblemDetails
        {
            Status = StatusCodes.Status404NotFound,
            Title = "找不到資源",
            Detail = ex.Message
        });

        options.Map<ValidationException>(ex => new ValidationProblemDetails
        {
            Status = StatusCodes.Status400BadRequest,
            Title = "驗證失敗",
            Detail = ex.Message
        });

        // 開發環境顯示詳細資訊
        options.IncludeExceptionDetails = (ctx, ex) =>
        {
            var env = ctx.RequestServices
                .GetRequiredService<IWebHostEnvironment>();
            return env.IsDevelopment();
        };
    });

    services.AddControllers();
}

public void Configure(IApplicationBuilder app)
{
    app.UseProblemDetails();
    
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

回應格式：

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "找不到資源",
  "status": 404,
  "detail": "找不到 ID 為 123 的產品",
  "traceId": "00-abc123..."
}
```

## 自訂例外類別

```csharp
public class EntityNotFoundException : Exception
{
    public string EntityName { get; }
    public object EntityId { get; }

    public EntityNotFoundException(string entityName, object entityId)
        : base($"找不到 {entityName}，ID: {entityId}")
    {
        EntityName = entityName;
        EntityId = entityId;
    }
}

public class ValidationException : Exception
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("一個或多個驗證錯誤")
    {
        Errors = errors;
    }
}

public class BusinessException : Exception
{
    public string Code { get; }

    public BusinessException(string code, string message)
        : base(message)
    {
        Code = code;
    }
}
```

## Controller 層級例外處理

### 使用 Try-Catch

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        try
        {
            var product = await _productService.GetByIdAsync(id);
            
            if (product == null)
            {
                return NotFound(new { message = $"找不到產品 {id}" });
            }

            return Ok(product);
        }
        catch (DbUpdateException ex)
        {
            _logger.LogError(ex, "資料庫錯誤");
            return StatusCode(500, new { message = "資料庫錯誤" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "未預期的錯誤");
            return StatusCode(500, new { message = "伺服器錯誤" });
        }
    }
}
```

### 使用 Filter

```csharp
public class ApiExceptionFilterAttribute : ExceptionFilterAttribute
{
    private readonly IWebHostEnvironment _env;

    public ApiExceptionFilterAttribute(IWebHostEnvironment env)
    {
        _env = env;
    }

    public override void OnException(ExceptionContext context)
    {
        var exception = context.Exception;

        var problemDetails = exception switch
        {
            EntityNotFoundException notFoundEx => new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "找不到資源",
                Detail = notFoundEx.Message
            },
            ValidationException validationEx => new ValidationProblemDetails(
                validationEx.Errors)
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "驗證失敗"
            },
            BusinessException businessEx => new ProblemDetails
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "業務邏輯錯誤",
                Detail = businessEx.Message,
                Extensions = { ["code"] = businessEx.Code }
            },
            _ => new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "伺服器錯誤",
                Detail = _env.IsDevelopment() 
                    ? exception.Message 
                    : "發生未預期的錯誤"
            }
        };

        context.Result = new ObjectResult(problemDetails)
        {
            StatusCode = problemDetails.Status
        };

        context.ExceptionHandled = true;
    }
}

// 全域註冊
services.AddControllers(options =>
{
    options.Filters.Add<ApiExceptionFilterAttribute>();
});

// 或在 Controller 上使用
[ApiExceptionFilter]
public class ProductsController : ControllerBase
{
}
```

## 模型驗證錯誤

### 自動驗證

```csharp
public class CreateProductRequest
{
    [Required(ErrorMessage = "產品名稱為必填")]
    [StringLength(100, MinimumLength = 3, 
        ErrorMessage = "產品名稱長度需介於 3-100 字元")]
    public string Name { get; set; }

    [Required]
    [Range(0.01, 999999.99, ErrorMessage = "價格需介於 0.01 到 999999.99")]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue, ErrorMessage = "庫存不可為負數")]
    public int Stock { get; set; }
}

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest request)
    {
        // [ApiController] 會自動驗證並回傳 400
        // 如果驗證失敗，不會執行到這裡

        var product = await _productService.CreateAsync(request);
        return Ok(product);
    }
}
```

### 自訂驗證錯誤格式

```csharp
services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var errors = context.ModelState
                .Where(e => e.Value.Errors.Count > 0)
                .ToDictionary(
                    e => e.Key,
                    e => e.Value.Errors.Select(x => x.ErrorMessage).ToArray()
                );

            var problemDetails = new ValidationProblemDetails(errors)
            {
                Status = StatusCodes.Status400BadRequest,
                Title = "驗證失敗",
                Detail = "請求包含無效的欄位"
            };

            return new BadRequestObjectResult(problemDetails);
        };
    });
```

## FluentValidation 整合

### 安裝套件

```bash
dotnet add package FluentValidation.AspNetCore
```

### 定義驗證器

```csharp
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("產品名稱為必填")
            .Length(3, 100).WithMessage("產品名稱長度需介於 3-100 字元");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("價格必須大於 0")
            .LessThanOrEqualTo(999999.99m).WithMessage("價格不可超過 999999.99");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("庫存不可為負數");

        RuleFor(x => x.CategoryId)
            .NotEmpty().WithMessage("類別為必填")
            .MustAsync(async (categoryId, cancellation) =>
            {
                // 自訂非同步驗證
                var exists = await CheckCategoryExistsAsync(categoryId);
                return exists;
            }).WithMessage("指定的類別不存在");
    }

    private async Task<bool> CheckCategoryExistsAsync(int categoryId)
    {
        // 驗證邏輯
        return true;
    }
}

// 註冊
services.AddControllers()
    .AddFluentValidation(fv =>
    {
        fv.RegisterValidatorsFromAssemblyContaining<Startup>();
        fv.ImplicitlyValidateChildProperties = true;
    });
```

## 錯誤日誌記錄

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            var requestId = context.TraceIdentifier;
            var userId = context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            var requestPath = context.Request.Path;
            var requestMethod = context.Request.Method;

            using (_logger.BeginScope(new Dictionary<string, object>
            {
                ["RequestId"] = requestId,
                ["UserId"] = userId,
                ["RequestPath"] = requestPath,
                ["RequestMethod"] = requestMethod
            }))
            {
                _logger.LogError(ex,
                    "處理請求時發生錯誤: {Method} {Path}",
                    requestMethod, requestPath);
            }

            await HandleExceptionAsync(context, ex, requestId);
        }
    }
}
```

## 健康檢查端點

```csharp
services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>();

app.UseEndpoints(endpoints =>
{
    endpoints.MapHealthChecks("/health");
    endpoints.MapControllers();
});
```

自訂健康檢查：

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ApplicationDbContext _context;

    public DatabaseHealthCheck(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _context.Database.CanConnectAsync(cancellationToken);
            return HealthCheckResult.Healthy("資料庫連線正常");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "資料庫連線失敗",
                ex);
        }
    }
}

services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database");
```

## 重試機制

```csharp
public class ProductService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<ProductService> _logger;

    public async Task<Product> GetProductWithRetryAsync(int id)
    {
        var maxRetries = 3;
        var delay = TimeSpan.FromSeconds(1);

        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                return await _context.Products.FindAsync(id);
            }
            catch (Exception ex) when (i < maxRetries - 1)
            {
                _logger.LogWarning(ex,
                    "查詢產品失敗，第 {Attempt} 次重試",
                    i + 1);

                await Task.Delay(delay * (i + 1));
            }
        }

        throw new Exception($"無法取得產品 {id}，已重試 {maxRetries} 次");
    }
}
```

或使用 Polly：

```bash
dotnet add package Polly
```

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt =>
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

await retryPolicy.ExecuteAsync(async () =>
{
    return await _context.Products.FindAsync(id);
});
```

## 錯誤回應標準化

```csharp
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public T Data { get; set; }
    public string Message { get; set; }
    public List<string> Errors { get; set; }

    public static ApiResponse<T> SuccessResponse(T data, string message = null)
    {
        return new ApiResponse<T>
        {
            Success = true,
            Data = data,
            Message = message
        };
    }

    public static ApiResponse<T> ErrorResponse(string message, List<string> errors = null)
    {
        return new ApiResponse<T>
        {
            Success = false,
            Message = message,
            Errors = errors ?? new List<string>()
        };
    }
}

// 使用
[HttpGet("{id}")]
public async Task<IActionResult> GetById(int id)
{
    try
    {
        var product = await _productService.GetByIdAsync(id);
        
        if (product == null)
        {
            return NotFound(ApiResponse<object>.ErrorResponse(
                "找不到產品"));
        }

        return Ok(ApiResponse<Product>.SuccessResponse(
            product, "查詢成功"));
    }
    catch (Exception ex)
    {
        return StatusCode(500, ApiResponse<object>.ErrorResponse(
            "伺服器錯誤", new List<string> { ex.Message }));
    }
}
```

## 小結

ASP.NET Core 的錯誤處理：
- 多層次的錯誤處理機制
- 支援 Problem Details 標準
- 整合驗證框架
- 完整的日誌記錄

相較於 Spring Boot：
- Middleware 比 Filter 更靈活
- Problem Details 內建支援
- 錯誤處理設定更簡潔
- FluentValidation 比 Hibernate Validator 更直覺

下週將探討 ASP.NET Core 的背景任務與定時作業。
