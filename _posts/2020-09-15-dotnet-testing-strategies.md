---
layout: post
title: ".NET Core 測試策略：單元測試與整合測試"
date: 2020-09-15 14:40:00 +0800
categories: [框架, .NET]
tags: [.NET Core, xUnit, Moq, Testing, Integration Tests]
---

這週研究 .NET Core 的測試策略，包含使用 xUnit 進行單元測試和整合測試的實務做法。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## 測試專案建立

### 建立測試專案

```bash
dotnet new xunit -n MyApp.Tests
cd MyApp.Tests
dotnet add reference ../MyApp/MyApp.csproj
```

### 安裝測試套件

```bash
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package Moq
dotnet add package FluentAssertions
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

## 單元測試基礎

### 基本測試結構

```csharp
using Xunit;
using FluentAssertions;

namespace MyApp.Tests
{
    public class CalculatorTests
    {
        [Fact]
        public void Add_TwoNumbers_ReturnsSum()
        {
            // Arrange
            var calculator = new Calculator();
            
            // Act
            var result = calculator.Add(2, 3);
            
            // Assert
            result.Should().Be(5);
        }

        [Theory]
        [InlineData(2, 3, 5)]
        [InlineData(0, 0, 0)]
        [InlineData(-1, 1, 0)]
        [InlineData(10, 20, 30)]
        public void Add_VariousInputs_ReturnsExpectedSum(int a, int b, int expected)
        {
            // Arrange
            var calculator = new Calculator();
            
            // Act
            var result = calculator.Add(a, b);
            
            // Assert
            result.Should().Be(expected);
        }

        [Fact]
        public void Divide_ByZero_ThrowsException()
        {
            // Arrange
            var calculator = new Calculator();
            
            // Act & Assert
            Action act = () => calculator.Divide(10, 0);
            act.Should().Throw<DivideByZeroException>()
                .WithMessage("除數不可為零");
        }
    }
}
```

## 使用 Moq 進行模擬

### Service 層測試

```csharp
public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _mockRepository;
    private readonly Mock<ILogger<ProductService>> _mockLogger;
    private readonly ProductService _service;

    public ProductServiceTests()
    {
        _mockRepository = new Mock<IProductRepository>();
        _mockLogger = new Mock<ILogger<ProductService>>();
        _service = new ProductService(_mockRepository.Object, _mockLogger.Object);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingProduct_ReturnsProduct()
    {
        // Arrange
        var productId = 1;
        var expectedProduct = new Product
        {
            Id = productId,
            Name = "Test Product",
            Price = 100
        };

        _mockRepository
            .Setup(repo => repo.GetByIdAsync(productId))
            .ReturnsAsync(expectedProduct);

        // Act
        var result = await _service.GetByIdAsync(productId);

        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(productId);
        result.Name.Should().Be("Test Product");

        _mockRepository.Verify(repo => repo.GetByIdAsync(productId), Times.Once);
    }

    [Fact]
    public async Task GetByIdAsync_NonExistingProduct_ReturnsNull()
    {
        // Arrange
        var productId = 999;

        _mockRepository
            .Setup(repo => repo.GetByIdAsync(productId))
            .ReturnsAsync((Product)null);

        // Act
        var result = await _service.GetByIdAsync(productId);

        // Assert
        result.Should().BeNull();
    }

    [Fact]
    public async Task CreateAsync_ValidProduct_ReturnsCreatedProduct()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "New Product",
            Price = 200,
            Stock = 50
        };

        var expectedProduct = new Product
        {
            Id = 1,
            Name = request.Name,
            Price = request.Price,
            Stock = request.Stock
        };

        _mockRepository
            .Setup(repo => repo.CreateAsync(It.IsAny<Product>()))
            .ReturnsAsync(expectedProduct);

        // Act
        var result = await _service.CreateAsync(request);

        // Assert
        result.Should().NotBeNull();
        result.Name.Should().Be(request.Name);
        result.Price.Should().Be(request.Price);

        _mockRepository.Verify(
            repo => repo.CreateAsync(It.Is<Product>(p =>
                p.Name == request.Name &&
                p.Price == request.Price &&
                p.Stock == request.Stock)),
            Times.Once);
    }

    [Fact]
    public async Task DeleteAsync_ExistingProduct_DeletesSuccessfully()
    {
        // Arrange
        var productId = 1;
        var product = new Product { Id = productId };

        _mockRepository
            .Setup(repo => repo.GetByIdAsync(productId))
            .ReturnsAsync(product);

        _mockRepository
            .Setup(repo => repo.DeleteAsync(productId))
            .Returns(Task.CompletedTask);

        // Act
        await _service.DeleteAsync(productId);

        // Assert
        _mockRepository.Verify(repo => repo.DeleteAsync(productId), Times.Once);
    }

    [Fact]
    public async Task DeleteAsync_NonExistingProduct_ThrowsException()
    {
        // Arrange
        var productId = 999;

        _mockRepository
            .Setup(repo => repo.GetByIdAsync(productId))
            .ReturnsAsync((Product)null);

        // Act & Assert
        await Assert.ThrowsAsync<NotFoundException>(
            () => _service.DeleteAsync(productId));
    }
}
```

### Controller 測試

```csharp
public class ProductsControllerTests
{
    private readonly Mock<IProductService> _mockService;
    private readonly ProductsController _controller;

