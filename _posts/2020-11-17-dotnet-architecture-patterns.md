---
layout: post
title: ".NET Core 架構設計模式"
date: 2020-11-17 15:15:00 +0800
categories: [框架, .NET]
tags: [.NET Core, Architecture, Clean Architecture, DDD]
---

這週深入探討 .NET Core 應用程式的架構設計模式，包含 Clean Architecture、DDD、CQRS 等企業級架構。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Clean Architecture

### 專案結構

```
MyApp/
├── MyApp.Domain/           # 領域層（核心業務邏輯）
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Interfaces/
│   └── Exceptions/
├── MyApp.Application/      # 應用層（用例）
│   ├── UseCases/
│   ├── DTOs/
│   ├── Interfaces/
│   └── Mappings/
├── MyApp.Infrastructure/   # 基礎設施層
│   ├── Persistence/
│   ├── External/
│   └── Services/
└── MyApp.Web/             # 展示層
    ├── Controllers/
    ├── ViewModels/
    └── Filters/
```

### 領域實體

```csharp
// MyApp.Domain/Entities/Product.cs
public class Product : BaseEntity
{
    public string Name { get; private set; }
    public Money Price { get; private set; }
    public int Stock { get; private set; }
    public ProductStatus Status { get; private set; }

    private Product() { } // EF Core

    public Product(string name, Money price, int stock)
    {
        if (string.IsNullOrWhiteSpace(name))
        {
            throw new DomainException("產品名稱不能為空");
        }

        Name = name;
        Price = price ?? throw new ArgumentNullException(nameof(price));
        Stock = stock;
        Status = ProductStatus.Draft;
    }

    public void UpdatePrice(Money newPrice)
    {
        if (newPrice.Amount < 0)
        {
            throw new DomainException("價格不能為負數");
        }

        Price = newPrice;
    }

    public void Publish()
    {
        if (Status == ProductStatus.Published)
        {
            throw new DomainException("產品已上架");
        }

        if (Stock <= 0)
        {
            throw new DomainException("庫存不足，無法上架");
        }

        Status = ProductStatus.Published;
    }

    public void Unpublish()
    {
        Status = ProductStatus.Draft;
    }

    public void DecreaseStock(int quantity)
    {
        if (quantity <= 0)
        {
            throw new ArgumentException("數量必須大於 0");
        }

        if (Stock < quantity)
        {
            throw new DomainException("庫存不足");
        }

        Stock -= quantity;
    }
}

// 值物件
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency = "TWD")
    {
        if (amount < 0)
        {
            throw new ArgumentException("金額不能為負數");
        }

        Amount = amount;
        Currency = currency;
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
        {
            throw new InvalidOperationException("幣別不同無法相加");
        }

        return new Money(a.Amount + b.Amount, a.Currency);
    }
}
```

### 應用層用例

```csharp
// MyApp.Application/UseCases/Products/CreateProductUseCase.cs
public class CreateProductUseCase
{
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMapper _mapper;

    public CreateProductUseCase(
        IProductRepository repository,
        IUnitOfWork unitOfWork,
        IMapper mapper)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _mapper = mapper;
    }

    public async Task<ProductDto> ExecuteAsync(CreateProductRequest request)
    {
        // 驗證
        if (await _repository.ExistsByNameAsync(request.Name))
        {
            throw new BusinessException("產品名稱已存在");
        }

        // 建立領域物件
        var price = new Money(request.Price);
        var product = new Product(request.Name, price, request.Stock);

        // 儲存
        await _repository.AddAsync(product);
        await _unitOfWork.SaveChangesAsync();

        // 轉換 DTO
        return _mapper.Map<ProductDto>(product);
    }
}

// 請求和回應 DTO
public class CreateProductRequest
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Currency { get; set; }
    public int Stock { get; set; }
    public string Status { get; set; }
}
```

### Repository 實作

