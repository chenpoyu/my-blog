---
layout: post
title: "ASP.NET Core 路由機制"
date: 2020-06-02 15:45:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Routing, Web API]
---

這週深入研究 ASP.NET Core 的路由系統，它決定如何將 HTTP 請求對應到控制器動作。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Spring MVC 路由回顧

在 Spring MVC 中，路由透過註解定義：

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @GetMapping
    public List<Product> getAll() { }
    
    @GetMapping("/{id}")
    public Product getById(@PathVariable Long id) { }
    
    @PostMapping
    public Product create(@RequestBody Product product) { }
    
    @PutMapping("/{id}")
    public Product update(
        @PathVariable Long id, 
        @RequestBody Product product) { }
    
    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) { }
}
```

## ASP.NET Core 路由類型

ASP.NET Core 提供兩種路由方式：

### 1. Conventional Routing (慣例路由)

在 `Startup.Configure` 中定義：

```csharp
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");
});
```

這種方式主要用於 MVC 應用（返回 View），較少用於 API。

### 2. Attribute Routing (屬性路由)

在 Controller 和 Action 上使用屬性：

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() { }
    
    [HttpGet("{id}")]
    public IActionResult GetById(int id) { }
}
```

這是 API 開發的推薦方式，與 Spring MVC 的註解非常類似。

## 基本路由定義

### Controller 層級路由

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    // GET api/products
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new List<Product>());
    }

    // GET api/products/5
    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        return Ok(new Product { Id = id });
    }

    // POST api/products
    [HttpPost]
    public IActionResult Create([FromBody] Product product)
    {
        return CreatedAtAction(nameof(GetById), 
            new { id = product.Id }, product);
    }

    // PUT api/products/5
    [HttpPut("{id}")]
    public IActionResult Update(int id, [FromBody] Product product)
    {
        return NoContent();
    }

    // DELETE api/products/5
    [HttpDelete("{id}")]
    public IActionResult Delete(int id)
    {
        return NoContent();
    }
}
```

### 使用 [controller] Token

```csharp
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 自動替換為 "api/products"
}

[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    // 自動替換為 "api/orders"
}
```

`[controller]` 會自動使用控制器名稱（去除 "Controller" 後綴）。

## 路由參數

### 路徑參數

```csharp
// GET api/products/5
[HttpGet("{id}")]
public IActionResult GetById(int id)
{
    return Ok(new { id });
}

// GET api/products/5/reviews/3
[HttpGet("{productId}/reviews/{reviewId}")]
public IActionResult GetReview(int productId, int reviewId)
{
    return Ok(new { productId, reviewId });
}
```

對應 Spring 的 `@PathVariable`：
```java
@GetMapping("/{id}")
public ResponseEntity<?> getById(@PathVariable Long id) { }
```

### 查詢參數

```csharp
// GET api/products?page=1&size=10&sortBy=price
[HttpGet]
public IActionResult GetAll(
    [FromQuery] int page = 1,
    [FromQuery] int size = 10,
    [FromQuery] string sortBy = "id")
{
    return Ok(new { page, size, sortBy });
}
```

對應 Spring 的 `@RequestParam`：
```java
@GetMapping
public ResponseEntity<?> getAll(
    @RequestParam(defaultValue = "1") int page,
    @RequestParam(defaultValue = "10") int size) { }
