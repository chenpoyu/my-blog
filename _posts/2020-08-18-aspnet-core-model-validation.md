---
layout: post
title: "ASP.NET Core 模型驗證與資料註解"
date: 2020-08-18 15:30:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Model Validation, Data Annotations, FluentValidation]
---

這週深入研究 ASP.NET Core 的模型驗證機制，包含內建的資料註解和 FluentValidation 驗證框架。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Spring Boot 驗證回顧

在 Spring Boot 中，我們使用 Bean Validation (JSR-380)：

```java
public class CreateProductRequest {
    
    @NotBlank(message = "產品名稱不可為空")
    @Size(min = 3, max = 100)
    private String name;
    
    @NotNull
    @DecimalMin("0.01")
    @DecimalMax("999999.99")
    private BigDecimal price;
    
    @Min(0)
    private Integer stock;
}

@RestController
public class ProductController {
    
    @PostMapping("/products")
    public ResponseEntity<?> create(@Valid @RequestBody CreateProductRequest request,
                                   BindingResult result) {
        if (result.hasErrors()) {
            return ResponseEntity.badRequest().body(result.getAllErrors());
        }
        // 處理請求
    }
}
```

## 內建資料註解

### 常用驗證屬性

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateProductRequest
{
    [Required(ErrorMessage = "產品名稱為必填")]
    [StringLength(100, MinimumLength = 3, 
        ErrorMessage = "產品名稱長度需介於 3-100 字元")]
    public string Name { get; set; }

    [Required(ErrorMessage = "價格為必填")]
    [Range(0.01, 999999.99, ErrorMessage = "價格需介於 0.01 到 999999.99")]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue, ErrorMessage = "庫存不可為負數")]
    public int Stock { get; set; }

    [Required]
    [EmailAddress(ErrorMessage = "請輸入有效的電子郵件地址")]
    public string ContactEmail { get; set; }

    [Url(ErrorMessage = "請輸入有效的網址")]
    public string Website { get; set; }

    [Phone(ErrorMessage = "請輸入有效的電話號碼")]
    public string PhoneNumber { get; set; }

    [RegularExpression(@"^[A-Z]{2}\d{6}$", 
        ErrorMessage = "產品代碼格式不正確（例: AB123456）")]
    public string ProductCode { get; set; }

    [DataType(DataType.Date)]
    [Display(Name = "製造日期")]
    public DateTime ManufactureDate { get; set; }

    [Compare(nameof(Price), ErrorMessage = "折扣價不可高於原價")]
    public decimal? DiscountPrice { get; set; }
}
```

### Controller 自動驗證

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(
        [FromBody] CreateProductRequest request)
    {
        // [ApiController] 會自動驗證模型
        // 如果驗證失敗，會自動回傳 400 BadRequest

        var product = await _productService.CreateAsync(request);
        return Ok(product);
    }

    // 手動檢查驗證結果
    [HttpPost("manual")]
    public async Task<IActionResult> CreateManual(
        [FromBody] CreateProductRequest request)
    {
        if (!ModelState.IsValid)
        {
            var errors = ModelState
                .Where(x => x.Value.Errors.Count > 0)
                .ToDictionary(
                    kvp => kvp.Key,
                    kvp => kvp.Value.Errors.Select(e => e.ErrorMessage).ToArray()
                );

            return BadRequest(new { errors });
        }

        var product = await _productService.CreateAsync(request);
        return Ok(product);
    }
}
```

## 自訂驗證屬性

### 簡單自訂屬性

```csharp
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(
        object value, ValidationContext validationContext)
    {
        if (value is DateTime date)
        {
            if (date <= DateTime.Now)
            {
                return new ValidationResult("日期必須是未來的日期");
            }
        }

        return ValidationResult.Success;
    }
}

// 使用
public class CreateEventRequest
{
    [Required]
    [FutureDate]
    public DateTime EventDate { get; set; }
}
```

### 帶參數的自訂屬性

```csharp
public class MinAgeAttribute : ValidationAttribute
{
    private readonly int _minAge;

    public MinAgeAttribute(int minAge)
    {
        _minAge = minAge;
    }

    protected override ValidationResult IsValid(
        object value, ValidationContext validationContext)
    {
        if (value is DateTime birthDate)
        {
            var age = DateTime.Today.Year - birthDate.Year;
            if (birthDate.Date > DateTime.Today.AddYears(-age))
            {
                age--;
            }

            if (age < _minAge)
            {
                return new ValidationResult(
                    $"年齡必須至少 {_minAge} 歲");
            }
        }

        return ValidationResult.Success;
    }
}

// 使用
public class UserRegistrationRequest
{
    [Required]
    [MinAge(18)]
    public DateTime BirthDate { get; set; }
}
```

