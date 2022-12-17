---
layout: post
title: ".NET Core 微服務架構設計"
date: 2020-09-29 14:30:00 +0800
categories: [架構, .NET]
tags: [.NET Core, Microservices, Architecture, API Gateway]
---

這週研究 .NET Core 微服務架構的設計模式與實踐，了解如何構建可擴展的分散式系統。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## 微服務架構概述

### 單體架構 vs 微服務架構

單體架構特點：
- 所有功能在單一應用程式中
- 部署簡單
- 開發初期快速
- 但難以擴展和維護

微服務架構特點：
- 每個服務獨立部署
- 技術棧可以不同
- 獨立擴展
- 故障隔離
- 複雜度較高

### 微服務設計原則

- 單一職責：每個服務只負責一個業務功能
- 自主性：服務可獨立開發、部署、擴展
- 去中心化：沒有單點故障
- 彈性：能夠處理失敗
- 可觀測性：日誌、監控、追蹤

## 服務拆分策略

### 按業務領域拆分

```
訂單服務 (Order Service)
- 建立訂單
- 查詢訂單
- 取消訂單

產品服務 (Product Service)
- 產品管理
- 庫存管理
- 產品搜尋

使用者服務 (User Service)
- 使用者註冊
- 使用者認證
- 個人資料管理

支付服務 (Payment Service)
- 處理支付
- 退款
- 支付記錄
```

### 專案結構

```
MyMicroservices/
├── src/
│   ├── Services/
│   │   ├── Order/
│   │   │   ├── Order.API/
│   │   │   ├── Order.Domain/
│   │   │   └── Order.Infrastructure/
│   │   ├── Product/
│   │   │   ├── Product.API/
│   │   │   ├── Product.Domain/
│   │   │   └── Product.Infrastructure/
│   │   └── User/
│   │       ├── User.API/
│   │       ├── User.Domain/
│   │       └── User.Infrastructure/
│   ├── ApiGateway/
│   │   └── Gateway.API/
│   └── Shared/
│       ├── Common/
│       ├── EventBus/
│       └── Logging/
└── docker-compose.yml
```

## API Gateway

### 使用 Ocelot

```bash
dotnet new webapi -n Gateway.API
cd Gateway.API
dotnet add package Ocelot
```

### Ocelot 設定

```json
// ocelot.json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/products/{everything}",
      "UpstreamHttpMethod": [ "Get", "Post", "Put", "Delete" ],
      "RateLimitOptions": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1s",
        "PeriodTimespan": 1,
        "Limit": 10
      }
    },
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "order-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/orders/{everything}",
      "UpstreamHttpMethod": [ "Get", "Post", "Put", "Delete" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer"
      }
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "http://localhost:5000",
    "RateLimitOptions": {
      "DisableRateLimitHeaders": false,
      "QuotaExceededMessage": "請求過於頻繁",
      "HttpStatusCode": 429
    }
  }
}
```

### 設定 API Gateway

```csharp
// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, config) =>
            {
                config.AddJsonFile("ocelot.json", optional: false, reloadOnChange: true);
            })
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}

// Startup.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddOcelot();
        
        // JWT 認證
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer("Bearer", options =>
            {
                options.Authority = "http://identity-service";
                options.RequireHttpsMetadata = false;
                options.Audience = "api";
            });
    }

    public async void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        await app.UseOcelot();
    }
}
```

## 服務間通訊

### HTTP REST 通訊

```csharp
public class ProductService
{
    private readonly HttpClient _httpClient;

    public ProductService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<Product> GetProductAsync(int productId)
    {
        var response = await _httpClient.GetAsync($"http://product-service/api/products/{productId}");
        response.EnsureSuccessStatusCode();

        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<Product>(content);
    }
}

// Startup.cs
services.AddHttpClient<ProductService>(client =>
{
    client.BaseAddress = new Uri("http://product-service");
    client.Timeout = TimeSpan.FromSeconds(30);
});

// 使用 Polly 重試策略
services.AddHttpClient<ProductService>()
    .AddTransientHttpErrorPolicy(builder =>
        builder.WaitAndRetryAsync(new[]
        {
            TimeSpan.FromSeconds(1),
            TimeSpan.FromSeconds(2),
            TimeSpan.FromSeconds(3)
        }))
    .AddTransientHttpErrorPolicy(builder =>
        builder.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

### 使用 gRPC 通訊

```protobuf
// product.proto
syntax = "proto3";

service ProductService {
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc CheckStock (CheckStockRequest) returns (CheckStockResponse);
}

message GetProductRequest {
  int32 product_id = 1;
}

message ProductResponse {
  int32 id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
}

message CheckStockRequest {
  int32 product_id = 1;
  int32 quantity = 2;
}

message CheckStockResponse {
  bool available = 1;
}
```

```csharp
// 客戶端使用
public class OrderService
{
    private readonly ProductService.ProductServiceClient _productClient;

    public OrderService(ProductService.ProductServiceClient productClient)
    {
        _productClient = productClient;
    }

