---
layout: post
title: "gRPC 在 ASP.NET Core 中的應用"
date: 2020-09-01 15:45:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, gRPC, Protobuf, Microservices]
---

這週研究 gRPC 在 ASP.NET Core 中的實作，了解如何使用 Protocol Buffers 定義服務並建立高效能的 RPC 通訊。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## gRPC 簡介

gRPC 是 Google 開發的高效能 RPC 框架，使用 HTTP/2 和 Protocol Buffers：

優點：
- 使用二進位序列化，比 JSON 更小更快
- HTTP/2 多工傳輸
- 支援串流傳輸
- 強型別服務定義
- 跨語言支援

適用場景：
- 微服務之間的通訊
- 需要高效能的後端服務
- 即時串流應用
- 多語言環境

## 建立 gRPC 服務

### 建立專案

```bash
dotnet new grpc -n GrpcService
cd GrpcService
```

專案結構：

```
GrpcService/
├── Protos/
│   └── greet.proto
├── Services/
│   └── GreeterService.cs
├── appsettings.json
├── Program.cs
└── Startup.cs
```

### 定義 Proto 檔案

`Protos/product.proto`：

```protobuf
syntax = "proto3";

option csharp_namespace = "GrpcService";

package product;

// 產品服務
service ProductService {
  // 取得產品
  rpc GetProduct (GetProductRequest) returns (ProductReply);
  
  // 取得產品列表
  rpc ListProducts (ListProductsRequest) returns (stream ProductReply);
  
  // 建立產品
  rpc CreateProduct (CreateProductRequest) returns (ProductReply);
  
  // 更新產品
  rpc UpdateProduct (UpdateProductRequest) returns (ProductReply);
  
  // 刪除產品
  rpc DeleteProduct (DeleteProductRequest) returns (DeleteProductReply);
  
  // 雙向串流 - 批次更新
  rpc BatchUpdate (stream UpdateProductRequest) returns (stream ProductReply);
}

// 取得產品請求
message GetProductRequest {
  int32 id = 1;
}

// 產品回應
message ProductReply {
  int32 id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
  string category = 5;
}

// 列表請求
message ListProductsRequest {
  int32 page = 1;
  int32 page_size = 2;
  string category = 3;
}

// 建立產品請求
message CreateProductRequest {
  string name = 1;
  double price = 2;
  int32 stock = 3;
  string category = 4;
}

// 更新產品請求
message UpdateProductRequest {
  int32 id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
}

// 刪除產品請求
message DeleteProductRequest {
  int32 id = 1;
}

// 刪除產品回應
message DeleteProductReply {
  bool success = 1;
  string message = 2;
}
```

### 設定 .csproj

```xml
<ItemGroup>
  <Protobuf Include="Protos\product.proto" GrpcServices="Server" />
</ItemGroup>

<ItemGroup>
  <PackageReference Include="Grpc.AspNetCore" Version="2.40.0" />
</ItemGroup>
```

### 實作 gRPC 服務

