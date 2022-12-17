---
layout: post
title: "ASP.NET Core 背景服務與定時任務"
date: 2020-08-11 14:25:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Background Services, Hosted Services, Quartz.NET]
---

這週研究 ASP.NET Core 的背景服務與定時任務機制，包含 `IHostedService` 和 Quartz.NET 排程框架。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Spring Boot 排程任務回顧

在 Spring Boot 中，我們使用 `@Scheduled` 處理定時任務：

```java
@Component
@EnableScheduling
public class ScheduledTasks {
    
    @Scheduled(fixedRate = 5000)
    public void reportCurrentTime() {
        System.out.println("Current time: " + LocalDateTime.now());
    }
    
    @Scheduled(cron = "0 0 2 * * ?")
    public void performBackup() {
        // 每天凌晨 2 點執行備份
    }
}
```

## IHostedService 基礎

ASP.NET Core 使用 `IHostedService` 實現背景服務：

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

### 簡單背景服務

```csharp
public class SimpleBackgroundService : IHostedService
{
    private readonly ILogger<SimpleBackgroundService> _logger;
    private Timer _timer;

    public SimpleBackgroundService(ILogger<SimpleBackgroundService> logger)
    {
        _logger = logger;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("背景服務啟動");

        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(5));

        return Task.CompletedTask;
    }

    private void DoWork(object state)
    {
        _logger.LogInformation("執行背景工作: {Time}", DateTime.Now);
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("背景服務停止");

        _timer?.Change(Timeout.Infinite, 0);
        _timer?.Dispose();

        return Task.CompletedTask;
    }
}

// Startup.cs 註冊
services.AddHostedService<SimpleBackgroundService>();
```

## BackgroundService 基類

`BackgroundService` 是 `IHostedService` 的抽象實作，簡化了背景服務開發：

```csharp
public class TimedBackgroundService : BackgroundService
{
    private readonly ILogger<TimedBackgroundService> _logger;
    private readonly IServiceProvider _serviceProvider;

    public TimedBackgroundService(
        ILogger<TimedBackgroundService> logger,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("定時背景服務啟動");

        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("執行定時任務: {Time}", DateTime.Now);

            try
            {
                await DoWorkAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "執行背景任務時發生錯誤");
            }

            // 每 5 秒執行一次
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }

        _logger.LogInformation("定時背景服務停止");
    }

    private async Task DoWorkAsync()
    {
        // 建立 scope 來解析 scoped services
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider
            .GetRequiredService<ApplicationDbContext>();

        var count = await dbContext.Products.CountAsync();
        _logger.LogInformation("資料庫中有 {Count} 個產品", count);
    }
}

services.AddHostedService<TimedBackgroundService>();
```

## 使用 Channel 處理背景佇列

### 背景佇列服務

```csharp
public interface IBackgroundTaskQueue
{
    ValueTask QueueBackgroundWorkItemAsync(Func<CancellationToken, ValueTask> workItem);
    ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(
        CancellationToken cancellationToken);
}

public class BackgroundTaskQueue : IBackgroundTaskQueue
{
    private readonly Channel<Func<CancellationToken, ValueTask>> _queue;

    public BackgroundTaskQueue(int capacity)
    {
        var options = new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait
        };
        _queue = Channel.CreateBounded<Func<CancellationToken, ValueTask>>(options);
    }

    public async ValueTask QueueBackgroundWorkItemAsync(
        Func<CancellationToken, ValueTask> workItem)
    {
        if (workItem == null)
        {
            throw new ArgumentNullException(nameof(workItem));
        }

        await _queue.Writer.WriteAsync(workItem);
    }

    public async ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(
        CancellationToken cancellationToken)
    {
        var workItem = await _queue.Reader.ReadAsync(cancellationToken);
        return workItem;
    }
}

// 註冊
services.AddSingleton<IBackgroundTaskQueue>(new BackgroundTaskQueue(100));
```

### 佇列處理服務

```csharp
public class QueuedHostedService : BackgroundService
{
    private readonly ILogger<QueuedHostedService> _logger;
    private readonly IBackgroundTaskQueue _taskQueue;

    public QueuedHostedService(
        IBackgroundTaskQueue taskQueue,
        ILogger<QueuedHostedService> logger)
    {
        _taskQueue = taskQueue;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("佇列處理服務啟動");

        await BackgroundProcessing(stoppingToken);
    }

    private async Task BackgroundProcessing(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var workItem = await _taskQueue.DequeueAsync(stoppingToken);

                await workItem(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // 服務停止時會拋出此例外
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "處理佇列任務時發生錯誤");
            }
        }

        _logger.LogInformation("佇列處理服務停止");
    }
}

services.AddHostedService<QueuedHostedService>();
```