    public async Task<bool> CreateOrderAsync(CreateOrderRequest request)
    {
        // 檢查庫存
        var stockResponse = await _productClient.CheckStockAsync(
            new CheckStockRequest
            {
                ProductId = request.ProductId,
                Quantity = request.Quantity
            });

        if (!stockResponse.Available)
        {
            throw new InvalidOperationException("庫存不足");
        }

        // 建立訂單
        // ...
        
        return true;
    }
}
```

## 事件驅動架構

### 使用 RabbitMQ

```bash
dotnet add package RabbitMQ.Client
```

```csharp
// 事件定義
public class OrderCreatedEvent
{
    public int OrderId { get; set; }
    public int UserId { get; set; }
    public decimal TotalAmount { get; set; }
    public List<OrderItem> Items { get; set; }
    public DateTime CreatedAt { get; set; }
}

// 事件發布者
public class EventBus
{
    private readonly IConnection _connection;
    private readonly IModel _channel;

    public EventBus(string hostname)
    {
        var factory = new ConnectionFactory { HostName = hostname };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }

    public void Publish<T>(string exchangeName, string routingKey, T message)
    {
        _channel.ExchangeDeclare(exchangeName, ExchangeType.Topic, durable: true);

        var json = JsonSerializer.Serialize(message);
        var body = Encoding.UTF8.GetBytes(json);

        var properties = _channel.CreateBasicProperties();
        properties.Persistent = true;
        properties.ContentType = "application/json";

        _channel.BasicPublish(
            exchange: exchangeName,
            routingKey: routingKey,
            basicProperties: properties,
            body: body);
    }
}

// 訂單服務發布事件
public class OrderService
{
    private readonly EventBus _eventBus;

    public async Task CreateOrderAsync(CreateOrderRequest request)
    {
        // 建立訂單
        var order = new Order
        {
            UserId = request.UserId,
            TotalAmount = request.TotalAmount,
            CreatedAt = DateTime.UtcNow
        };

        await _repository.CreateAsync(order);

        // 發布事件
        _eventBus.Publish("order.events", "order.created", new OrderCreatedEvent
        {
            OrderId = order.Id,
            UserId = order.UserId,
            TotalAmount = order.TotalAmount,
            CreatedAt = order.CreatedAt
        });
    }
}

// 事件訂閱者
public class OrderEventHandler : BackgroundService
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    private readonly ILogger<OrderEventHandler> _logger;

    public OrderEventHandler(ILogger<OrderEventHandler> logger)
    {
        _logger = logger;
        
        var factory = new ConnectionFactory { HostName = "rabbitmq" };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();

        _channel.ExchangeDeclare("order.events", ExchangeType.Topic, durable: true);
        _channel.QueueDeclare("inventory.order.created", durable: true, exclusive: false, autoDelete: false);
        _channel.QueueBind("inventory.order.created", "order.events", "order.created");
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var consumer = new EventingBasicConsumer(_channel);
        
        consumer.Received += async (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var message = Encoding.UTF8.GetString(body);
            
            try
            {
                var orderEvent = JsonSerializer.Deserialize<OrderCreatedEvent>(message);
                await HandleOrderCreatedAsync(orderEvent);
                
                _channel.BasicAck(ea.DeliveryTag, false);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "處理訂單事件失敗");
                _channel.BasicNack(ea.DeliveryTag, false, true);
            }
        };

        _channel.BasicConsume("inventory.order.created", autoAck: false, consumer);

        return Task.CompletedTask;
    }

    private async Task HandleOrderCreatedAsync(OrderCreatedEvent orderEvent)
    {
        _logger.LogInformation("處理訂單建立事件: {OrderId}", orderEvent.OrderId);
        
        // 更新庫存
        foreach (var item in orderEvent.Items)
        {
            await UpdateInventoryAsync(item.ProductId, -item.Quantity);
        }
    }
}
```

## Saga 模式

### 編排式 Saga (Orchestration)

```csharp
public class OrderSaga
{
    private readonly IOrderService _orderService;
    private readonly IPaymentService _paymentService;
    private readonly IInventoryService _inventoryService;
    private readonly IShippingService _shippingService;

    public async Task<SagaResult> ExecuteAsync(CreateOrderRequest request)
    {
        var sagaId = Guid.NewGuid();
        var context = new SagaContext { SagaId = sagaId };

        try
        {
            // 步驟 1: 建立訂單
            var order = await _orderService.CreateOrderAsync(request);
            context.OrderId = order.Id;

            // 步驟 2: 預留庫存
            await _inventoryService.ReserveStockAsync(order.Id, request.Items);

            // 步驟 3: 處理支付
            var payment = await _paymentService.ProcessPaymentAsync(order.Id, order.TotalAmount);
            context.PaymentId = payment.Id;

            // 步驟 4: 建立出貨單
            await _shippingService.CreateShipmentAsync(order.Id, request.ShippingAddress);

            return SagaResult.Success(context);
        }
        catch (Exception ex)
        {
            // 補償交易
            await CompensateAsync(context, ex);
            return SagaResult.Failure(ex.Message);
        }
    }