```csharp
// MyApp.Domain/Interfaces/IProductRepository.cs
public interface IProductRepository : IRepository<Product>
{
    Task<Product> GetByIdAsync(int id);
    Task<List<Product>> GetPublishedAsync();
    Task<bool> ExistsByNameAsync(string name);
    Task AddAsync(Product product);
}

// MyApp.Infrastructure/Persistence/ProductRepository.cs
public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;

    public ProductRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Product> GetByIdAsync(int id)
    {
        return await _context.Products.FindAsync(id);
    }

    public async Task<List<Product>> GetPublishedAsync()
    {
        return await _context.Products
            .Where(p => p.Status == ProductStatus.Published)
            .ToListAsync();
    }

    public async Task<bool> ExistsByNameAsync(string name)
    {
        return await _context.Products
            .AnyAsync(p => p.Name == name);
    }

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
    }
}
```

## Domain-Driven Design

### 聚合根

```csharp
// 訂單聚合根
public class Order : AggregateRoot
{
    public int CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public DateTime OrderDate { get; private set; }
    public Address ShippingAddress { get; private set; }

    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { } // EF Core

    public Order(int customerId, Address shippingAddress)
    {
        CustomerId = customerId;
        ShippingAddress = shippingAddress ?? throw new ArgumentNullException(nameof(shippingAddress));
        OrderDate = DateTime.UtcNow;
        Status = OrderStatus.Pending;

        AddDomainEvent(new OrderCreatedEvent(this));
    }

    public void AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Pending)
        {
            throw new DomainException("只能在待處理狀態新增項目");
        }

        var existingItem = _items.FirstOrDefault(i => i.ProductId == product.Id);
        if (existingItem != null)
        {
            existingItem.AddQuantity(quantity);
        }
        else
        {
            var item = new OrderItem(product.Id, product.Name, product.Price, quantity);
            _items.Add(item);
        }

        AddDomainEvent(new OrderItemAddedEvent(this, product.Id, quantity));
    }

    public void RemoveItem(int productId)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item == null)
        {
            throw new DomainException("項目不存在");
        }

        _items.Remove(item);
        AddDomainEvent(new OrderItemRemovedEvent(this, productId));
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
        {
            throw new DomainException("只能確認待處理的訂單");
        }

        if (!_items.Any())
        {
            throw new DomainException("訂單沒有項目");
        }

        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(this));
    }

    public Money GetTotal()
    {
        return _items
            .Select(i => i.GetSubtotal())
            .Aggregate((a, b) => a + b);
    }
}

// 訂單項目（實體）
public class OrderItem : Entity
{
    public int OrderId { get; private set; }
    public int ProductId { get; private set; }
    public string ProductName { get; private set; }
    public Money UnitPrice { get; private set; }
    public int Quantity { get; private set; }

    private OrderItem() { }

    public OrderItem(int productId, string productName, Money unitPrice, int quantity)
    {
        if (quantity <= 0)
        {
            throw new ArgumentException("數量必須大於 0");
        }

        ProductId = productId;
        ProductName = productName;
        UnitPrice = unitPrice;
        Quantity = quantity;
    }

    public void AddQuantity(int quantity)
    {
        if (quantity <= 0)
        {
            throw new ArgumentException("數量必須大於 0");
        }

        Quantity += quantity;
    }

    public Money GetSubtotal()
    {
        return new Money(UnitPrice.Amount * Quantity, UnitPrice.Currency);
    }
}
```

### 領域事件

```csharp
// MyApp.Domain/Events/OrderCreatedEvent.cs
public class OrderCreatedEvent : DomainEvent
{
    public Order Order { get; }

    public OrderCreatedEvent(Order order)
    {
        Order = order;
    }
}

// 事件處理器
public class OrderCreatedEventHandler : IDomainEventHandler<OrderCreatedEvent>
{
    private readonly ILogger<OrderCreatedEventHandler> _logger;
    private readonly IEmailService _emailService;

    public async Task Handle(OrderCreatedEvent domainEvent)
    {
        _logger.LogInformation("訂單建立: {OrderId}", domainEvent.Order.Id);

        // 發送通知
        await _emailService.SendOrderConfirmationAsync(domainEvent.Order);
    }
}

// 領域事件發布
public interface IDomainEventDispatcher
{
    Task DispatchAsync(DomainEvent domainEvent);
}

public class DomainEventDispatcher : IDomainEventDispatcher
{
    private readonly IServiceProvider _serviceProvider;

    public async Task DispatchAsync(DomainEvent domainEvent)
    {
        var eventType = domainEvent.GetType();
        var handlerType = typeof(IDomainEventHandler<>).MakeGenericType(eventType);
        
        var handlers = _serviceProvider.GetServices(handlerType);

        foreach (dynamic handler in handlers)
        {
            await handler.Handle((dynamic)domainEvent);
        }
    }
}
```

