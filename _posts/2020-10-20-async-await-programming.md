---
layout: post
title: "深入理解 async/await 非同步程式設計"
date: 2020-10-20 15:30:00 +0800
categories: [框架, .NET]
tags: [.NET Core, async, await, Task, TPL]
---

這週深入研究 .NET Core 中的 async/await 非同步程式設計模式，理解如何正確使用非同步來提升應用程式效能。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## 為什麼需要非同步

### 同步 vs 非同步

```csharp
// 同步版本 - 阻塞執行緒
public class SyncService
{
    public List<Product> GetProducts()
    {
        var products = _httpClient.GetStringAsync("/api/products").Result;
        return JsonSerializer.Deserialize<List<Product>>(products);
    }

    public void ProcessOrders()
    {
        foreach (var order in _orders)
        {
            var result = ProcessOrder(order).Result;
            _logger.LogInformation($"已處理訂單 {order.Id}");
        }
    }
}

// 非同步版本 - 不阻塞執行緒
public class AsyncService
{
    public async Task<List<Product>> GetProductsAsync()
    {
        var products = await _httpClient.GetStringAsync("/api/products");
        return JsonSerializer.Deserialize<List<Product>>(products);
    }

    public async Task ProcessOrdersAsync()
    {
        foreach (var order in _orders)
        {
            var result = await ProcessOrderAsync(order);
            _logger.LogInformation($"已處理訂單 {order.Id}");
        }
    }
}
```

### 效能比較

```csharp
public class PerformanceComparison
{
    [Benchmark]
    public void SyncVersion()
    {
        for (int i = 0; i < 100; i++)
        {
            var result = _httpClient.GetStringAsync("/api/data").Result;
        }
    }

    [Benchmark]
    public async Task AsyncVersion()
    {
        var tasks = new List<Task<string>>();
        
        for (int i = 0; i < 100; i++)
        {
            tasks.Add(_httpClient.GetStringAsync("/api/data"));
        }

        await Task.WhenAll(tasks);
    }
}
```

## Task 基礎

### 建立和執行 Task

```csharp
public class TaskBasics
{
    // 使用 Task.Run 執行 CPU 密集操作
    public async Task<int> CalculateAsync(int number)
    {
        return await Task.Run(() =>
        {
            int result = 0;
            for (int i = 0; i < number; i++)
            {
                result += i;
            }
            return result;
        });
    }

    // 直接回傳 Task
    public Task<string> GetDataAsync()
    {
        return _httpClient.GetStringAsync("/api/data");
    }

    // 回傳已完成的 Task
    public Task<int> GetCachedValueAsync(string key)
    {
        if (_cache.TryGetValue(key, out int value))
        {
            return Task.FromResult(value);
        }

        return FetchFromDatabaseAsync(key);
    }

    // 延遲執行
    public async Task DelayedOperationAsync()
    {
        await Task.Delay(TimeSpan.FromSeconds(5));
        Console.WriteLine("5 秒後執行");
    }
}
```

### Task 組合

```csharp
public class TaskComposition
{
    // 等待所有任務完成
    public async Task<OrderSummary> GetOrderSummaryAsync(int orderId)
    {
        var orderTask = _orderRepository.GetByIdAsync(orderId);
        var itemsTask = _orderItemRepository.GetByOrderIdAsync(orderId);
        var customerTask = _customerRepository.GetByOrderIdAsync(orderId);

        await Task.WhenAll(orderTask, itemsTask, customerTask);

        return new OrderSummary
        {
            Order = await orderTask,
            Items = await itemsTask,
            Customer = await customerTask
        };
    }

    // 等待任一任務完成
    public async Task<string> GetFastestResponseAsync()
    {
        var server1 = _httpClient.GetStringAsync("https://server1.com/api");
        var server2 = _httpClient.GetStringAsync("https://server2.com/api");
        var server3 = _httpClient.GetStringAsync("https://server3.com/api");

        var completedTask = await Task.WhenAny(server1, server2, server3);
        return await completedTask;
    }

    // 批次處理
    public async Task<List<ProcessResult>> ProcessBatchAsync(List<Order> orders)
    {
        const int batchSize = 10;
        var results = new List<ProcessResult>();

        for (int i = 0; i < orders.Count; i += batchSize)
        {
            var batch = orders.Skip(i).Take(batchSize);
            var tasks = batch.Select(order => ProcessOrderAsync(order));
            
            var batchResults = await Task.WhenAll(tasks);
            results.AddRange(batchResults);
        }

        return results;
    }
}
```