### 類別層級驗證

```csharp
public class DateRangeAttribute : ValidationAttribute
{
    private readonly string _startDateProperty;
    private readonly string _endDateProperty;

    public DateRangeAttribute(string startDateProperty, string endDateProperty)
    {
        _startDateProperty = startDateProperty;
        _endDateProperty = endDateProperty;
    }

    protected override ValidationResult IsValid(
        object value, ValidationContext validationContext)
    {
        var startDateProperty = validationContext.ObjectType
            .GetProperty(_startDateProperty);
        var endDateProperty = validationContext.ObjectType
            .GetProperty(_endDateProperty);

        if (startDateProperty == null || endDateProperty == null)
        {
            return new ValidationResult("找不到指定的屬性");
        }

        var startDate = (DateTime?)startDateProperty.GetValue(validationContext.ObjectInstance);
        var endDate = (DateTime?)endDateProperty.GetValue(validationContext.ObjectInstance);

        if (startDate.HasValue && endDate.HasValue && startDate > endDate)
        {
            return new ValidationResult("開始日期不可晚於結束日期");
        }

        return ValidationResult.Success;
    }
}

// 使用
[DateRange(nameof(StartDate), nameof(EndDate))]
public class SearchRequest
{
    public DateTime? StartDate { get; set; }
    public DateTime? EndDate { get; set; }
}
```

### 需要 DI 的驗證

```csharp
public class UniqueEmailAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(
        object value, ValidationContext validationContext)
    {
        // 從 DI 容器取得服務
        var dbContext = validationContext
            .GetService<ApplicationDbContext>();

        if (value is string email)
        {
            var exists = dbContext.Users.Any(u => u.Email == email);
            if (exists)
            {
                return new ValidationResult("此電子郵件已被使用");
            }
        }

        return ValidationResult.Success;
    }
}

// 使用
public class UserRegistrationRequest
{
    [Required]
    [EmailAddress]
    [UniqueEmail]
    public string Email { get; set; }
}
```

## FluentValidation

### 安裝套件

```bash
dotnet add package FluentValidation.AspNetCore
```

### 定義驗證器

```csharp
using FluentValidation;

public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    private readonly ApplicationDbContext _context;

    public CreateProductRequestValidator(ApplicationDbContext context)
    {
        _context = context;

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("產品名稱為必填")
            .Length(3, 100).WithMessage("產品名稱長度需介於 3-100 字元")
            .MustAsync(BeUniqueName).WithMessage("產品名稱已存在");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("價格必須大於 0")
            .LessThanOrEqualTo(999999.99m).WithMessage("價格不可超過 999999.99");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("庫存不可為負數");

        RuleFor(x => x.CategoryId)
            .NotEmpty().WithMessage("類別為必填")
            .MustAsync(CategoryExists).WithMessage("指定的類別不存在");

        RuleFor(x => x.DiscountPrice)
            .LessThanOrEqualTo(x => x.Price)
            .When(x => x.DiscountPrice.HasValue)
            .WithMessage("折扣價不可高於原價");

        RuleFor(x => x.ContactEmail)
            .NotEmpty().WithMessage("聯絡信箱為必填")
            .EmailAddress().WithMessage("請輸入有效的電子郵件地址");
    }

    private async Task<bool> BeUniqueName(string name, CancellationToken token)
    {
        return !await _context.Products.AnyAsync(p => p.Name == name, token);
    }

    private async Task<bool> CategoryExists(int categoryId, CancellationToken token)
    {
        return await _context.Categories.AnyAsync(c => c.Id == categoryId, token);
    }
}
```

### 註冊 FluentValidation

```csharp
// Startup.cs
services.AddControllers()
    .AddFluentValidation(fv =>
    {
        // 自動註冊所有驗證器
        fv.RegisterValidatorsFromAssemblyContaining<Startup>();
        
        // 自動驗證子屬性
        fv.ImplicitlyValidateChildProperties = true;
        
        // 禁用內建驗證（只使用 FluentValidation）
        fv.DisableDataAnnotationsValidation = false;
    });
```

### 複雜驗證規則