```csharp
using Grpc.Core;
using Microsoft.Extensions.Logging;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace GrpcService.Services
{
    public class ProductService : product.ProductService.ProductServiceBase
    {
        private readonly ILogger<ProductService> _logger;
        private static readonly List<ProductReply> _products = new()
        {
            new ProductReply { Id = 1, Name = "iPhone 13", Price = 35900, Stock = 100, Category = "Electronics" },
            new ProductReply { Id = 2, Name = "MacBook Pro", Price = 75900, Stock = 50, Category = "Electronics" },
            new ProductReply { Id = 3, Name = "Clean Code", Price = 580, Stock = 200, Category = "Books" }
        };

        public ProductService(ILogger<ProductService> logger)
        {
            _logger = logger;
        }

        // Unary RPC - 單一請求回應
        public override Task<ProductReply> GetProduct(
            GetProductRequest request,
            ServerCallContext context)
        {
            _logger.LogInformation("取得產品: {Id}", request.Id);

            var product = _products.FirstOrDefault(p => p.Id == request.Id);

            if (product == null)
            {
                throw new RpcException(new Status(
                    StatusCode.NotFound,
                    $"找不到產品 {request.Id}"));
            }

            return Task.FromResult(product);
        }

        // Server Streaming RPC - 伺服器串流
        public override async Task ListProducts(
            ListProductsRequest request,
            IServerStreamWriter<ProductReply> responseStream,
            ServerCallContext context)
        {
            _logger.LogInformation(
                "列出產品: Category={Category}, Page={Page}, PageSize={PageSize}",
                request.Category, request.Page, request.PageSize);

            var query = _products.AsQueryable();

            if (!string.IsNullOrEmpty(request.Category))
            {
                query = query.Where(p => p.Category == request.Category);
            }

            var products = query
                .Skip((request.Page - 1) * request.PageSize)
                .Take(request.PageSize);

            foreach (var product in products)
            {
                // 檢查是否取消
                if (context.CancellationToken.IsCancellationRequested)
                {
                    break;
                }

                // 傳送產品至串流
                await responseStream.WriteAsync(product);

                // 模擬處理延遲
                await Task.Delay(100);
            }
        }

        // Unary RPC - 建立產品
        public override Task<ProductReply> CreateProduct(
            CreateProductRequest request,
            ServerCallContext context)
        {
            _logger.LogInformation("建立產品: {Name}", request.Name);

            var product = new ProductReply
            {
                Id = _products.Max(p => p.Id) + 1,
                Name = request.Name,
                Price = request.Price,
                Stock = request.Stock,
                Category = request.Category
            };

            _products.Add(product);

            return Task.FromResult(product);
        }

        // Unary RPC - 更新產品
        public override Task<ProductReply> UpdateProduct(
            UpdateProductRequest request,
            ServerCallContext context)
        {
            _logger.LogInformation("更新產品: {Id}", request.Id);

            var product = _products.FirstOrDefault(p => p.Id == request.Id);

            if (product == null)
            {
                throw new RpcException(new Status(
                    StatusCode.NotFound,
                    $"找不到產品 {request.Id}"));
            }

            product.Name = request.Name;
            product.Price = request.Price;
            product.Stock = request.Stock;

            return Task.FromResult(product);
        }

        // Unary RPC - 刪除產品
        public override Task<DeleteProductReply> DeleteProduct(
            DeleteProductRequest request,
            ServerCallContext context)
        {
            _logger.LogInformation("刪除產品: {Id}", request.Id);

            var product = _products.FirstOrDefault(p => p.Id == request.Id);

            if (product == null)
            {
                return Task.FromResult(new DeleteProductReply
                {
                    Success = false,
                    Message = $"找不到產品 {request.Id}"
                });
            }

            _products.Remove(product);

            return Task.FromResult(new DeleteProductReply
            {
                Success = true,
                Message = "產品已刪除"
            });
        }

        // Bidirectional Streaming RPC - 雙向串流
        public override async Task BatchUpdate(
            IAsyncStreamReader<UpdateProductRequest> requestStream,
            IServerStreamWriter<ProductReply> responseStream,
            ServerCallContext context)
        {
            _logger.LogInformation("開始批次更新");

            await foreach (var request in requestStream.ReadAllAsync())
            {
                _logger.LogInformation("批次更新產品: {Id}", request.Id);

                var product = _products.FirstOrDefault(p => p.Id == request.Id);

                if (product != null)
                {
                    product.Name = request.Name;
                    product.Price = request.Price;
                    product.Stock = request.Stock;

                    await responseStream.WriteAsync(product);
                }
            }

            _logger.LogInformation("批次更新完成");
        }
    }
}
```

### 註冊服務

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddGrpc();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseRouting();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapGrpcService<ProductService>();

        endpoints.MapGet("/", async context =>
        {
            await context.Response.WriteAsync(
                "Communication with gRPC endpoints must be made through a gRPC client.");
        });
    });
}
```

## 建立 gRPC 客戶端

### 建立 Console 專案

```bash
dotnet new console -n GrpcClient
cd GrpcClient
```

### 設定 .csproj

```xml
<ItemGroup>
  <PackageReference Include="Google.Protobuf" Version="3.19.4" />
  <PackageReference Include="Grpc.Net.Client" Version="2.40.0" />
  <PackageReference Include="Grpc.Tools" Version="2.43.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>

<ItemGroup>
  <Protobuf Include="Protos\product.proto" GrpcServices="Client" />
</ItemGroup>
```

### 使用客戶端

```csharp
using Grpc.Net.Client;
using GrpcService;
using System;
using System.Threading.Tasks;