## async/await 最佳實踐

### ConfigureAwait

```csharp
public class ConfigureAwaitExample
{
    // ASP.NET Core 不需要 ConfigureAwait(false)
    public async Task<Product> GetProductAsync(int id)
    {
        var product = await _repository.GetByIdAsync(id);
        return product;
    }

    // 但在類別庫中建議使用
    public async Task<string> LibraryMethodAsync()
    {
        var data = await _httpClient.GetStringAsync("/api/data")
            .ConfigureAwait(false);

        var processed = await ProcessDataAsync(data)
            .ConfigureAwait(false);

        return processed;
    }

    // UI 應用程式需要回到原始 Context
    public async Task UpdateUiAsync()
    {
        var data = await FetchDataAsync().ConfigureAwait(true);
        
        // 這裡需要在 UI 執行緒上執行
        _label.Text = data;
    }
}
```

### 避免 async void

```csharp
public class AsyncVoidExample
{
    // 錯誤：無法追蹤例外
    public async void BadMethodAsync()
    {
        await Task.Delay(1000);
        throw new Exception("無法被捕捉");
    }

    // 正確：回傳 Task
    public async Task GoodMethodAsync()
    {
        await Task.Delay(1000);
        throw new Exception("可以被捕捉");
    }

    // 唯一允許使用 async void 的情況：事件處理
    public async void Button_Click(object sender, EventArgs e)
    {
        try
        {
            await ProcessClickAsync();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "處理點擊事件失敗");
        }
    }
}
```

### 避免混用 async 和 sync

```csharp
public class SyncAsyncMixing
{
    // 錯誤：可能造成死鎖
    public Product GetProductBad(int id)
    {
        return GetProductAsync(id).Result;
    }

    // 錯誤：可能造成死鎖
    public Product GetProductBad2(int id)
    {
        return GetProductAsync(id).GetAwaiter().GetResult();
    }

    // 正確：完全非同步
    public async Task<Product> GetProductGood(int id)
    {
        return await GetProductAsync(id);
    }

    // 如果必須同步，提供同步版本
    public Product GetProductSync(int id)
    {
        return _repository.GetById(id);
    }

    public async Task<Product> GetProductAsync(int id)
    {
        return await _repository.GetByIdAsync(id);
    }
}
```

## 取消機制

### 使用 CancellationToken

```csharp
public class CancellationExample
{
    public async Task<List<Product>> SearchProductsAsync(
        string keyword,
        CancellationToken cancellationToken = default)
    {
        // 檢查取消
        cancellationToken.ThrowIfCancellationRequested();

        var products = await _repository.SearchAsync(keyword, cancellationToken);

        // 長時間運算中定期檢查
        var filtered = new List<Product>();
        foreach (var product in products)
        {
            cancellationToken.ThrowIfCancellationRequested();

            if (await IsValidAsync(product, cancellationToken))
            {
                filtered.Add(product);
            }
        }

        return filtered;
    }

    // 設定超時
    public async Task<string> GetDataWithTimeoutAsync()
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

        try
        {
            return await _httpClient.GetStringAsync("/api/data", cts.Token);
        }
        catch (OperationCanceledException)
        {
            _logger.LogWarning("請求超時");
            throw new TimeoutException("API 請求超時");
        }
    }

    // 組合多個 CancellationToken
    public async Task ProcessWithMultipleTokensAsync(
        CancellationToken userToken,
        CancellationToken appToken)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(
            userToken, appToken);

        await LongRunningOperationAsync(cts.Token);
    }
}
```

### Controller 中的取消

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("search")]
    public async Task<ActionResult<List<Product>>> Search(
        [FromQuery] string keyword,
        CancellationToken cancellationToken)
    {
        try
        {
            var products = await _productService.SearchAsync(
                keyword, cancellationToken);

            return Ok(products);
        }
        catch (OperationCanceledException)
        {
            _logger.LogInformation("搜尋已取消: {Keyword}", keyword);
            return StatusCode(499); // Client Closed Request
        }
    }
}
```

## ValueTask

### 何時使用 ValueTask

```csharp
public class ValueTaskExample
{
    private Product _cachedProduct;