```csharp
public class UpdateProductRequestValidator : AbstractValidator<UpdateProductRequest>
{
    public UpdateProductRequestValidator()
    {
        // 條件驗證
        When(x => x.IsActive, () =>
        {
            RuleFor(x => x.Stock)
                .GreaterThan(0)
                .WithMessage("啟用的產品庫存必須大於 0");
        });

        // 多欄位驗證
        RuleFor(x => x)
            .Must(x => x.StartDate < x.EndDate)
            .When(x => x.StartDate.HasValue && x.EndDate.HasValue)
            .WithMessage("開始日期必須早於結束日期")
            .WithName("DateRange");

        // 集合驗證
        RuleForEach(x => x.Images)
            .SetValidator(new ProductImageValidator());

        // 巢狀物件驗證
        RuleFor(x => x.Specification)
            .SetValidator(new ProductSpecificationValidator())
            .When(x => x.Specification != null);

        // 正規表達式驗證
        RuleFor(x => x.ProductCode)
            .Matches(@"^[A-Z]{2}\d{6}$")
            .WithMessage("產品代碼格式不正確（例: AB123456）");

        // 自訂驗證
        RuleFor(x => x.Tags)
            .Must(tags => tags == null || tags.Count <= 5)
            .WithMessage("標籤數量不可超過 5 個");
    }
}

public class ProductImageValidator : AbstractValidator<ProductImageDto>
{
    public ProductImageValidator()
    {
        RuleFor(x => x.Url)
            .NotEmpty().WithMessage("圖片網址為必填")
            .Must(BeValidUrl).WithMessage("請輸入有效的圖片網址");

        RuleFor(x => x.DisplayOrder)
            .GreaterThanOrEqualTo(0).WithMessage("顯示順序不可為負數");
    }

    private bool BeValidUrl(string url)
    {
        return Uri.TryCreate(url, UriKind.Absolute, out var uriResult)
            && (uriResult.Scheme == Uri.UriSchemeHttp 
                || uriResult.Scheme == Uri.UriSchemeHttps);
    }
}
```

### 自訂錯誤訊息

```csharp
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        // 使用 placeholder
        RuleFor(x => x.Name)
            .Length(3, 100)
            .WithMessage("產品名稱長度需介於 {MinLength} 到 {MaxLength} 字元，目前長度為 {TotalLength}");

        // 使用多語系
        RuleFor(x => x.Price)
            .GreaterThan(0)
            .WithMessage(x => GetLocalizedMessage("PriceGreaterThanZero"));

        // 自訂錯誤碼
        RuleFor(x => x.CategoryId)
            .NotEmpty()
            .WithErrorCode("CATEGORY_REQUIRED")
            .WithMessage("類別為必填");
    }

    private string GetLocalizedMessage(string key)
    {
        // 從資源檔或資料庫取得多語系訊息
        return key;
    }
}
```

## 手動驗證

### 使用 Validator 手動驗證

```csharp
public class ProductService
{
    private readonly IValidator<CreateProductRequest> _validator;

    public ProductService(IValidator<CreateProductRequest> validator)
    {
        _validator = validator;
    }

    public async Task<Product> CreateAsync(CreateProductRequest request)
    {
        // 手動驗證
        var validationResult = await _validator.ValidateAsync(request);

        if (!validationResult.IsValid)
        {
            var errors = validationResult.Errors
                .Select(e => $"{e.PropertyName}: {e.ErrorMessage}");

            throw new ValidationException(
                string.Join(", ", errors));
        }

        // 建立產品
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            Stock = request.Stock
        };

        return product;
    }
}
```

### 使用 DataAnnotations.Validator

```csharp
public class ManualValidationService
{
    public bool TryValidate<T>(T model, out ICollection<ValidationResult> results)
    {
        var context = new ValidationContext(model);
        results = new List<ValidationResult>();

        return Validator.TryValidateObject(
            model, context, results, validateAllProperties: true);
    }

    public void ValidateAndThrow<T>(T model)
    {
        if (!TryValidate(model, out var results))
        {
            var errors = results.Select(r => r.ErrorMessage);
            throw new ValidationException(string.Join(", ", errors));
        }
    }
}

// 使用
var service = new ManualValidationService();
var request = new CreateProductRequest();

if (!service.TryValidate(request, out var results))
{
    foreach (var error in results)
    {
        Console.WriteLine(error.ErrorMessage);
    }
}
```

## 自訂驗證錯誤回應

### 統一錯誤格式

```csharp
services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var errors = context.ModelState
                .Where(e => e.Value.Errors.Count > 0)
                .Select(e => new
                {
                    Field = e.Key,
                    Messages = e.Value.Errors.Select(x => x.ErrorMessage).ToArray()
                })
                .ToList();

            var response = new
            {
                Success = false,
                Message = "驗證失敗",
                Errors = errors,
                TraceId = context.HttpContext.TraceIdentifier
            };

            return new BadRequestObjectResult(response);
        };
    });
```