    private async Task CompensateAsync(SagaContext context, Exception exception)
    {
        try
        {
            // 逆向執行補償
            if (context.PaymentId.HasValue)
            {
                await _paymentService.RefundAsync(context.PaymentId.Value);
            }

            if (context.OrderId.HasValue)
            {
                await _inventoryService.ReleaseStockAsync(context.OrderId.Value);
                await _orderService.CancelOrderAsync(context.OrderId.Value);
            }
        }
        catch (Exception compensationEx)
        {
            // 記錄補償失敗，需要人工介入
            _logger.LogError(compensationEx, "Saga 補償失敗，SagaId: {SagaId}", context.SagaId);
        }
    }
}
```

## 服務發現

### 使用 Consul

```bash
dotnet add package Consul
```

```csharp
// 服務註冊
public class ConsulServiceRegistry
{
    private readonly IConsulClient _consulClient;

    public async Task RegisterServiceAsync(string serviceName, string serviceId, string host, int port)
    {
        var registration = new AgentServiceRegistration
        {
            ID = serviceId,
            Name = serviceName,
            Address = host,
            Port = port,
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{host}:{port}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5),
                DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(1)
            }
        };

        await _consulClient.Agent.ServiceRegister(registration);
    }

    public async Task DeregisterServiceAsync(string serviceId)
    {
        await _consulClient.Agent.ServiceDeregister(serviceId);
    }
}

// Startup.cs
public void Configure(IApplicationBuilder app, IHostApplicationLifetime lifetime)
{
    var serviceId = $"{Configuration["ServiceName"]}-{Guid.NewGuid()}";
    var serviceName = Configuration["ServiceName"];
    var host = Configuration["ServiceHost"];
    var port = int.Parse(Configuration["ServicePort"]);

    var registry = app.ApplicationServices.GetRequiredService<ConsulServiceRegistry>();

    lifetime.ApplicationStarted.Register(async () =>
    {
        await registry.RegisterServiceAsync(serviceName, serviceId, host, port);
    });

    lifetime.ApplicationStopping.Register(async () =>
    {
        await registry.DeregisterServiceAsync(serviceId);
    });
}

// 服務發現
public class ProductServiceClient
{
    private readonly IConsulClient _consulClient;
    private readonly HttpClient _httpClient;

    public async Task<Product> GetProductAsync(int productId)
    {
        // 從 Consul 取得服務位址
        var services = await _consulClient.Health.Service("product-service", "", true);
        var service = services.Response.FirstOrDefault();

        if (service == null)
        {
            throw new Exception("找不到產品服務");
        }

        var url = $"http://{service.Service.Address}:{service.Service.Port}/api/products/{productId}";
        var response = await _httpClient.GetAsync(url);
        
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Product>();
    }
}
```

## 分散式追蹤

### 使用 Jaeger

```bash
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Exporter.Jaeger
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddOpenTelemetryTracing(builder =>
    {
        builder
            .SetResourceBuilder(ResourceBuilder.CreateDefault()
                .AddService(Configuration["ServiceName"]))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSqlClientInstrumentation()
            .AddJaegerExporter(options =>
            {
                options.AgentHost = Configuration["Jaeger:Host"];
                options.AgentPort = int.Parse(Configuration["Jaeger:Port"]);
            });
    });
}
```

## Docker Compose 整合

```yaml
# docker-compose.yml
version: '3.8'

services:
  gateway:
    build: ./Gateway.API
    ports:
      - "5000:80"
    depends_on:
      - order-service
      - product-service
      - user-service

  order-service:
    build: ./Services/Order/Order.API
    environment:
      - ConnectionStrings__DefaultConnection=Server=order-db;Database=OrderDb;User=sa;Password=YourPassword123!
      - RabbitMQ__Host=rabbitmq
    depends_on:
      - order-db
      - rabbitmq

  product-service:
    build: ./Services/Product/Product.API
    environment:
      - ConnectionStrings__DefaultConnection=Server=product-db;Database=ProductDb;User=sa;Password=YourPassword123!
    depends_on:
      - product-db

  user-service:
    build: ./Services/User/User.API
    environment:
      - ConnectionStrings__DefaultConnection=Server=user-db;Database=UserDb;User=sa;Password=YourPassword123!
    depends_on:
      - user-db

  order-db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!

  product-db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!

  user-db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "6831:6831/udp"
```

## 小結

.NET Core 微服務架構：
- API Gateway 統一入口
- 事件驅動實現鬆耦合
- Saga 模式處理分散式交易
- 服務發現動態定位服務
- 分散式追蹤監控請求

相較於單體架構：
- 獨立部署和擴展
- 技術棧多樣化
- 故障隔離
- 但複雜度增加
- 需要完善的監控

下週將探討應用程式效能監控與追蹤。
