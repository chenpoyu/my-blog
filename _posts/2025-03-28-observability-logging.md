---
layout: post
title: "可觀測性：Log 要寫什麼、怎麼寫、寫多少"
date: 2025-03-28 11:10:00 +0800
categories: [維運]
tags: [Logging, Application Insights, 可觀測性, .NET Core]
---

## 半夜被叫起來

上週五晚上 11 點，手機響了。

是監控系統發的告警：「API 錯誤率超過 5%」。

我打開 Azure Portal，看到 Application Insights 的錯誤圖表一片紅。

但除了「500 Internal Server Error」之外，什麼資訊都沒有。

不知道是哪個 API 出錯、不知道錯誤原因、不知道影響多少使用者。

我只好 VPN 連進去，查 log⋯⋯然後發現 log 裡也沒什麼有用的資訊。

花了一個小時才找到問題：Redis 連線突然斷掉，但程式沒有正確處理 exception，導致所有需要快取的 API 都掛掉。

修好之後，我決定：**要好好整理 logging 策略**。

## Logging 的三個層次

很多人以為 logging 就是：「出錯了就 `logger.LogError()`，正常就 `logger.LogInformation()`。」

這樣寫出來的 log 通常不太有用。

好的 logging 要有三個層次：

### 1. Tracing：追蹤請求流程

當一個 API 請求進來,它可能會經過：
1. API Controller
2. Application Service
3. gRPC Client（呼叫其他模組）
4. Repository
5. Database

每個步驟都要記錄，才能知道請求在哪裡卡住、在哪裡出錯。

```csharp
public class CouponService
{
    private readonly ILogger<CouponService> _logger;
    private readonly IMemberGrpcClient _memberClient;
    
    public async Task<List<CouponDto>> GetAvailableCouponsAsync(string memberId)
    {
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["MemberId"] = memberId,
            ["Operation"] = "GetAvailableCoupons"
        });
        
        _logger.LogInformation("開始查詢可用優惠券");
        
        try
        {
            // 呼叫 Member 模組
            _logger.LogDebug("呼叫 Member 模組查詢會員資訊");
            var member = await _memberClient.GetMemberAsync(memberId);
            
            if (!member.IsActive)
            {
                _logger.LogInformation("會員未啟用，回傳空列表");
                return new List<CouponDto>();
            }
            
            // 查詢優惠券
            _logger.LogDebug("查詢會員等級 {Level} 的優惠券", member.Level);
            var coupons = await _couponRepository.GetByMemberLevelAsync(member.Level);
            
            _logger.LogInformation("查詢完成，找到 {Count} 張優惠券", coupons.Count);
            
            return coupons;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "查詢優惠券失敗");
            throw;
        }
    }
}
```

這樣寫的好處：
- 每個步驟都有記錄
- 用 `BeginScope` 把相關的 log 關聯起來
- Exception 會自動帶上 stack trace

在 Application Insights 可以看到完整的請求流程：

```
[INFO] 開始查詢可用優惠券 (MemberId=M001, Operation=GetAvailableCoupons)
[DEBUG] 呼叫 Member 模組查詢會員資訊 (MemberId=M001)
[DEBUG] 查詢會員等級 Gold 的優惠券 (MemberId=M001)
[INFO] 查詢完成，找到 3 張優惠券 (MemberId=M001)
```

### 2. Metrics：量化指標

Log 是「發生了什麼事」，Metrics 是「數字上的變化」。

比如：
- API 回應時間
- 錯誤率
- 快取命中率
- 資料庫查詢次數

這些要用 metrics 記錄，不是 log。

```csharp
public class CouponService
{
    private readonly ILogger<CouponService> _logger;
    private readonly IMemberGrpcClient _memberClient;
    private readonly IMetrics _metrics;
    
    public async Task<List<CouponDto>> GetAvailableCouponsAsync(string memberId)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var member = await _memberClient.GetMemberAsync(memberId);
            var coupons = await _couponRepository.GetByMemberLevelAsync(member.Level);
            
            // 記錄成功的 metric
            _metrics.Increment("coupons.query.success");
            _metrics.Histogram("coupons.query.duration", stopwatch.ElapsedMilliseconds);
            
            return coupons;
        }
        catch (Exception ex)
        {
            // 記錄失敗的 metric
            _metrics.Increment("coupons.query.error");
            _logger.LogError(ex, "查詢優惠券失敗");
            throw;
        }
    }
}
```