### 使用 ProblemDetails

```csharp
services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var problemDetails = new ValidationProblemDetails(context.ModelState)
            {
                Type = "https://example.com/validation-error",
                Title = "驗證錯誤",
                Status = StatusCodes.Status400BadRequest,
                Detail = "請求包含一個或多個驗證錯誤",
                Instance = context.HttpContext.Request.Path
            };

            problemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;

            return new BadRequestObjectResult(problemDetails)
            {
                ContentTypes = { "application/problem+json" }
            };
        };
    });
```

回應範例：

```json
{
  "type": "https://example.com/validation-error",
  "title": "驗證錯誤",
  "status": 400,
  "detail": "請求包含一個或多個驗證錯誤",
  "instance": "/api/products",
  "traceId": "00-abc123...",
  "errors": {
    "Name": ["產品名稱為必填"],
    "Price": ["價格必須大於 0"]
  }
}
```

## 客戶端驗證

### 搭配 jQuery Validation

```html
@model CreateProductViewModel

<form asp-action="Create" method="post">
    <div class="form-group">
        <label asp-for="Name"></label>
        <input asp-for="Name" class="form-control" />
        <span asp-validation-for="Name" class="text-danger"></span>
    </div>

    <div class="form-group">
        <label asp-for="Price"></label>
        <input asp-for="Price" class="form-control" />
        <span asp-validation-for="Price" class="text-danger"></span>
    </div>

    <button type="submit" class="btn btn-primary">建立</button>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

會產生：

```html
<input type="text" 
       id="Name" 
       name="Name" 
       data-val="true" 
       data-val-required="產品名稱為必填"
       data-val-length="產品名稱長度需介於 3-100 字元"
       data-val-length-min="3"
       data-val-length-max="100" />
```

## 效能最佳化

### 快取驗證器

```csharp
public class ValidatorCache
{
    private readonly ConcurrentDictionary<Type, object> _validators = new();
    private readonly IServiceProvider _serviceProvider;

    public ValidatorCache(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IValidator<T> GetValidator<T>()
    {
        return (IValidator<T>)_validators.GetOrAdd(typeof(T), type =>
        {
            return _serviceProvider.GetRequiredService<IValidator<T>>();
        });
    }
}
```

### 批次驗證

```csharp
public async Task<List<ValidationResult>> ValidateBatchAsync<T>(
    IEnumerable<T> items, 
    IValidator<T> validator)
{
    var validationTasks = items.Select(item => 
        validator.ValidateAsync(item));

    var results = await Task.WhenAll(validationTasks);

    return results
        .Where(r => !r.IsValid)
        .SelectMany(r => r.Errors)
        .Select(e => new ValidationResult(e.ErrorMessage, new[] { e.PropertyName }))
        .ToList();
}
```

## 單元測試

```csharp
public class CreateProductRequestValidatorTests
{
    private readonly CreateProductRequestValidator _validator;
    private readonly Mock<ApplicationDbContext> _mockContext;

    public CreateProductRequestValidatorTests()
    {
        _mockContext = new Mock<ApplicationDbContext>();
        _validator = new CreateProductRequestValidator(_mockContext.Object);
    }

    [Fact]
    public async Task Name_WhenEmpty_ShouldHaveValidationError()
    {
        var request = new CreateProductRequest { Name = "" };

        var result = await _validator.ValidateAsync(request);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == nameof(request.Name));
    }

    [Theory]
    [InlineData("AB")]
    [InlineData("A")]
    public async Task Name_WhenTooShort_ShouldHaveValidationError(string name)
    {
        var request = new CreateProductRequest { Name = name };

        var result = await _validator.ValidateAsync(request);

        result.IsValid.Should().BeFalse();
    }

    [Fact]
    public async Task Price_WhenZero_ShouldHaveValidationError()
    {
        var request = new CreateProductRequest { Price = 0 };

        var result = await _validator.ValidateAsync(request);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == nameof(request.Price));
    }
}
```

## 小結

ASP.NET Core 驗證機制：
- 內建 Data Annotations 提供基本驗證
- FluentValidation 提供更強大和靈活的驗證
- 自動整合至 Model Binding
- 支援非同步驗證

相較於 Spring Boot：
- FluentValidation 語法比 Bean Validation 更流暢
- 自訂驗證屬性比 `@Constraint` 簡單
- 錯誤訊息客製化更容易
- 非同步驗證支援更完整

下週將探討 ASP.NET Core 的 API 版本控制與文件產生。