namespace GrpcClient
{
    class Program
    {
        static async Task Main(string[] args)
        {
            using var channel = GrpcChannel.ForAddress("https://localhost:5001");
            var client = new ProductService.ProductServiceClient(channel);

            // 取得產品
            Console.WriteLine("=== 取得產品 ===");
            var product = await client.GetProductAsync(new GetProductRequest { Id = 1 });
            Console.WriteLine($"產品: {product.Name}, 價格: {product.Price}");

            // 列表產品（Server Streaming）
            Console.WriteLine("\n=== 列表產品 ===");
            var listCall = client.ListProducts(new ListProductsRequest
            {
                Page = 1,
                PageSize = 10,
                Category = "Electronics"
            });

            await foreach (var p in listCall.ResponseStream.ReadAllAsync())
            {
                Console.WriteLine($"- {p.Name}: ${p.Price}");
            }

            // 建立產品
            Console.WriteLine("\n=== 建立產品 ===");
            var newProduct = await client.CreateProductAsync(new CreateProductRequest
            {
                Name = "iPad Pro",
                Price = 25900,
                Stock = 80,
                Category = "Electronics"
            });
            Console.WriteLine($"建立成功: {newProduct.Id} - {newProduct.Name}");

            // 更新產品
            Console.WriteLine("\n=== 更新產品 ===");
            var updatedProduct = await client.UpdateProductAsync(new UpdateProductRequest
            {
                Id = newProduct.Id,
                Name = "iPad Pro 2020",
                Price = 26900,
                Stock = 75
            });
            Console.WriteLine($"更新成功: {updatedProduct.Name}, 價格: {updatedProduct.Price}");

            // 刪除產品
            Console.WriteLine("\n=== 刪除產品 ===");
            var deleteResult = await client.DeleteProductAsync(new DeleteProductRequest
            {
                Id = newProduct.Id
            });
            Console.WriteLine($"刪除結果: {deleteResult.Success}, {deleteResult.Message}");

            // 批次更新（Bidirectional Streaming）
            Console.WriteLine("\n=== 批次更新 ===");
            using var batchCall = client.BatchUpdate();

            // 傳送更新請求
            var updateTask = Task.Run(async () =>
            {
                for (int i = 1; i <= 3; i++)
                {
                    await batchCall.RequestStream.WriteAsync(new UpdateProductRequest
                    {
                        Id = i,
                        Name = $"Updated Product {i}",
                        Price = 1000 + i * 100,
                        Stock = 50 + i * 10
                    });
                }

                await batchCall.RequestStream.CompleteAsync();
            });

            // 接收更新結果
            var receiveTask = Task.Run(async () =>
            {
                await foreach (var response in batchCall.ResponseStream.ReadAllAsync())
                {
                    Console.WriteLine($"更新完成: {response.Name}, 價格: {response.Price}");
                }
            });

            await Task.WhenAll(updateTask, receiveTask);

            Console.WriteLine("\nPress any key to exit...");
            Console.ReadKey();
        }
    }
}
```

## 進階功能

### Deadline 和 Timeout

```csharp
// 伺服器端
public override async Task<ProductReply> GetProduct(
    GetProductRequest request,
    ServerCallContext context)
{
    // 檢查 deadline
    if (context.Deadline < DateTime.UtcNow.AddSeconds(1))
    {
        throw new RpcException(new Status(
            StatusCode.DeadlineExceeded,
            "處理時間不足"));
    }

    // 模擬處理時間
    await Task.Delay(500);

    var product = _products.FirstOrDefault(p => p.Id == request.Id);
    return product;
}

// 客戶端
var deadline = DateTime.UtcNow.AddSeconds(5);
var product = await client.GetProductAsync(
    new GetProductRequest { Id = 1 },
    deadline: deadline);
```

### Metadata (Headers)

```csharp
// 客戶端發送 metadata
var metadata = new Metadata
{
    { "Authorization", "Bearer token123" },
    { "X-Custom-Header", "custom-value" }
};

var product = await client.GetProductAsync(
    new GetProductRequest { Id = 1 },
    headers: metadata);

// 伺服器端讀取 metadata
public override Task<ProductReply> GetProduct(
    GetProductRequest request,
    ServerCallContext context)
{
    var authHeader = context.RequestHeaders.GetValue("Authorization");
    var customHeader = context.RequestHeaders.GetValue("X-Custom-Header");

    _logger.LogInformation("Auth: {Auth}, Custom: {Custom}", 
        authHeader, customHeader);

    // 設定回應 headers
    context.ResponseTrailers.Add("X-Response-Id", Guid.NewGuid().ToString());

    var product = _products.FirstOrDefault(p => p.Id == request.Id);
    return Task.FromResult(product);
}
```

### 攔截器 (Interceptor)

```csharp
// 伺服器端攔截器
public class ServerLoggingInterceptor : Interceptor
{
    private readonly ILogger<ServerLoggingInterceptor> _logger;

    public ServerLoggingInterceptor(ILogger<ServerLoggingInterceptor> logger)
    {
        _logger = logger;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        _logger.LogInformation("收到請求: {Method}", context.Method);

        try
        {
            var response = await continuation(request, context);
            _logger.LogInformation("請求成功: {Method}", context.Method);
            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "請求失敗: {Method}", context.Method);
            throw;
        }
    }
}

// 註冊攔截器
services.AddGrpc(options =>
{
    options.Interceptors.Add<ServerLoggingInterceptor>();
});

// 客戶端攔截器
public class ClientLoggingInterceptor : Interceptor
{
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        Console.WriteLine($"發送請求: {context.Method.FullName}");