在 Application Insights 可以畫圖表：
- 過去 24 小時的查詢成功率
- P50/P95/P99 回應時間
- 錯誤數趨勢

這些數字比 log 更直觀，更容易發現問題。

### 3. Events：重要事件

有些事件特別重要，要單獨記錄：
- 使用者註冊
- 交易完成
- 系統設定變更
- 安全相關事件（登入失敗、權限異常）

這些不只是 log，而是 **business event**。

```csharp
public async Task<RegisterResult> RegisterAsync(RegisterRequest request)
{
    var member = await _memberRepository.CreateAsync(request);
    
    // 記錄 business event
    _logger.LogInformation(
        "會員註冊成功：{MemberId}, {Name}, {Email}", 
        member.Id, 
        member.Name, 
        member.Email);
    
    // 發送到 event tracking system
    await _eventTracker.TrackAsync(new
    {
        EventName = "MemberRegistered",
        MemberId = member.Id,
        RegisterSource = request.Source,
        Timestamp = DateTime.UtcNow
    });
    
    return new RegisterResult { Success = true };
}
```

這些 event 可以用來：
- 分析使用者行為
- 計算轉換率
- 追蹤業務指標

## Log Level 的使用原則

.NET 有六個 log level：

- **Trace**：最詳細，幾乎所有動作
- **Debug**：除錯用的資訊
- **Information**：正常流程的重要步驟
- **Warning**：可能有問題，但不影響運作
- **Error**：發生錯誤，但系統還能繼續
- **Critical**：嚴重錯誤，系統可能無法運作

我們的使用原則：

### Trace
本機開發用，不要打到正式環境。

```csharp
_logger.LogTrace("進入方法：GetAvailableCouponsAsync, 參數：{Params}", JsonSerializer.Serialize(request));
```

### Debug
測試環境用，正式環境關掉。

```csharp
_logger.LogDebug("快取未命中，查詢資料庫");
_logger.LogDebug("gRPC 呼叫耗時：{Duration}ms", duration);
```

### Information
正常流程的關鍵步驟，正式環境會記錄。

```csharp
_logger.LogInformation("會員註冊成功：{MemberId}", memberId);
_logger.LogInformation("訂單建立完成：{OrderId}, 金額：{Amount}", orderId, amount);
```

### Warning
可能有問題，但不影響使用者。

```csharp
_logger.LogWarning("快取連線失敗，fallback 到資料庫");
_logger.LogWarning("API 回應時間過長：{Duration}ms", duration);
```

### Error
發生錯誤，影響使用者。

```csharp
_logger.LogError(ex, "資料庫查詢失敗");
_logger.LogError(ex, "gRPC 呼叫逾時");
```

### Critical
系統層級的嚴重錯誤。

```csharp
_logger.LogCritical(ex, "資料庫連線池耗盡");
_logger.LogCritical(ex, "Redis 完全無法連線");
```

Critical 的 log 會立刻發送告警簡訊給 on-call 工程師。

## Structured Logging

不要用字串拼接寫 log：

```csharp
// ❌ 不好
_logger.LogInformation("會員 " + memberId + " 註冊成功");
```

要用 structured logging：

```csharp
// ✅ 好
_logger.LogInformation("會員註冊成功：{MemberId}", memberId);
```

這樣 Application Insights 可以把 `MemberId` 當成 field，方便查詢：

```sql
traces
| where customDimensions.MemberId == "M001"
| order by timestamp desc
```

更進階的寫法：

```csharp
using var scope = _logger.BeginScope(new Dictionary<string, object>
{
    ["MemberId"] = memberId,
    ["OperationId"] = operationId,
    ["UserAgent"] = userAgent
});

_logger.LogInformation("開始處理請求");
// ... 其他 log
_logger.LogInformation("請求處理完成");
```