### 規格模式

```csharp
// 規格介面
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    Expression<Func<T, object>> OrderBy { get; }
    Expression<Func<T, object>> OrderByDescending { get; }
}

// 基礎規格
public abstract class BaseSpecification<T> : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria { get; private set; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>> OrderBy { get; private set; }
    public Expression<Func<T, object>> OrderByDescending { get; private set; }

    protected void AddCriteria(Expression<Func<T, bool>> criteria)
    {
        Criteria = criteria;
    }

    protected void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        Includes.Add(includeExpression);
    }

    protected void ApplyOrderBy(Expression<Func<T, object>> orderByExpression)
    {
        OrderBy = orderByExpression;
    }

    protected void ApplyOrderByDescending(Expression<Func<T, object>> orderByDescendingExpression)
    {
        OrderByDescending = orderByDescendingExpression;
    }
}

// 具體規格
public class PublishedProductsSpecification : BaseSpecification<Product>
{
    public PublishedProductsSpecification(string category = null)
    {
        if (string.IsNullOrEmpty(category))
        {
            AddCriteria(p => p.Status == ProductStatus.Published);
        }
        else
        {
            AddCriteria(p => p.Status == ProductStatus.Published && p.Category == category);
        }

        ApplyOrderBy(p => p.Name);
    }
}

// 使用規格
public class ProductService
{
    private readonly IRepository<Product> _repository;

    public async Task<List<Product>> GetPublishedProductsAsync(string category = null)
    {
        var spec = new PublishedProductsSpecification(category);
        return await _repository.ListAsync(spec);
    }
}
```

## CQRS 模式

### Command 和 Query 分離

```csharp
// Command（寫入）
public class CreateOrderCommand : IRequest<int>
{
    public int CustomerId { get; set; }
    public Address ShippingAddress { get; set; }
    public List<OrderItemDto> Items { get; set; }
}

public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public async Task<int> Handle(CreateOrderCommand command, CancellationToken cancellationToken)
    {
        var order = new Order(command.CustomerId, command.ShippingAddress);

        foreach (var item in command.Items)
        {
            var product = await _productRepository.GetByIdAsync(item.ProductId);
            order.AddItem(product, item.Quantity);
        }

        await _repository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return order.Id;
    }
}

// Query（讀取）
public class GetOrderQuery : IRequest<OrderDto>
{
    public int OrderId { get; set; }
}

public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    private readonly ApplicationDbContext _context;
    private readonly IMapper _mapper;

    public async Task<OrderDto> Handle(GetOrderQuery query, CancellationToken cancellationToken)
    {
        var order = await _context.Orders
            .Include(o => o.Items)
            .AsNoTracking()
            .FirstOrDefaultAsync(o => o.Id == query.OrderId, cancellationToken);

        return _mapper.Map<OrderDto>(order);
    }
}
```

### 使用 MediatR

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddMediatR(typeof(Startup));
    
    // 註冊 Pipeline Behaviors
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
}

// Controller
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<ActionResult<int>> CreateOrder([FromBody] CreateOrderCommand command)
    {
        var orderId = await _mediator.Send(command);
        return CreatedAtAction(nameof(GetOrder), new { id = orderId }, orderId);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<OrderDto>> GetOrder(int id)
    {
        var order = await _mediator.Send(new GetOrderQuery { OrderId = id });
        
        if (order == null)
        {
            return NotFound();
        }

        return Ok(order);
    }
}
```

### Pipeline Behaviors

```csharp
// 驗證 Behavior
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public async Task<TResponse> Handle(
        TRequest request,
        CancellationToken cancellationToken,
        RequestHandlerDelegate<TResponse> next)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);

            var validationResults = await Task.WhenAll(
                _validators.Select(v => v.ValidateAsync(context, cancellationToken)));

            var failures = validationResults
                .SelectMany(r => r.Errors)
                .Where(f => f != null)
                .ToList();

            if (failures.Any())
            {
                throw new ValidationException(failures);
            }
        }

        return await next();
    }
}