    public ProductsControllerTests()
    {
        _mockService = new Mock<IProductService>();
        _controller = new ProductsController(_mockService.Object);
    }

    [Fact]
    public async Task GetById_ExistingProduct_ReturnsOkResult()
    {
        // Arrange
        var productId = 1;
        var product = new ProductDto
        {
            Id = productId,
            Name = "Test Product",
            Price = 100
        };

        _mockService
            .Setup(service => service.GetByIdAsync(productId))
            .ReturnsAsync(product);

        // Act
        var result = await _controller.GetById(productId);

        // Assert
        var okResult = result.Should().BeOfType<OkObjectResult>().Subject;
        var returnedProduct = okResult.Value.Should().BeOfType<ProductDto>().Subject;
        returnedProduct.Id.Should().Be(productId);
    }

    [Fact]
    public async Task GetById_NonExistingProduct_ReturnsNotFound()
    {
        // Arrange
        var productId = 999;

        _mockService
            .Setup(service => service.GetByIdAsync(productId))
            .ReturnsAsync((ProductDto)null);

        // Act
        var result = await _controller.GetById(productId);

        // Assert
        result.Should().BeOfType<NotFoundResult>();
    }

    [Fact]
    public async Task Create_ValidRequest_ReturnsCreatedResult()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "New Product",
            Price = 200
        };

        var createdProduct = new ProductDto
        {
            Id = 1,
            Name = request.Name,
            Price = request.Price
        };

        _mockService
            .Setup(service => service.CreateAsync(request))
            .ReturnsAsync(createdProduct);

        // Act
        var result = await _controller.Create(request);

        // Assert
        var createdResult = result.Should().BeOfType<CreatedAtActionResult>().Subject;
        createdResult.ActionName.Should().Be(nameof(_controller.GetById));
        createdResult.RouteValues["id"].Should().Be(createdProduct.Id);
    }

    [Fact]
    public async Task Create_InvalidModelState_ReturnsBadRequest()
    {
        // Arrange
        var request = new CreateProductRequest();
        _controller.ModelState.AddModelError("Name", "名稱為必填");

        // Act
        var result = await _controller.Create(request);

        // Assert
        result.Should().BeOfType<BadRequestObjectResult>();
    }
}
```

## 整合測試

### WebApplicationFactory 設定

```csharp
public class CustomWebApplicationFactory<TStartup> : WebApplicationFactory<TStartup>
    where TStartup : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 移除原本的 DbContext
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));

            if (descriptor != null)
            {
                services.Remove(descriptor);
            }

            // 使用 In-Memory 資料庫
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb");
            });

            // 建立測試資料
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var scopedServices = scope.ServiceProvider;
            var db = scopedServices.GetRequiredService<ApplicationDbContext>();

            db.Database.EnsureCreated();
            SeedTestData(db);
        });
    }

    private void SeedTestData(ApplicationDbContext db)
    {
        db.Products.AddRange(
            new Product { Id = 1, Name = "Product 1", Price = 100, Stock = 50 },
            new Product { Id = 2, Name = "Product 2", Price = 200, Stock = 30 },
            new Product { Id = 3, Name = "Product 3", Price = 300, Stock = 20 }
        );

        db.SaveChanges();
    }
}
```

### API 整合測試

```csharp
public class ProductsApiTests : IClassFixture<CustomWebApplicationFactory<Startup>>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory<Startup> _factory;

    public ProductsApiTests(CustomWebApplicationFactory<Startup> factory)
    {
        _factory = factory;
        _client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });
    }

    [Fact]
    public async Task GetProducts_ReturnsSuccessAndProducts()
    {
        // Act
        var response = await _client.GetAsync("/api/products");

        // Assert
        response.EnsureSuccessStatusCode();
        response.Content.Headers.ContentType.ToString()
            .Should().Be("application/json; charset=utf-8");

        var content = await response.Content.ReadAsStringAsync();
        var products = JsonSerializer.Deserialize<List<ProductDto>>(content,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

        products.Should().NotBeEmpty();
        products.Should().HaveCount(3);
    }

    [Fact]
    public async Task GetProductById_ExistingId_ReturnsProduct()
    {
        // Act
        var response = await _client.GetAsync("/api/products/1");

        // Assert
        response.EnsureSuccessStatusCode();

        var content = await response.Content.ReadAsStringAsync();
        var product = JsonSerializer.Deserialize<ProductDto>(content,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

        product.Should().NotBeNull();
        product.Id.Should().Be(1);
        product.Name.Should().Be("Product 1");
    }

    [Fact]
    public async Task GetProductById_NonExistingId_ReturnsNotFound()
    {
        // Act
        var response = await _client.GetAsync("/api/products/999");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task CreateProduct_ValidRequest_ReturnsCreated()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "New Product",
            Price = 150,
            Stock = 40
        };

        var content = new StringContent(
            JsonSerializer.Serialize(request),
            Encoding.UTF8,
            "application/json");

        // Act
        var response = await _client.PostAsync("/api/products", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var responseContent = await response.Content.ReadAsStringAsync();
        var product = JsonSerializer.Deserialize<ProductDto>(responseContent,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

        product.Should().NotBeNull();
        product.Name.Should().Be(request.Name);
        product.Price.Should().Be(request.Price);
    }

    [Fact]
    public async Task CreateProduct_InvalidRequest_ReturnsBadRequest()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "", // 無效：名稱為空
            Price = -10 // 無效：價格為負
        };

        var content = new StringContent(
            JsonSerializer.Serialize(request),
            Encoding.UTF8,
            "application/json");

        // Act
        var response = await _client.PostAsync("/api/products", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task UpdateProduct_ValidRequest_ReturnsOk()
    {
        // Arrange
        var request = new UpdateProductRequest
        {
            Name = "Updated Product",
            Price = 180,
            Stock = 35
        };

        var content = new StringContent(
            JsonSerializer.Serialize(request),
            Encoding.UTF8,
            "application/json");

        // Act
        var response = await _client.PutAsync("/api/products/1", content);

        // Assert
        response.EnsureSuccessStatusCode();

        var responseContent = await response.Content.ReadAsStringAsync();
        var product = JsonSerializer.Deserialize<ProductDto>(responseContent,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

        product.Name.Should().Be(request.Name);
        product.Price.Should().Be(request.Price);
    }

    [Fact]
    public async Task DeleteProduct_ExistingId_ReturnsNoContent()
    {
        // Act
        var response = await _client.DeleteAsync("/api/products/1");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // 確認已刪除
        var getResponse = await _client.GetAsync("/api/products/1");
        getResponse.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

## Repository 測試（使用 In-Memory 資料庫）

```csharp
public class ProductRepositoryTests
{
    private readonly ApplicationDbContext _context;
    private readonly ProductRepository _repository;

    public ProductRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        _context = new ApplicationDbContext(options);
        _repository = new ProductRepository(_context);

        SeedTestData();
    }

    private void SeedTestData()
    {
        _context.Products.AddRange(
            new Product { Id = 1, Name = "Product 1", Price = 100, Stock = 50 },
            new Product { Id = 2, Name = "Product 2", Price = 200, Stock = 30 }
        );
        _context.SaveChanges();
    }

    [Fact]
    public async Task GetAllAsync_ReturnsAllProducts()
    {
        // Act
        var products = await _repository.GetAllAsync();

        // Assert
        products.Should().HaveCount(2);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingId_ReturnsProduct()
    {
        // Act
        var product = await _repository.GetByIdAsync(1);

        // Assert
        product.Should().NotBeNull();
        product.Name.Should().Be("Product 1");
    }

    [Fact]
    public async Task CreateAsync_ValidProduct_AddsToDatabase()
    {
        // Arrange
        var newProduct = new Product
        {
            Name = "Product 3",
            Price = 300,
            Stock = 20
        };

        // Act
        var result = await _repository.CreateAsync(newProduct);

        // Assert
        result.Id.Should().BeGreaterThan(0);
        _context.Products.Should().HaveCount(3);
    }

    [Fact]
    public async Task UpdateAsync_ExistingProduct_UpdatesInDatabase()
    {
        // Arrange
        var product = await _repository.GetByIdAsync(1);
        product.Name = "Updated Product";
        product.Price = 150;

        // Act
        await _repository.UpdateAsync(product);

        // Assert
        var updatedProduct = await _repository.GetByIdAsync(1);
        updatedProduct.Name.Should().Be("Updated Product");
        updatedProduct.Price.Should().Be(150);
    }

    [Fact]
    public async Task DeleteAsync_ExistingId_RemovesFromDatabase()
    {
        // Act
        await _repository.DeleteAsync(1);

        // Assert
        _context.Products.Should().HaveCount(1);
        var deletedProduct = await _repository.GetByIdAsync(1);
        deletedProduct.Should().BeNull();
    }

    public void Dispose()
    {
        _context.Database.EnsureDeleted();
        _context.Dispose();
    }
}
```

## 測試組織與命名

### 測試類別組織

```csharp
// 按功能分組
public class ProductService_GetByIdAsync_Tests { }
public class ProductService_CreateAsync_Tests { }
public class ProductService_UpdateAsync_Tests { }

// 使用巢狀類別
public class ProductServiceTests
{
    public class GetByIdAsync
    {
        [Fact]
        public async Task ExistingProduct_ReturnsProduct() { }

        [Fact]
        public async Task NonExistingProduct_ReturnsNull() { }
    }

    public class CreateAsync
    {
        [Fact]
        public async Task ValidProduct_ReturnsCreatedProduct() { }

        [Fact]
        public async Task InvalidProduct_ThrowsException() { }
    }
}
```

### 測試命名慣例

```csharp
// 模式：MethodName_Scenario_ExpectedBehavior

[Fact]
public void Add_TwoPositiveNumbers_ReturnsSum() { }

[Fact]
public void Divide_ByZero_ThrowsDivideByZeroException() { }

[Fact]
public async Task GetByIdAsync_ExistingProduct_ReturnsProduct() { }

[Fact]
public async Task CreateAsync_DuplicateName_ThrowsValidationException() { }
```

## 測試資料建立器

### Test Data Builder Pattern

```csharp
public class ProductBuilder
{
    private int _id = 1;
    private string _name = "Test Product";
    private decimal _price = 100;
    private int _stock = 50;
    private string _category = "Electronics";

    public ProductBuilder WithId(int id)
    {
        _id = id;
        return this;
    }

    public ProductBuilder WithName(string name)
    {
        _name = name;
        return this;
    }

    public ProductBuilder WithPrice(decimal price)
    {
        _price = price;
        return this;
    }

    public ProductBuilder WithStock(int stock)
    {
        _stock = stock;
        return this;
    }

    public ProductBuilder WithCategory(string category)
    {
        _category = category;
        return this;
    }

    public Product Build()
    {
        return new Product
        {
            Id = _id,
            Name = _name,
            Price = _price,
            Stock = _stock,
            Category = _category
        };
    }
}

// 使用
[Fact]
public async Task Test_WithBuilder()
{
    // Arrange
    var product = new ProductBuilder()
        .WithName("iPhone 13")
        .WithPrice(35900)
        .WithStock(100)
        .Build();

    // ...
}
```

## 執行測試

### CLI 執行

```bash
# 執行所有測試
dotnet test

# 執行特定測試專案
dotnet test MyApp.Tests/MyApp.Tests.csproj

# 產生覆蓋率報告
dotnet test /p:CollectCoverage=true

# 篩選測試
dotnet test --filter "FullyQualifiedName~ProductService"
dotnet test --filter "Category=Integration"

# 平行執行
dotnet test --parallel
```

### 測試覆蓋率

```bash
# 安裝 coverlet
dotnet add package coverlet.msbuild

# 產生覆蓋率
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover

# 使用 ReportGenerator 產生 HTML 報告
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:coverage.opencover.xml -targetdir:coverage-report
```

## 小結

.NET Core 測試策略：
- xUnit 提供現代化的測試框架
- Moq 簡化依賴模擬
- WebApplicationFactory 支援完整的整合測試
- FluentAssertions 提供可讀性高的斷言

相較於 JUnit + Mockito：
- xUnit 的 Theory 比 Parameterized 更簡潔
- Moq 語法比 Mockito 更直覺
- WebApplicationFactory 整合測試更容易
- In-Memory 資料庫測試更輕量

下週將探討 Docker 容器化部署策略。