### 在 Controller 中使用

```csharp
[ApiController]
[Route("api/[controller]")]
public class EmailsController : ControllerBase
{
    private readonly IBackgroundTaskQueue _taskQueue;
    private readonly ILogger<EmailsController> _logger;

    public EmailsController(
        IBackgroundTaskQueue taskQueue,
        ILogger<EmailsController> logger)
    {
        _taskQueue = taskQueue;
        _logger = logger;
    }

    [HttpPost("send")]
    public async Task<IActionResult> SendEmail([FromBody] EmailRequest request)
    {
        // 將工作加入背景佇列
        await _taskQueue.QueueBackgroundWorkItemAsync(async token =>
        {
            _logger.LogInformation("開始傳送郵件至 {Email}", request.To);
            
            await Task.Delay(5000, token); // 模擬傳送郵件
            
            _logger.LogInformation("郵件已傳送至 {Email}", request.To);
        });

        return Ok(new { message = "郵件已加入傳送佇列" });
    }
}
```

## Quartz.NET 排程框架

### 安裝套件

```bash
dotnet add package Quartz
dotnet add package Quartz.Extensions.Hosting
```

### 定義 Job

```csharp
public class DataSyncJob : IJob
{
    private readonly ILogger<DataSyncJob> _logger;
    private readonly IServiceProvider _serviceProvider;

    public DataSyncJob(
        ILogger<DataSyncJob> logger,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        _logger.LogInformation("開始執行資料同步作業");

        try
        {
            using var scope = _serviceProvider.CreateScope();
            var dbContext = scope.ServiceProvider
                .GetRequiredService<ApplicationDbContext>();

            // 執行資料同步
            var products = await dbContext.Products
                .Where(p => p.UpdatedAt >= DateTime.Now.AddDays(-1))
                .ToListAsync();

            _logger.LogInformation("同步了 {Count} 個產品", products.Count);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "資料同步失敗");
            throw;
        }
    }
}
```

### 設定 Quartz

```csharp
// Startup.cs
services.AddQuartz(q =>
{
    // 使用 DI 容器來建立 Job
    q.UseMicrosoftDependencyInjectionJobFactory();

    // 定義 Job
    var jobKey = new JobKey("DataSyncJob");
    q.AddJob<DataSyncJob>(opts => opts.WithIdentity(jobKey));

    // 定義 Trigger - 每 5 分鐘執行一次
    q.AddTrigger(opts => opts
        .ForJob(jobKey)
        .WithIdentity("DataSyncJob-trigger")
        .WithSimpleSchedule(x => x
            .WithIntervalInMinutes(5)
            .RepeatForever()));

    // Cron 排程 - 每天凌晨 2 點執行
    q.AddTrigger(opts => opts
        .ForJob(jobKey)
        .WithIdentity("DataSyncJob-cron-trigger")
        .WithCronSchedule("0 0 2 * * ?"));
});

// 加入 Quartz 背景服務
services.AddQuartzHostedService(options =>
{
    // 等待 Job 執行完畢才關閉應用程式
    options.WaitForJobsToComplete = true;
});
```

### 複雜的 Job 範例

```csharp
public class ReportGenerationJob : IJob
{
    private readonly ILogger<ReportGenerationJob> _logger;
    private readonly IEmailService _emailService;
    private readonly IReportService _reportService;

    public ReportGenerationJob(
        ILogger<ReportGenerationJob> logger,
        IEmailService emailService,
        IReportService reportService)
    {
        _logger = logger;
        _emailService = emailService;
        _reportService = reportService;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        var dataMap = context.JobDetail.JobDataMap;
        var reportType = dataMap.GetString("ReportType");
        var recipientEmail = dataMap.GetString("RecipientEmail");

        _logger.LogInformation(
            "開始產生 {ReportType} 報表，收件者: {Email}",
            reportType, recipientEmail);

        try
        {
            // 產生報表
            var report = await _reportService.GenerateReportAsync(reportType);

            // 傳送郵件
            await _emailService.SendReportAsync(recipientEmail, report);

            _logger.LogInformation("報表已成功傳送至 {Email}", recipientEmail);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "產生或傳送報表失敗");
            
            // 重新排程
            var trigger = TriggerBuilder.Create()
                .ForJob(context.JobDetail)
                .StartAt(DateTimeOffset.Now.AddMinutes(5))
                .Build();

            await context.Scheduler.RescheduleJob(
                context.Trigger.Key, trigger);
        }
    }
}

// 設定帶參數的 Job
q.AddJob<ReportGenerationJob>(opts => opts
    .WithIdentity("ReportJob")
    .UsingJobData("ReportType", "Daily")
    .UsingJobData("RecipientEmail", "admin@example.com"));
```

