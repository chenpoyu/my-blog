---
layout: post
title: ".NET Core 應用程式效能監控"
date: 2020-10-06 15:50:00 +0800
categories: [框架, .NET]
tags: [.NET Core, Monitoring, Performance, APM]
---

這週研究 .NET Core 應用程式的效能監控，包含日誌、指標收集和 APM 工具整合。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Application Insights

### 安裝與設定

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        })
        .ConfigureLogging((context, logging) =>
        {
            logging.AddApplicationInsights(
                context.Configuration["ApplicationInsights:InstrumentationKey"]);
        });

// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddApplicationInsightsTelemetry(options =>
    {
        options.InstrumentationKey = Configuration["ApplicationInsights:InstrumentationKey"];
        options.EnableAdaptiveSampling = true;
        options.EnableQuickPulseMetricStream = true;
    });

    services.AddControllers();
}
```

### 自訂遙測

```csharp
public class ProductService
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        TelemetryClient telemetryClient,
        ILogger<ProductService> logger)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
    }

    public async Task<Product> GetByIdAsync(int id)
    {
        using var operation = _telemetryClient.StartOperation<DependencyTelemetry>("GetProduct");
        operation.Telemetry.Type = "Database";
        operation.Telemetry.Data = $"ProductId: {id}";

        try
        {
            var product = await _repository.GetByIdAsync(id);

            _telemetryClient.TrackEvent("ProductRetrieved", new Dictionary<string, string>
            {
                { "ProductId", id.ToString() },
                { "ProductName", product?.Name }
            });

            operation.Telemetry.Success = true;
            return product;
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex, new Dictionary<string, string>
            {
                { "ProductId", id.ToString() }
            });

            operation.Telemetry.Success = false;
            throw;
        }
    }

    public async Task<List<Product>> SearchAsync(string keyword)
    {
        var stopwatch = Stopwatch.StartNew();

        var products = await _repository.SearchAsync(keyword);

        stopwatch.Stop();

        _telemetryClient.TrackMetric("SearchDuration", stopwatch.ElapsedMilliseconds, new Dictionary<string, string>
        {
            { "Keyword", keyword },
            { "ResultCount", products.Count.ToString() }
        });

        return products;
    }
}
```

## Prometheus + Grafana

### 安裝套件

```bash
dotnet add package prometheus-net.AspNetCore
```

### 設定 Prometheus

```csharp
// Startup.cs
using Prometheus;

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // 啟用 HTTP 指標收集
    app.UseHttpMetrics();

    app.UseRouting();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
        
        // 暴露 Prometheus 指標端點
        endpoints.MapMetrics();
    });
}
```

### 自訂指標

```csharp
public class ProductService
{
    private static readonly Counter ProductsRetrieved = Metrics
        .CreateCounter("products_retrieved_total", "取得產品的次數");

    private static readonly Histogram ProductSearchDuration = Metrics
        .CreateHistogram("product_search_duration_seconds", "產品搜尋耗時");

    private static readonly Gauge ActiveSearches = Metrics
        .CreateGauge("product_active_searches", "進行中的搜尋數量");

    public async Task<Product> GetByIdAsync(int id)
    {
        ProductsRetrieved.Inc();
        
        var product = await _repository.GetByIdAsync(id);
        return product;
    }

    public async Task<List<Product>> SearchAsync(string keyword)
    {
        ActiveSearches.Inc();

        try
        {
            using (ProductSearchDuration.NewTimer())
            {
                var products = await _repository.SearchAsync(keyword);
                return products;
            }
        }
        finally
        {
            ActiveSearches.Dec();
        }
    }
}
```

### Docker Compose 整合

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'myapp'
    static_configs:
      - targets: ['web:80']
    metrics_path: '/metrics'
```

## Serilog 結構化日誌

