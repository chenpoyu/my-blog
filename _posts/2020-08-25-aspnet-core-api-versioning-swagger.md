---
layout: post
title: "ASP.NET Core API 版本控制與文件產生"
date: 2020-08-25 14:15:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, API Versioning, Swagger, OpenAPI]
---

這週研究 ASP.NET Core 的 API 版本控制策略，以及如何使用 Swagger/OpenAPI 自動產生 API 文件。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## API 版本控制

### 安裝套件

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Versioning
dotnet add package Microsoft.AspNetCore.Mvc.Versioning.ApiExplorer
```

### URL 路徑版本控制

```csharp
// Startup.cs
services.AddApiVersioning(options =>
{
    // 預設版本
    options.DefaultApiVersion = new ApiVersion(1, 0);
    // 未指定版本時使用預設版本
    options.AssumeDefaultVersionWhenUnspecified = true;
    // 在 Response Header 回傳支援的版本
    options.ReportApiVersions = true;
});

// V1 Controller
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new { version = "1.0", data = "Products V1" });
    }

    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        return Ok(new { version = "1.0", id, name = "Product V1" });
    }
}

// V2 Controller
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new
        {
            version = "2.0",
            data = new[]
            {
                new { id = 1, name = "Product 1", category = "Electronics" },
                new { id = 2, name = "Product 2", category = "Books" }
            }
        });
    }
}
```

請求範例：
- `/api/v1/products` - V1 API
- `/api/v2/products` - V2 API

### Query String 版本控制

```csharp
services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new QueryStringApiVersionReader("api-version");
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
});

[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1()
    {
        return Ok(new { version = "1.0" });
    }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2()
    {
        return Ok(new { version = "2.0" });
    }
}
```

請求範例：
- `/api/products?api-version=1.0`
- `/api/products?api-version=2.0`

### Header 版本控制

```csharp
services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new HeaderApiVersionReader("X-API-Version");
});
```

請求時加入 Header：
```
X-API-Version: 1.0
```

### 組合多種版本控制方式

```csharp
services.AddApiVersioning(options =>
{
    options.ApiVersionReader = ApiVersionReader.Combine(
        new QueryStringApiVersionReader("api-version"),
        new HeaderApiVersionReader("X-API-Version"),
        new MediaTypeApiVersionReader("version")
    );
});
```

### 版本廢棄處理

```csharp
[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1()
    {
        return Ok(new { version = "1.0", deprecated = true });
    }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2()
    {
        return Ok(new { version = "2.0" });
    }
}
```

回應 Header 會包含：
```
api-supported-versions: 1.0, 2.0
api-deprecated-versions: 1.0
```

## Swagger / OpenAPI

### 安裝 Swashbuckle

```bash
dotnet add package Swashbuckle.AspNetCore
```

### 基本設定

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();

    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo
        {
            Title = "My API",
            Version = "v1",
            Description = "API 文件說明",
            Contact = new OpenApiContact
            {
                Name = "開發團隊",
                Email = "dev@example.com",
                Url = new Uri("https://example.com")
            },
            License = new OpenApiLicense
            {
                Name = "MIT License",
                Url = new Uri("https://opensource.org/licenses/MIT")
            }
        });

        // 讀取 XML 註解
        var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        c.IncludeXmlComments(xmlPath);
    });
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
        c.RoutePrefix = string.Empty; // 設定 Swagger UI 在根路徑
    });

    app.UseRouting();
    app.UseAuthorization();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

啟用 XML 註解（在 .csproj 中）：

```xml
<PropertyGroup>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

### 使用 XML 註解