    // 使用 ValueTask 避免快取命中時的配置
    public ValueTask<Product> GetProductAsync(int id)
    {
        if (_cachedProduct?.Id == id)
        {
            return new ValueTask<Product>(_cachedProduct);
        }

        return new ValueTask<Product>(FetchProductAsync(id));
    }

    private async Task<Product> FetchProductAsync(int id)
    {
        var product = await _repository.GetByIdAsync(id);
        _cachedProduct = product;
        return product;
    }

    // 高頻率呼叫的方法
    public async ValueTask<bool> IsAuthorizedAsync(int userId, string resource)
    {
        // 檢查快取
        var cacheKey = $"auth:{userId}:{resource}";
        if (_cache.TryGetValue(cacheKey, out bool cached))
        {
            return cached;
        }

        // 查詢資料庫
        var result = await _authRepository.CheckPermissionAsync(userId, resource);
        
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(5));
        return result;
    }
}
```

## 平行處理

### 使用 Task.WhenAll 和 Parallel.ForEach

```csharp
public class ParallelProcessing
{
    // 使用 Task.WhenAll 進行平行處理
    public async Task ProcessItemsAsync(List<Item> items)
    {
        var tasks = items.Select(item => ProcessItemAsync(item));
        await Task.WhenAll(tasks);
    }
    
    // 或使用 Parallel.ForEach 搭配 async
    public async Task ProcessItemsParallelAsync(List<Item> items)
    {
        var tasks = new ConcurrentBag<Task>();
        
        Parallel.ForEach(items, 
            new ParallelOptions { MaxDegreeOfParallelism = 4 },
            item =>
            {
                tasks.Add(ProcessItemAsync(item));
            });
            
        await Task.WhenAll(tasks);
    }

    // .NET Core 3.1 的替代方案
    public async Task ProcessItemsAsyncCore31(List<Item> items)
    {
        var semaphore = new SemaphoreSlim(4); // 限制並發數
        var tasks = items.Select(async item =>
        {
            await semaphore.WaitAsync();
            try
            {
                await ProcessItemAsync(item);
            }
            finally
            {
                semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);
    }

    // 使用 Dataflow
    public async Task ProcessWithDataflowAsync(List<Item> items)
    {
        var options = new ExecutionDataflowBlockOptions
        {
            MaxDegreeOfParallelism = 4
        };

        var block = new ActionBlock<Item>(
            async item => await ProcessItemAsync(item),
            options);

        foreach (var item in items)
        {
            await block.SendAsync(item);
        }

        block.Complete();
        await block.Completion;
    }
}
```

## IAsyncEnumerable

### 非同步串流

```csharp
public class AsyncEnumerableExample
{
    // 產生非同步串流
    public async IAsyncEnumerable<Product> GetProductsStreamAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var skip = 0;
        const int take = 100;

        while (true)
        {
            var products = await _repository.GetPageAsync(skip, take, cancellationToken);

            if (!products.Any())
            {
                break;
            }

            foreach (var product in products)
            {
                cancellationToken.ThrowIfCancellationRequested();
                yield return product;
            }

            skip += take;
        }
    }

    // 消費非同步串流
    public async Task ProcessProductsAsync()
    {
        await foreach (var product in GetProductsStreamAsync())
        {
            await ProcessProductAsync(product);
            _logger.LogInformation($"已處理產品 {product.Id}");
        }
    }

    // 轉換非同步串流
    public async IAsyncEnumerable<ProductDto> TransformProductsAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var product in GetProductsStreamAsync(cancellationToken))
        {
            yield return new ProductDto
            {
                Id = product.Id,
                Name = product.Name,
                Price = product.Price
            };
        }
    }
}
```

### Controller 中使用 IAsyncEnumerable

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("stream")]
    public IAsyncEnumerable<Product> StreamProducts(
        CancellationToken cancellationToken)
    {
        return _productService.GetProductsStreamAsync(cancellationToken);
    }

    [HttpGet("stream-json")]
    public async IAsyncEnumerable<Product> StreamProductsAsJson(
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        await foreach (var product in _productService.GetProductsStreamAsync(cancellationToken))
        {
            yield return product;
            
            // 每 100 筆延遲一下，避免過載
            if (product.Id % 100 == 0)
            {
                await Task.Delay(100, cancellationToken);
            }
        }
    }
}
```