### Cron 表達式

```csharp
// 每天早上 8:00
"0 0 8 * * ?"

// 每週一早上 9:00
"0 0 9 ? * MON"

// 每月 1 號凌晨 1:00
"0 0 1 1 * ?"

// 每 15 分鐘
"0 */15 * * * ?"

// 工作日的 9:00-18:00，每小時執行
"0 0 9-18 ? * MON-FRI"

// 範例設定
services.AddQuartz(q =>
{
    // 每天早上 8:00 產生日報表
    q.AddJob<DailyReportJob>(opts => opts.WithIdentity("DailyReport"))
        .AddTrigger(opts => opts
            .ForJob("DailyReport")
            .WithCronSchedule("0 0 8 * * ?"));

    // 每週一早上 9:00 產生週報表
    q.AddJob<WeeklyReportJob>(opts => opts.WithIdentity("WeeklyReport"))
        .AddTrigger(opts => opts
            .ForJob("WeeklyReport")
            .WithCronSchedule("0 0 9 ? * MON"));

    // 每月 1 號凌晨 1:00 產生月報表
    q.AddJob<MonthlyReportJob>(opts => opts.WithIdentity("MonthlyReport"))
        .AddTrigger(opts => opts
            .ForJob("MonthlyReport")
            .WithCronSchedule("0 0 1 1 * ?"));
});
```

## 健康檢查整合

```csharp
public class BackgroundServiceHealthCheck : IHealthCheck
{
    private readonly IBackgroundTaskQueue _taskQueue;

    public BackgroundServiceHealthCheck(IBackgroundTaskQueue taskQueue)
    {
        _taskQueue = taskQueue;
    }

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        // 檢查佇列狀態
        try
        {
            // 可以加入更複雜的檢查邏輯
            return Task.FromResult(
                HealthCheckResult.Healthy("背景服務運作正常"));
        }
        catch (Exception ex)
        {
            return Task.FromResult(
                HealthCheckResult.Unhealthy("背景服務異常", ex));
        }
    }
}

services.AddHealthChecks()
    .AddCheck<BackgroundServiceHealthCheck>("background_service");
```

## 實際應用範例

### 定期清理過期資料

```csharp
public class DataCleanupService : BackgroundService
{
    private readonly ILogger<DataCleanupService> _logger;
    private readonly IServiceProvider _serviceProvider;

    public DataCleanupService(
        ILogger<DataCleanupService> logger,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await CleanupExpiredDataAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "清理資料時發生錯誤");
            }

            // 每天執行一次
            await Task.Delay(TimeSpan.FromDays(1), stoppingToken);
        }
    }

    private async Task CleanupExpiredDataAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider
            .GetRequiredService<ApplicationDbContext>();

        var expiredDate = DateTime.Now.AddMonths(-3);

        var expiredLogs = await dbContext.AuditLogs
            .Where(log => log.CreatedAt < expiredDate)
            .ToListAsync();

        dbContext.AuditLogs.RemoveRange(expiredLogs);
        await dbContext.SaveChangesAsync();

        _logger.LogInformation(
            "清理了 {Count} 筆過期日誌",
            expiredLogs.Count);
    }
}
```

### 檔案監視服務