// 日誌 Behavior
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public async Task<TResponse> Handle(
        TRequest request,
        CancellationToken cancellationToken,
        RequestHandlerDelegate<TResponse> next)
    {
        _logger.LogInformation("處理 {RequestName}", typeof(TRequest).Name);

        var stopwatch = Stopwatch.StartNew();

        try
        {
            var response = await next();

            stopwatch.Stop();
            _logger.LogInformation(
                "完成 {RequestName} - 耗時 {ElapsedMilliseconds}ms",
                typeof(TRequest).Name,
                stopwatch.ElapsedMilliseconds);

            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "處理 {RequestName} 時發生錯誤", typeof(TRequest).Name);
            throw;
        }
    }
}

// 交易 Behavior
public class TransactionBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IUnitOfWork _unitOfWork;

    public async Task<TResponse> Handle(
        TRequest request,
        CancellationToken cancellationToken,
        RequestHandlerDelegate<TResponse> next)
    {
        using var transaction = await _unitOfWork.BeginTransactionAsync();

        try
        {
            var response = await next();

            await _unitOfWork.SaveChangesAsync(cancellationToken);
            await transaction.CommitAsync(cancellationToken);

            return response;
        }
        catch
        {
            await transaction.RollbackAsync(cancellationToken);
            throw;
        }
    }
}
```

## Vertical Slice Architecture

### 功能導向組織

```
MyApp/
├── Features/
│   ├── Products/
│   │   ├── Create/
│   │   │   ├── CreateProductCommand.cs
│   │   │   ├── CreateProductHandler.cs
│   │   │   ├── CreateProductValidator.cs
│   │   │   └── CreateProductEndpoint.cs
│   │   ├── GetById/
│   │   │   ├── GetProductQuery.cs
│   │   │   ├── GetProductHandler.cs
│   │   │   └── GetProductEndpoint.cs
│   │   └── Update/
│   │       ├── UpdateProductCommand.cs
│   │       ├── UpdateProductHandler.cs
│   │       └── UpdateProductEndpoint.cs
│   └── Orders/
│       ├── Create/
│       ├── GetById/
│       └── Cancel/
└── Shared/
    ├── Database/
    ├── Messaging/
    └── Utilities/
```

### 實作範例

```csharp
// Features/Products/Create/CreateProductCommand.cs
public record CreateProductCommand(string Name, decimal Price, int Stock) : IRequest<ProductDto>;

// Features/Products/Create/CreateProductHandler.cs
public class CreateProductHandler : IRequestHandler<CreateProductCommand, ProductDto>
{
    private readonly ApplicationDbContext _context;
    private readonly IMapper _mapper;

    public async Task<ProductDto> Handle(
        CreateProductCommand command,
        CancellationToken cancellationToken)
    {
        var product = new Product
        {
            Name = command.Name,
            Price = command.Price,
            Stock = command.Stock
        };

        _context.Products.Add(product);
        await _context.SaveChangesAsync(cancellationToken);

        return _mapper.Map<ProductDto>(product);
    }
}

// Features/Products/Create/CreateProductValidator.cs
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.Price)
            .GreaterThan(0);

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0);
    }
}

// Features/Products/Create/CreateProductEndpoint.cs
public class CreateProductEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("api/products", async (CreateProductCommand command, IMediator mediator) =>
        {
            var product = await mediator.Send(command);
            return Results.Created($"/api/products/{product.Id}", product);
        })
        .WithName("CreateProduct")
        .WithTags("Products")
        .Produces<ProductDto>(StatusCodes.Status201Created)
        .ProducesValidationProblem();
    }
}
```

## 小結

架構設計模式選擇：
- Clean Architecture：適合大型企業專案，清晰分層
- DDD：適合複雜業務邏輯，強調領域模型
- CQRS：適合讀寫分離需求，提升效能
- Vertical Slice：適合中小型專案，減少跨層依賴

相較於 Java 生態：
- Clean Architecture 類似 Hexagonal Architecture
- CQRS 實作比 Axon Framework 更輕量
- MediatR 類似 Spring 的 @EventListener
- Vertical Slice 更貼近微服務的垂直切分

下週將分享 .NET Core 學習旅程的總結和心得。