### 安裝套件

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Elasticsearch
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
```

### 設定 Serilog

```csharp
// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
            .MinimumLevel.Override("System", LogEventLevel.Warning)
            .Enrich.FromLogContext()
            .Enrich.WithMachineName()
            .Enrich.WithThreadId()
            .WriteTo.Console(
                outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
            .WriteTo.File(
                path: "logs/log-.txt",
                rollingInterval: RollingInterval.Day,
                outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
            .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
            {
                AutoRegisterTemplate = true,
                IndexFormat = "myapp-logs-{0:yyyy.MM.dd}",
                NumberOfShards = 2,
                NumberOfReplicas = 1
            })
            .CreateLogger();

        try
        {
            Log.Information("應用程式啟動");
            CreateHostBuilder(args).Build().Run();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "應用程式啟動失敗");
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .UseSerilog()
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

### 使用結構化日誌

```csharp
public class ProductService
{
    private readonly ILogger<ProductService> _logger;

    public async Task<Product> GetByIdAsync(int id)
    {
        _logger.LogInformation("取得產品 {ProductId}", id);

        var product = await _repository.GetByIdAsync(id);

        if (product == null)
        {
            _logger.LogWarning("找不到產品 {ProductId}", id);
            return null;
        }

        _logger.LogInformation(
            "成功取得產品 {ProductId}: {ProductName}, 價格: {Price}",
            product.Id, product.Name, product.Price);

        return product;
    }

    public async Task CreateAsync(CreateProductRequest request)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["UserId"] = _currentUser.Id,
            ["Action"] = "CreateProduct"
        }))
        {
            _logger.LogInformation("建立產品: {@ProductRequest}", request);

            try
            {
                var product = await _repository.CreateAsync(request);

                _logger.LogInformation(
                    "產品建立成功 {ProductId}: {ProductName}",
                    product.Id, product.Name);

                return product;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "建立產品失敗: {@ProductRequest}", request);
                throw;
            }
        }
    }
}
```

## ELK Stack 整合

### Docker Compose 設定

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"
    depends_on:
      - elasticsearch
    environment:
      - ElasticConfiguration__Uri=http://elasticsearch:9200

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:7.14.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

volumes:
  elasticsearch-data:
```

## 效能分析

### MiniProfiler

```bash
dotnet add package MiniProfiler.AspNetCore.Mvc
dotnet add package MiniProfiler.EntityFrameworkCore
```

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddMiniProfiler(options =>
    {
        options.RouteBasePath = "/profiler";
        options.ColorScheme = ColorScheme.Auto;
    }).AddEntityFramework();

    services.AddDbContext<ApplicationDbContext>(options =>
    {
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
        options.EnableSensitiveDataLogging();
    });

    services.AddControllers();
}

public void Configure(IApplicationBuilder app)
{
    app.UseMiniProfiler();

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### 自訂效能追蹤

```csharp
public class ProductService
{
    private readonly IProfiler _profiler;

    public async Task<List<Product>> GetPopularProductsAsync()
    {
        using (_profiler.Step("GetPopularProducts"))
        {
            List<Product> products;

            using (_profiler.Step("Database Query"))
            {
                products = await _context.Products
                    .Where(p => p.ViewCount > 1000)
                    .OrderByDescending(p => p.ViewCount)
                    .Take(10)
                    .ToListAsync();
            }

            using (_profiler.Step("Calculate Discount"))
            {
                foreach (var product in products)
                {
                    product.DiscountPrice = CalculateDiscount(product.Price);
                }
            }

            return products;
        }
    }
}
```

## 診斷工具

### dotnet-trace

```bash
# 安裝工具
dotnet tool install --global dotnet-trace

# 收集追蹤資料
dotnet trace collect --process-id <PID>

# 轉換為 Speedscope 格式
dotnet trace convert trace.nettrace --format Speedscope
```

### dotnet-counters

```bash
# 安裝工具
dotnet tool install --global dotnet-counters

# 即時監控
dotnet counters monitor --process-id <PID>

# 監控特定計數器
dotnet counters monitor --process-id <PID> System.Runtime

# 收集計數器資料
dotnet counters collect --process-id <PID> --output counters.csv
```

### dotnet-dump

```bash
# 安裝工具
dotnet tool install --global dotnet-dump

# 收集記憶體傾印
dotnet dump collect --process-id <PID>

# 分析僾印檔案
dotnet dump analyze core_20201006_135045
```

## 健康檢查

### 進階健康檢查

```csharp
// Startup.cs
services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>(
        name: "database",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "db", "sql" })
    .AddRedis(
        Configuration["Redis:Configuration"],
        name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "cache", "redis" })
    .AddUrlGroup(
        new Uri("http://product-service/health"),
        name: "product-service",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "service", "product" })
    .AddCheck<CustomHealthCheck>("custom-check");

// 設定不同端點
app.UseEndpoints(endpoints =>
{
    // 詳細檢查（內部使用）
    endpoints.MapHealthChecks("/health", new HealthCheckOptions
    {
        ResponseWriter = WriteHealthCheckResponse
    });

    // 簡單檢查（Load Balancer 使用）
    endpoints.MapHealthChecks("/health/ready", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("ready")
    });

    // 存活檢查
    endpoints.MapHealthChecks("/health/live", new HealthCheckOptions
    {
        Predicate = _ => false
    });

    endpoints.MapControllers();
});