所有在這個 scope 裡的 log 都會帶上 `MemberId`、`OperationId`、`UserAgent`，不用每次都寫。

## 敏感資訊的處理

有些資訊不能寫進 log：
- 密碼
- 信用卡號
- 身分證字號
- API Token

要特別小心：

```csharp
// ❌ 不好，密碼會被記錄
_logger.LogInformation("登入請求：{Request}", JsonSerializer.Serialize(loginRequest));

// ✅ 好，只記錄必要資訊
_logger.LogInformation("登入請求：帳號={Account}", loginRequest.Account);
```

我們寫了一個 extension method 來自動遮蔽敏感資訊：

```csharp
public static class LoggerExtensions
{
    private static readonly string[] SensitiveFields = 
        { "password", "creditCard", "token", "secret" };
    
    public static void LogSafeInformation(
        this ILogger logger, 
        string message, 
        object data)
    {
        var sanitized = SanitizeObject(data);
        logger.LogInformation(message, sanitized);
    }
    
    private static object SanitizeObject(object obj)
    {
        // 遞迴檢查 object 的所有 property
        // 如果 property name 包含敏感字眼，用 "***" 取代
        // ...
    }
}
```

## Application Insights 的設定

Azure 的 Application Insights 很強大，但要設定好：

```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    options.EnableAdaptiveSampling = true; // 自動取樣，避免 log 太多
    options.EnableQuickPulseMetricStream = true; // 即時監控
});

// 設定 log level
builder.Logging.AddApplicationInsights();
builder.Logging.AddFilter<ApplicationInsightsLoggerProvider>(
    "", 
    LogLevel.Information); // 只記錄 Information 以上的 log
```

取樣（Sampling）很重要。

如果每個 request 都記錄完整的 log，流量大的時候會產生巨量的資料：
- Azure 會收很貴的費用
- 查詢會變慢
- 真正重要的 log 被淹沒

啟用 adaptive sampling 後，Application Insights 會自動：
- 正常情況：取樣 10%（只記錄 10% 的 request）
- 錯誤發生：取樣 100%（所有錯誤都記錄）

這樣既省錢，又不會漏掉重要的錯誤。

## 告警設定

有了 log 和 metrics,還要設定告警：

```yaml
# Application Insights Alert Rules

1. API 錯誤率超過 5%
   - 條件：過去 5 分鐘的錯誤率 > 5%
   - 動作：發送 Email + Slack 通知

2. API 回應時間超過 1 秒
   - 條件：P95 回應時間 > 1000ms
   - 動作：發送 Slack 通知

3. 資料庫連線失敗
   - 條件：出現 "database connection failed" 的 log
   - 動作：發送簡訊 + Email（Critical 等級）

4. Redis 連線失敗
   - 條件：出現 "redis connection failed" 的 log
   - 動作：發送 Slack 通知（Warning 等級，因為有 fallback）
```

告警不要太敏感，不然會「狼來了」。

我們的原則：
- **Critical**：立刻處理，發簡訊
- **Error**：30 分鐘內處理，發 Email
- **Warning**：隔天處理，發 Slack

## 目前的效果

整理 logging 策略後，除錯速度提升很多。

上次又發生一次 API 錯誤,我打開 Application Insights：

1. 看 metrics：發現是 Points 模組的錯誤率飆高
2. 看 trace：發現是資料庫 timeout
3. 看 SQL query log：發現是某個查詢沒加索引，掃描了 10 萬筆資料

從發現問題到找到原因，只花了 **5 分鐘**。

如果是之前的 logging,可能要花一個小時。

## 接下來要做的

Logging 做得還不夠好，還要改進：

1. **Distributed Tracing**：用 OpenTelemetry 追蹤跨模組的請求
2. **Log Aggregation**：把所有 log 集中到 Elasticsearch，方便查詢
3. **Anomaly Detection**：用 AI 自動偵測異常的 log pattern

這些是進階的功能，會在接下來幾個月慢慢做。

專案還剩四個月，維運這塊要越來越完善。

下週會寫效能優化的經驗，特別是怎麼找出 bottleneck。