## 常見陷阱

### 例外處理

```csharp
public class ExceptionHandling
{
    // 錯誤：吞掉例外
    public async Task BadExceptionHandlingAsync()
    {
        try
        {
            await Task.WhenAll(
                OperationAsync(),
                AnotherOperationAsync()
            );
        }
        catch (Exception ex)
        {
            // 只會捕捉第一個例外
            _logger.LogError(ex, "發生錯誤");
        }
    }

    // 正確：處理所有例外
    public async Task GoodExceptionHandlingAsync()
    {
        var task1 = OperationAsync();
        var task2 = AnotherOperationAsync();

        try
        {
            await Task.WhenAll(task1, task2);
        }
        catch
        {
            var exceptions = new List<Exception>();

            if (task1.IsFaulted)
            {
                exceptions.AddRange(task1.Exception.InnerExceptions);
            }

            if (task2.IsFaulted)
            {
                exceptions.AddRange(task2.Exception.InnerExceptions);
            }

            foreach (var ex in exceptions)
            {
                _logger.LogError(ex, "發生錯誤");
            }

            throw new AggregateException(exceptions);
        }
    }
}
```

### 記憶體洩漏

```csharp
public class MemoryLeakExample
{
    // 錯誤：忘記釋放資源
    public async Task BadResourceManagementAsync()
    {
        var client = new HttpClient();
        var response = await client.GetStringAsync("/api/data");
        // 忘記釋放 HttpClient
    }

    // 正確：使用 using
    public async Task GoodResourceManagementAsync()
    {
        using var client = new HttpClient();
        var response = await client.GetStringAsync("/api/data");
    }

    // 更好：注入 IHttpClientFactory
    public class Service
    {
        private readonly IHttpClientFactory _httpClientFactory;

        public async Task BestPracticeAsync()
        {
            var client = _httpClientFactory.CreateClient();
            var response = await client.GetStringAsync("/api/data");
        }
    }
}
```

## 效能優化

### 避免不必要的 async

```csharp
public class PerformanceOptimization
{
    // 不好：不必要的 async/await
    public async Task<Product> GetProductBadAsync(int id)
    {
        return await _repository.GetByIdAsync(id);
    }

    // 較好：直接回傳 Task
    public Task<Product> GetProductGoodAsync(int id)
    {
        return _repository.GetByIdAsync(id);
    }

    // 需要例外處理時才用 async
    public async Task<Product> GetProductWithErrorHandlingAsync(int id)
    {
        try
        {
            return await _repository.GetByIdAsync(id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "取得產品失敗: {ProductId}", id);
            throw;
        }
    }
}
```

### 快取 Task

```csharp
public class TaskCaching
{
    private Task<List<Category>> _categoriesTask;
    private readonly SemaphoreSlim _lock = new SemaphoreSlim(1, 1);

    // 快取 Task 避免重複查詢
    public async Task<List<Category>> GetCategoriesAsync()
    {
        if (_categoriesTask != null && !_categoriesTask.IsFaulted)
        {
            return await _categoriesTask;
        }

        await _lock.WaitAsync();
        try
        {
            if (_categoriesTask == null || _categoriesTask.IsFaulted)
            {
                _categoriesTask = _repository.GetAllCategoriesAsync();
            }
        }
        finally
        {
            _lock.Release();
        }

        return await _categoriesTask;
    }

    // 重新整理快取
    public void InvalidateCache()
    {
        _categoriesTask = null;
    }
}
```

## 小結

async/await 最佳實踐：
- 全鏈路非同步，避免混用同步和非同步
- 使用 CancellationToken 支援取消
- 避免 async void，除了事件處理
- 適當使用 ValueTask 優化效能
- 使用 IAsyncEnumerable 處理串流資料

相較於 Java：
- async/await 比 CompletableFuture 更直觀
- ValueTask 類似 Java 的內聯優化
- IAsyncEnumerable 比 Reactor/RxJava 更簡單
- CancellationToken 比 Future.cancel() 更完整

下週將探討 Entity Framework Core 的進階功能。