// 自訂健康檢查回應
private static Task WriteHealthCheckResponse(HttpContext context, HealthReport report)
{
    context.Response.ContentType = "application/json";

    var result = JsonSerializer.Serialize(new
    {
        status = report.Status.ToString(),
        checks = report.Entries.Select(e => new
        {
            name = e.Key,
            status = e.Value.Status.ToString(),
            description = e.Value.Description,
            duration = e.Value.Duration.ToString(),
            exception = e.Value.Exception?.Message,
            data = e.Value.Data
        }),
        totalDuration = report.TotalDuration.ToString()
    }, new JsonSerializerOptions
    {
        WriteIndented = true
    });

    return context.Response.WriteAsync(result);
}
```

### 自訂健康檢查

```csharp
public class CustomHealthCheck : IHealthCheck
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<CustomHealthCheck> _logger;

    public CustomHealthCheck(
        IHttpClientFactory httpClientFactory,
        ILogger<CustomHealthCheck> logger)
    {
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var client = _httpClientFactory.CreateClient();
            var response = await client.GetAsync(
                "http://external-api/health",
                cancellationToken);

            if (response.IsSuccessStatusCode)
            {
                return HealthCheckResult.Healthy("外部 API 正常運作");
            }

            return HealthCheckResult.Degraded(
                $"外部 API 回應異常: {response.StatusCode}");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "健康檢查失敗");
            return HealthCheckResult.Unhealthy(
                "無法連線至外部 API",
                ex);
        }
    }
}
```

## 效能指標收集

### 自訂 EventCounter

```csharp
[EventSource(Name = "MyApp-ProductService")]
public sealed class ProductServiceEventSource : EventSource
{
    public static readonly ProductServiceEventSource Instance = new ProductServiceEventSource();

    private readonly EventCounter _productRetrievedCounter;
    private readonly EventCounter _searchDurationCounter;

    private ProductServiceEventSource()
    {
        _productRetrievedCounter = new EventCounter("products-retrieved", this)
        {
            DisplayName = "Products Retrieved",
            DisplayUnits = "count"
        };

        _searchDurationCounter = new EventCounter("search-duration", this)
        {
            DisplayName = "Search Duration",
            DisplayUnits = "ms"
        };
    }

    public void ProductRetrieved()
    {
        _productRetrievedCounter.WriteMetric(1);
    }

    public void SearchCompleted(double durationMs)
    {
        _searchDurationCounter.WriteMetric(durationMs);
    }

    protected override void Dispose(bool disposing)
    {
        _productRetrievedCounter?.Dispose();
        _searchDurationCounter?.Dispose();
        base.Dispose(disposing);
    }
}

// 使用
public class ProductService
{
    public async Task<Product> GetByIdAsync(int id)
    {
        var product = await _repository.GetByIdAsync(id);
        
        ProductServiceEventSource.Instance.ProductRetrieved();
        
        return product;
    }

    public async Task<List<Product>> SearchAsync(string keyword)
    {
        var stopwatch = Stopwatch.StartNew();
        
        var products = await _repository.SearchAsync(keyword);
        
        stopwatch.Stop();
        ProductServiceEventSource.Instance.SearchCompleted(stopwatch.ElapsedMilliseconds);
        
        return products;
    }
}
```

### 收集 EventCounter

```bash
dotnet counters monitor --process-id <PID> --counters MyApp-ProductService
```

## 警報設定

### Application Insights 警報

```json
// 在 Azure Portal 設定警報規則
{
  "condition": {
    "allOf": [
      {
        "metricName": "requests/failed",
        "operator": "GreaterThan",
        "threshold": 10,
        "timeAggregation": "Total"
      }
    ]
  },
  "actions": [
    {
      "actionGroupId": "/subscriptions/.../actionGroups/myActionGroup"
    }
  ]
}
```

### Prometheus 警報規則

```yaml
# prometheus-rules.yml
groups:
  - name: myapp
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "高錯誤率警報"
          description: "{{ $labels.instance }} 的錯誤率超過 5%"

      - alert: SlowResponse
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "回應時間過慢"
          description: "95% 的請求回應時間超過 1 秒"
```

## 小結

.NET Core 效能監控：
- Application Insights 提供完整的 APM 功能
- Prometheus + Grafana 開源監控方案
- Serilog 結構化日誌
- 內建診斷工具強大

相較於 Spring Boot：
- Application Insights 整合更簡單
- dotnet-trace/counters/dump 比 JVM 工具更輕量
- EventCounter 比 Micrometer 更容易使用
- 健康檢查機制更靈活

下週將探討 .NET Core 安全性最佳實踐。