```csharp
public class FileWatcherService : BackgroundService
{
    private readonly ILogger<FileWatcherService> _logger;
    private readonly string _watchPath;
    private FileSystemWatcher _watcher;

    public FileWatcherService(
        ILogger<FileWatcherService> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _watchPath = configuration["FileWatcher:Path"];
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _watcher = new FileSystemWatcher(_watchPath)
        {
            NotifyFilter = NotifyFilters.FileName | NotifyFilters.Size,
            Filter = "*.csv"
        };

        _watcher.Created += OnFileCreated;
        _watcher.EnableRaisingEvents = true;

        _logger.LogInformation("檔案監視服務啟動，監視路徑: {Path}", _watchPath);

        return Task.CompletedTask;
    }

    private async void OnFileCreated(object sender, FileSystemEventArgs e)
    {
        _logger.LogInformation("偵測到新檔案: {FileName}", e.Name);

        try
        {
            // 處理檔案
            await ProcessFileAsync(e.FullPath);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "處理檔案 {FileName} 時發生錯誤", e.Name);
        }
    }

    private async Task ProcessFileAsync(string filePath)
    {
        // 檔案處理邏輯
        await Task.Delay(1000);
        _logger.LogInformation("檔案處理完成: {FilePath}", filePath);
    }

    public override void Dispose()
    {
        _watcher?.Dispose();
        base.Dispose();
    }
}
```

### 快取預熱服務

```csharp
public class CacheWarmupService : IHostedService
{
    private readonly ILogger<CacheWarmupService> _logger;
    private readonly IMemoryCache _cache;
    private readonly IServiceProvider _serviceProvider;

    public CacheWarmupService(
        ILogger<CacheWarmupService> logger,
        IMemoryCache cache,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _cache = cache;
        _serviceProvider = serviceProvider;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("開始快取預熱");

        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider
            .GetRequiredService<ApplicationDbContext>();

        // 載入熱門資料至快取
        var hotProducts = await dbContext.Products
            .Where(p => p.ViewCount > 1000)
            .ToListAsync(cancellationToken);

        foreach (var product in hotProducts)
        {
            var cacheKey = $"product_{product.Id}";
            _cache.Set(cacheKey, product, TimeSpan.FromHours(1));
        }

        _logger.LogInformation(
            "快取預熱完成，載入了 {Count} 個產品",
            hotProducts.Count);
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("快取預熱服務停止");
        return Task.CompletedTask;
    }
}

services.AddHostedService<CacheWarmupService>();
```

## 監控與管理

### Job 狀態監控

```csharp
[ApiController]
[Route("api/[controller]")]
public class JobsController : ControllerBase
{
    private readonly ISchedulerFactory _schedulerFactory;

    public JobsController(ISchedulerFactory schedulerFactory)
    {
        _schedulerFactory = schedulerFactory;
    }

    [HttpGet]
    public async Task<IActionResult> GetJobs()
    {
        var scheduler = await _schedulerFactory.GetScheduler();
        var jobKeys = await scheduler.GetJobKeys(GroupMatcher<JobKey>.AnyGroup());

        var jobs = new List<object>();

        foreach (var jobKey in jobKeys)
        {
            var detail = await scheduler.GetJobDetail(jobKey);
            var triggers = await scheduler.GetTriggersOfJob(jobKey);

            jobs.Add(new
            {
                JobName = jobKey.Name,
                JobGroup = jobKey.Group,
                Triggers = triggers.Select(t => new
                {
                    TriggerName = t.Key.Name,
                    NextFireTime = t.GetNextFireTimeUtc(),
                    PreviousFireTime = t.GetPreviousFireTimeUtc()
                })
            });
        }

        return Ok(jobs);
    }

    [HttpPost("{jobName}/trigger")]
    public async Task<IActionResult> TriggerJob(string jobName)
    {
        var scheduler = await _schedulerFactory.GetScheduler();
        var jobKey = new JobKey(jobName);

        if (!await scheduler.CheckExists(jobKey))
        {
            return NotFound(new { message = $"找不到 Job: {jobName}" });
        }

        await scheduler.TriggerJob(jobKey);
        return Ok(new { message = $"已觸發 Job: {jobName}" });
    }
}
```

## 小結

ASP.NET Core 背景服務特點：
- `IHostedService` 與 `BackgroundService` 提供簡潔的 API
- Channel 提供高效能的佇列處理
- Quartz.NET 提供強大的排程功能
- 完整的 DI 整合

相較於 Spring Boot：
- `BackgroundService` 比 `@Scheduled` 更靈活
- Quartz.NET 比 Spring Scheduler 功能更豐富
- Channel 效能優於 `BlockingQueue`
- 生命週期管理更清晰

下週將探討 ASP.NET Core 的模型驗證與資料註解。