```csharp
/// <summary>
/// 產品管理 API
/// </summary>
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    /// <summary>
    /// 取得所有產品
    /// </summary>
    /// <returns>產品列表</returns>
    /// <response code="200">成功取得產品列表</response>
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<ProductDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }

    /// <summary>
    /// 根據 ID 取得產品
    /// </summary>
    /// <param name="id">產品 ID</param>
    /// <returns>產品詳細資訊</returns>
    /// <response code="200">成功取得產品</response>
    /// <response code="404">找不到產品</response>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        
        if (product == null)
        {
            return NotFound(new { message = $"找不到產品 {id}" });
        }

        return Ok(product);
    }

    /// <summary>
    /// 建立新產品
    /// </summary>
    /// <param name="request">產品資料</param>
    /// <returns>建立的產品</returns>
    /// <response code="201">成功建立產品</response>
    /// <response code="400">請求資料無效</response>
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest request)
    {
        var product = await _productService.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    /// <summary>
    /// 更新產品
    /// </summary>
    /// <param name="id">產品 ID</param>
    /// <param name="request">更新的產品資料</param>
    /// <returns>更新的產品</returns>
    /// <response code="200">成功更新產品</response>
    /// <response code="404">找不到產品</response>
    [HttpPut("{id}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateProductRequest request)
    {
        var product = await _productService.UpdateAsync(id, request);
        return Ok(product);
    }

    /// <summary>
    /// 刪除產品
    /// </summary>
    /// <param name="id">產品 ID</param>
    /// <returns>無內容</returns>
    /// <response code="204">成功刪除產品</response>
    /// <response code="404">找不到產品</response>
    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        await _productService.DeleteAsync(id);
        return NoContent();
    }
}

/// <summary>
/// 建立產品請求
/// </summary>
public class CreateProductRequest
{
    /// <summary>
    /// 產品名稱
    /// </summary>
    /// <example>iPhone 13 Pro</example>
    [Required]
    public string Name { get; set; }

    /// <summary>
    /// 產品價格
    /// </summary>
    /// <example>35900</example>
    [Required]
    [Range(0.01, 999999.99)]
    public decimal Price { get; set; }

    /// <summary>
    /// 庫存數量
    /// </summary>
    /// <example>100</example>
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
}
```

### 多版本 API 文件

```csharp
services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

services.AddVersionedApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
    c.SwaggerDoc("v2", new OpenApiInfo { Title = "My API", Version = "v2" });
});

app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    c.SwaggerEndpoint("/swagger/v2/swagger.json", "My API V2");
});
```

動態產生版本文件：

```csharp
public class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions>
{
    private readonly IApiVersionDescriptionProvider _provider;

    public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider)
    {
        _provider = provider;
    }

    public void Configure(SwaggerGenOptions options)
    {
        foreach (var description in _provider.ApiVersionDescriptions)
        {
            options.SwaggerDoc(
                description.GroupName,
                new OpenApiInfo
                {
                    Title = $"My API {description.ApiVersion}",
                    Version = description.ApiVersion.ToString(),
                    Description = description.IsDeprecated 
                        ? "此版本已廢棄" 
                        : string.Empty
                });
        }
    }
}

// 註冊
services.AddTransient<IConfigureOptions<SwaggerGenOptions>, ConfigureSwaggerOptions>();
services.AddSwaggerGen();
```

### JWT 認證支援

```csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });

    // 加入 JWT 認證
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme. \r\n\r\n " +
                      "Enter 'Bearer' [space] and then your token in the text input below.\r\n\r\n" +
                      "Example: \"Bearer 12345abcdef\"",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

### 自訂 Schema

```csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });

    // 自訂 Schema 名稱
    c.CustomSchemaIds(type => type.FullName);

    // 使用 Pascal Case
    c.DescribeAllParametersInCamelCase();

    // 自訂 Operation ID
    c.CustomOperationIds(apiDesc =>
    {
        return apiDesc.TryGetMethodInfo(out MethodInfo methodInfo)
            ? methodInfo.Name
            : null;
    });

    // 隱藏某些 API
    c.DocInclusionPredicate((docName, apiDesc) =>
    {
        if (!apiDesc.TryGetMethodInfo(out MethodInfo methodInfo))
        {
            return false;
        }

        var versions = methodInfo.DeclaringType
            .GetCustomAttributes<ApiVersionAttribute>(true)
            .SelectMany(attr => attr.Versions);

        return versions.Any(v => $"v{v}" == docName);
    });
});
```

### 加入範例值

```csharp
public class ProductExample : IExamplesProvider<ProductDto>
{
    public ProductDto GetExamples()
    {
        return new ProductDto
        {
            Id = 1,
            Name = "iPhone 13 Pro",
            Price = 35900,
            Stock = 100,
            CategoryName = "Electronics"
        };
    }
}

// 安裝套件
// dotnet add package Swashbuckle.AspNetCore.Filters