        var metadata = context.Options.Headers ?? new Metadata();
        metadata.Add("X-Request-Id", Guid.NewGuid().ToString());

        var options = context.Options.WithHeaders(metadata);
        context = new ClientInterceptorContext<TRequest, TResponse>(
            context.Method,
            context.Host,
            options);

        return continuation(request, context);
    }
}

// 使用客戶端攔截器
var channel = GrpcChannel.ForAddress("https://localhost:5001");
var callInvoker = channel.Intercept(new ClientLoggingInterceptor());
var client = new ProductService.ProductServiceClient(callInvoker);
```

### 錯誤處理

```csharp
// 伺服器端
public override Task<ProductReply> GetProduct(
    GetProductRequest request,
    ServerCallContext context)
{
    if (request.Id <= 0)
    {
        throw new RpcException(new Status(
            StatusCode.InvalidArgument,
            "產品 ID 必須大於 0"));
    }

    var product = _products.FirstOrDefault(p => p.Id == request.Id);

    if (product == null)
    {
        var metadata = new Metadata
        {
            { "product-id", request.Id.ToString() }
        };

        throw new RpcException(
            new Status(StatusCode.NotFound, $"找不到產品 {request.Id}"),
            metadata);
    }

    return Task.FromResult(product);
}

// 客戶端
try
{
    var product = await client.GetProductAsync(new GetProductRequest { Id = 999 });
}
catch (RpcException ex)
{
    Console.WriteLine($"錯誤碼: {ex.StatusCode}");
    Console.WriteLine($"訊息: {ex.Status.Detail}");

    if (ex.Trailers.Count > 0)
    {
        var productId = ex.Trailers.GetValue("product-id");
        Console.WriteLine($"產品 ID: {productId}");
    }
}
```

### 整合 EF Core

```csharp
public class ProductService : product.ProductService.ProductServiceBase
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        ApplicationDbContext context,
        ILogger<ProductService> logger)
    {
        _context = context;
        _logger = logger;
    }

    public override async Task<ProductReply> GetProduct(
        GetProductRequest request,
        ServerCallContext context)
    {
        var product = await _context.Products
            .Where(p => p.Id == request.Id)
            .Select(p => new ProductReply
            {
                Id = p.Id,
                Name = p.Name,
                Price = (double)p.Price,
                Stock = p.Stock,
                Category = p.Category.Name
            })
            .FirstOrDefaultAsync();

        if (product == null)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound,
                $"找不到產品 {request.Id}"));
        }

        return product;
    }

    public override async Task ListProducts(
        ListProductsRequest request,
        IServerStreamWriter<ProductReply> responseStream,
        ServerCallContext context)
    {
        var query = _context.Products.AsQueryable();

        if (!string.IsNullOrEmpty(request.Category))
        {
            query = query.Where(p => p.Category.Name == request.Category);
        }

        var products = await query
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(p => new ProductReply
            {
                Id = p.Id,
                Name = p.Name,
                Price = (double)p.Price,
                Stock = p.Stock,
                Category = p.Category.Name
            })
            .ToListAsync();

        foreach (var product in products)
        {
            if (context.CancellationToken.IsCancellationRequested)
            {
                break;
            }

            await responseStream.WriteAsync(product);
        }
    }
}
```

## 效能優化

### 連線重用

```csharp
// 單例客戶端
public class ProductServiceClientFactory
{
    private readonly GrpcChannel _channel;
    private readonly ProductService.ProductServiceClient _client;

    public ProductServiceClientFactory(string address)
    {
        _channel = GrpcChannel.ForAddress(address);
        _client = new ProductService.ProductServiceClient(_channel);
    }

    public ProductService.ProductServiceClient GetClient() => _client;

    public async ValueTask DisposeAsync()
    {
        await _channel.ShutdownAsync();
        _channel.Dispose();
    }
}

// 註冊為單例
services.AddSingleton(new ProductServiceClientFactory("https://localhost:5001"));
```

### 壓縮

```csharp
// 伺服器端
services.AddGrpc(options =>
{
    options.ResponseCompressionLevel = System.IO.Compression.CompressionLevel.Optimal;
});

// 客戶端
var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
{
    CompressionProviders = new[]
    {
        new GzipCompressionProvider(System.IO.Compression.CompressionLevel.Optimal)
    }
});
```

## 小結

ASP.NET Core 中的 gRPC：
- 使用 Protocol Buffers 定義服務契約
- 支援四種 RPC 模式（Unary、Server Streaming、Client Streaming、Bidirectional Streaming）
- 內建攔截器和錯誤處理
- 完整的 HTTP/2 支援

相較於 REST API：
- 效能更好（二進位序列化）
- 強型別服務定義
- 支援雙向串流
- 更適合微服務通訊

下週將探討 SignalR 即時通訊框架。