```

### 使用模型綁定查詢參數

```csharp
public class ProductQueryParameters
{
    public int Page { get; set; } = 1;
    public int Size { get; set; } = 10;
    public string SortBy { get; set; } = "id";
    public string Category { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
}

// GET api/products?page=1&size=10&category=電子&minPrice=1000
[HttpGet]
public IActionResult GetAll([FromQuery] ProductQueryParameters parameters)
{
    return Ok(parameters);
}
```

## 路由約束

### 型別約束

```csharp
// id 必須是整數
[HttpGet("{id:int}")]
public IActionResult GetById(int id) { }

// id 必須是 GUID
[HttpGet("{id:guid}")]
public IActionResult GetByGuid(Guid id) { }

// 價格必須是 decimal
[HttpGet("price/{price:decimal}")]
public IActionResult GetByPrice(decimal price) { }
```

### 範圍約束

```csharp
// id 必須 >= 1
[HttpGet("{id:int:min(1)}")]
public IActionResult GetById(int id) { }

// page 必須在 1-100 之間
[HttpGet("page/{page:int:range(1,100)}")]
public IActionResult GetByPage(int page) { }
```

### 長度約束

```csharp
// code 必須是 3 個字元
[HttpGet("code/{code:length(3)}")]
public IActionResult GetByCode(string code) { }

// name 長度 2-20
[HttpGet("name/{name:length(2,20)}")]
public IActionResult GetByName(string name) { }
```

### 正規表示式約束

```csharp
// slug 只能包含小寫字母、數字和連字號
[HttpGet("{slug:regex(^[[a-z0-9-]]+$)}")]
public IActionResult GetBySlug(string slug) { }
```

### 常用約束列表

| 約束 | 說明 | 範例 |
|------|------|------|
| `:int` | 整數 | `{id:int}` |
| `:bool` | 布林值 | `{active:bool}` |
| `:datetime` | 日期時間 | `{date:datetime}` |
| `:decimal` | 十進位數 | `{price:decimal}` |
| `:double` | 雙精度浮點數 | `{value:double}` |
| `:float` | 浮點數 | `{value:float}` |
| `:guid` | GUID | `{id:guid}` |
| `:long` | 長整數 | `{id:long}` |
| `:min(value)` | 最小值 | `{age:int:min(18)}` |
| `:max(value)` | 最大值 | `{age:int:max(120)}` |
| `:range(min,max)` | 範圍 | `{month:int:range(1,12)}` |
| `:alpha` | 英文字母 | `{name:alpha}` |
| `:length(n)` | 固定長度 | `{code:length(6)}` |
| `:minlength(n)` | 最小長度 | `{name:minlength(2)}` |
| `:maxlength(n)` | 最大長度 | `{name:maxlength(50)}` |

## 多重路由

同一個 Action 可以有多個路由：

```csharp
[HttpGet("id/{id:int}")]
[HttpGet("code/{code}")]
public IActionResult Get(int? id, string code)
{
    if (id.HasValue)
        return Ok(new { source = "id", value = id });
    else
        return Ok(new { source = "code", value = code });
}

// GET api/products/id/5
// GET api/products/code/ABC123
```

## 路由名稱

為路由命名，方便產生 URL：

```csharp
[HttpGet("{id}", Name = "GetProduct")]
public IActionResult GetById(int id)
{
    var product = new Product { Id = id, Name = "iPhone" };
    return Ok(product);
}

[HttpPost]
public IActionResult Create([FromBody] Product product)
{
    product.Id = 123;
    
    // 使用路由名稱產生 URL
    return CreatedAtRoute("GetProduct", 
        new { id = product.Id }, 
        product);
    // 返回 Location: api/products/123
}
```

Spring 的對應：
```java
@PostMapping
public ResponseEntity<?> create(@RequestBody Product product) {
    product.setId(123L);
    URI location = ServletUriComponentsBuilder
        .fromCurrentRequest()
        .path("/{id}")
        .buildAndExpand(product.getId())
        .toUri();
    return ResponseEntity.created(location).body(product);
}
```

## 路由順序

當多個路由可能符合時，ASP.NET Core 依照以下順序：

1. 完全符合的路由
2. 有約束的路由
3. 沒有約束的路由

```csharp
// 順序 1: 完全符合
[HttpGet("featured")]
public IActionResult GetFeatured() { }

// 順序 2: 有約束
[HttpGet("{id:int}")]
public IActionResult GetById(int id) { }

// 順序 3: 無約束
[HttpGet("{slug}")]
public IActionResult GetBySlug(string slug) { }

// GET api/products/featured  → GetFeatured()
// GET api/products/123       → GetById(123)
// GET api/products/iphone-13 → GetBySlug("iphone-13")
```

## 區域路由 (Areas)

用於組織大型應用：

```csharp
[Area("Admin")]
[Route("api/admin/[controller]")]
public class ProductsController : ControllerBase
{
    // GET api/admin/products
    [HttpGet]
    public IActionResult GetAll() { }
}

[Area("Public")]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET api/products
    [HttpGet]
    public IActionResult GetAll() { }
}
```

## IActionResult 回應類型

```csharp
[HttpGet("{id}")]
public IActionResult GetById(int id)
{
    var product = GetProduct(id);
    
    if (product == null)
        return NotFound();                    // 404
    
    return Ok(product);                       // 200
}

[HttpPost]
public IActionResult Create([FromBody] Product product)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);        // 400
    
    SaveProduct(product);
    
    return CreatedAtAction(                   // 201
        nameof(GetById),
        new { id = product.Id },
        product);
}

[HttpPut("{id}")]
public IActionResult Update(int id, [FromBody] Product product)
{
    if (id != product.Id)
        return BadRequest();                  // 400
    
    if (!ProductExists(id))
        return NotFound();                    // 404
    
    UpdateProduct(product);
    
    return NoContent();                       // 204
}
```

常用回應方法：

| 方法 | HTTP 狀態碼 | 說明 |
|------|------------|------|
| `Ok()` | 200 | 成功 |
| `Created()` | 201 | 已建立 |
| `CreatedAtAction()` | 201 | 已建立，包含 Location header |
| `NoContent()` | 204 | 成功，無內容 |
| `BadRequest()` | 400 | 錯誤請求 |
| `Unauthorized()` | 401 | 未授權 |
| `Forbidden()` | 403 | 禁止存取 |
| `NotFound()` | 404 | 找不到 |
| `Conflict()` | 409 | 衝突 |
| `StatusCode(int)` | 自訂 | 自訂狀態碼 |

## 路由除錯

啟用詳細路由記錄：

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore.Routing": "Debug",
      "Microsoft.AspNetCore.Mvc": "Debug"
    }
  }
}
```

## 小結

ASP.NET Core 的路由系統：
- 屬性路由與 Spring MVC 註解概念相似
- 路由約束提供更嚴格的型別檢查
- 內建支援 RESTful API 設計
- 路由命名方便產生 URL

相較於 Spring MVC：
- 約束更豐富（Spring 需要自行驗證）
- `[ApiController]` 自動處理模型驗證
- 路由定義更加簡潔

下週將研究 Entity Framework Core，這是 .NET 的 ORM 框架。