services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
    c.ExampleFilters();
});

services.AddSwaggerExamplesFromAssemblyOf<Startup>();

// 在 DTO 使用
[SwaggerSchema(Title = "產品", Description = "產品詳細資訊")]
public class ProductDto
{
    [SwaggerSchema("產品 ID")]
    public int Id { get; set; }

    [SwaggerSchema("產品名稱", Example = "iPhone 13 Pro")]
    public string Name { get; set; }

    [SwaggerSchema("產品價格", Format = "decimal")]
    public decimal Price { get; set; }
}
```

### Response 範例

```csharp
[HttpGet("{id}")]
[SwaggerResponse(200, "成功取得產品", typeof(ProductDto))]
[SwaggerResponse(404, "找不到產品")]
[SwaggerResponseExample(200, typeof(ProductExample))]
public async Task<IActionResult> GetById(int id)
{
    var product = await _productService.GetByIdAsync(id);
    
    if (product == null)
    {
        return NotFound(new { message = $"找不到產品 {id}" });
    }

    return Ok(product);
}
```

### 自訂 Swagger UI

```csharp
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    
    // 自訂標題
    c.DocumentTitle = "My API 文件";
    
    // 預設展開深度
    c.DefaultModelsExpandDepth(2);
    c.DefaultModelExpandDepth(2);
    
    // 預設展開所有操作
    c.DocExpansion(DocExpansion.List);
    
    // 啟用深層連結
    c.EnableDeepLinking();
    
    // 啟用篩選
    c.EnableFilter();
    
    // 顯示請求時間
    c.DisplayRequestDuration();
    
    // 注入自訂 CSS
    c.InjectStylesheet("/swagger-ui/custom.css");
    
    // 注入自訂 JavaScript
    c.InjectJavascript("/swagger-ui/custom.js");
});
```

### 自訂 Swagger 過濾器

```csharp
public class AddHeaderParameterFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        if (operation.Parameters == null)
        {
            operation.Parameters = new List<OpenApiParameter>();
        }

        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "X-Custom-Header",
            In = ParameterLocation.Header,
            Description = "自訂 Header",
            Required = false,
            Schema = new OpenApiSchema
            {
                Type = "string"
            }
        });
    }
}

services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
    c.OperationFilter<AddHeaderParameterFilter>();
});
```

### 隱藏某些 API

```csharp
[ApiExplorerSettings(IgnoreApi = true)]
[HttpGet("internal")]
public IActionResult InternalOnly()
{
    return Ok("This API is not documented");
}
```

### 產生 OpenAPI 規格檔

```csharp
public class SwaggerExportMiddleware
{
    private readonly RequestDelegate _next;

    public SwaggerExportMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path == "/swagger/export")
        {
            var swagger = context.RequestServices
                .GetRequiredService<ISwaggerProvider>();

            var doc = swagger.GetSwagger("v1");

            var json = doc.SerializeAsJson(OpenApiSpecVersion.OpenApi3_0);

            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync(json);
            return;
        }

        await _next(context);
    }
}

app.UseMiddleware<SwaggerExportMiddleware>();
```

## NSwag 替代方案

### 安裝 NSwag

```bash
dotnet add package NSwag.AspNetCore
```

### 設定 NSwag

```csharp
services.AddOpenApiDocument(config =>
{
    config.PostProcess = document =>
    {
        document.Info.Version = "v1";
        document.Info.Title = "My API";
        document.Info.Description = "API 文件說明";
    };
});

app.UseOpenApi();
app.UseSwaggerUi3();
```

NSwag 的優勢：
- 支援產生 TypeScript/C# Client
- 更好的 Schema 推導
- 支援 Response Type 產生

## 小結

ASP.NET Core API 版本控制與文件：
- 多種版本控制策略（URL、Query、Header）
- Swashbuckle 提供完整的 OpenAPI 支援
- XML 註解自動產生文件
- 支援 JWT 認證文件

相較於 Spring Boot：
- API 版本控制設定更簡潔
- Swagger 整合比 SpringFox 容易
- NSwag 提供更多程式碼產生功能
- 自訂 Swagger UI 更靈活

下週將探討 ASP.NET Core 的 gRPC 服務開發。
