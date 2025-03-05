---
layout: post
title: "模組間通訊的選擇：為什麼我們用 gRPC 而不是 Message Queue"
date: 2025-03-06 14:30:00 +0800
categories: [架構設計]
tags: [gRPC, 微模組, .NET Core, 模組通訊]
---

## 一個簡單的需求

上週在開發優惠券模組的時候，前端同事問了一個問題：

「使用者結帳的時候，要即時顯示可用的優惠券和折扣金額。這個 API 要多快？」

「越快越好吧，」我說，「使用者不會想等太久。」

「那⋯⋯500ms 內可以嗎?」

我想了想結帳流程需要做的事：
1. 檢查會員是否有效（呼叫 Member 模組）
2. 查詢可用優惠券（Coupon 模組本身）
3. 確認點數餘額（呼叫 Points 模組）
4. 計算最佳折扣組合
5. 回傳結果

「可以，但要用 gRPC。」

## 團隊的質疑

當我跟團隊說要用 gRPC 做模組間通訊時，有個工程師提出疑問：

「為什麼不用 message queue？我看很多微服務架構都是用 event-driven 的方式，用 RabbitMQ 或 Azure Service Bus 來通訊。」

這是個好問題。Event-driven 架構確實是現在的主流，解耦做得也很好。

但我問他：「如果結帳 API 要等 message queue 的回應，會發生什麼事？」

他想了想：「會變成非同步⋯⋯」

「對，但使用者在等。他按下『使用優惠券』的按鈕，畫面要立刻顯示折扣後的金額。不能跳一個『處理中，請稍後』的訊息。」

## 同步 vs 非同步的抉擇

這就是關鍵：**有些操作必須是同步的**。

Message queue 很適合這些情境：
- 發送通知（簡訊、Email、推播）
- 記錄 log
- 資料同步（備份、匯出）
- 背景任務（報表產生、圖片壓縮）

這些事情不需要「立刻」完成，可以慢慢處理。用 message queue 可以：
- 削峰填谷（流量高峰時排隊處理）
- 失敗重試（訊息沒處理完可以重送）
- 完全解耦（發送者不需要知道接收者是誰）

但結帳、查詢、驗證這類操作，使用者在等結果，**必須在幾百毫秒內回應**。

如果用 message queue：
```
API → 發送 message → Queue → Consumer 接收 → 處理 → 發送 response message → Queue → API 接收
```

來回至少要好幾百毫秒，甚至上秒。使用者體驗會很差。

## gRPC 的優勢

gRPC 是 Google 開發的 RPC（Remote Procedure Call）框架，用 Protocol Buffers 作為序列化格式。

在我們的微模組架構裡，雖然模組都在同一個 process，但用 gRPC 的方式定義介面，有幾個好處：

### 1. 效能好

Protocol Buffers 是 binary 格式，比 JSON 小很多，序列化也更快。

```csharp
// Member 模組定義 .proto
syntax = "proto3";

service MemberService {
  rpc GetMember (GetMemberRequest) returns (GetMemberResponse);
  rpc CheckMemberStatus (CheckStatusRequest) returns (CheckStatusResponse);
}

message GetMemberRequest {
  string member_id = 1;
}

message GetMemberResponse {
  string member_id = 1;
  string name = 2;
  string level = 3;
  bool is_active = 4;
  int32 points = 5;
}
```

在同一個 process 裡，gRPC 的 call 幾乎沒有延遲。我們實測結帳 API 從呼叫 Member、Points 到回應，總共不到 50ms。

### 2. 強型別

.proto 檔案定義了清楚的 contract，編譯時就會產生對應的 C# class。

```csharp
// Coupon 模組呼叫 Member 服務
public class CouponService
{
    private readonly MemberService.MemberServiceClient _memberClient;
    
    public CouponService(MemberService.MemberServiceClient memberClient)
    {
        _memberClient = memberClient;
    }
    
    public async Task<List<CouponDto>> GetAvailableCouponsAsync(string memberId)
    {
        // 透過 gRPC 查詢會員資訊
        var memberResponse = await _memberClient.GetMemberAsync(
            new GetMemberRequest { MemberId = memberId });
        
        if (!memberResponse.IsActive)
        {
            return new List<CouponDto>(); // 會員未啟用，不給優惠券
        }
        
        // 根據會員等級查詢可用優惠券
        var coupons = await _couponRepository.GetByMemberLevelAsync(memberResponse.Level);
        
        return coupons;
    }
}
```

IDE 會有完整的 IntelliSense，不會拼錯欄位名稱。而且如果 Member 模組改了欄位，Coupon 模組這邊編譯會失敗，馬上就知道要改。

### 3. 未來可以拆

現在所有模組在同一個 process，gRPC call 是 in-process 的。

但如果未來真的要拆成微服務，只要把 Member 模組獨立部署，改一下 gRPC 的連線設定就好：

```csharp
// 現在：in-process
builder.Services.AddGrpcClient<MemberService.MemberServiceClient>(o =>
{
    o.Address = new Uri("http://localhost"); // 本機
});

// 未來：remote service
builder.Services.AddGrpcClient<MemberService.MemberServiceClient>(o =>
{
    o.Address = new Uri("https://member-service.myapp.com"); // 遠端服務
});
```

Coupon 模組的程式碼完全不用改，只要改設定檔。

## 什麼時候用 Message Queue？

雖然我們模組間用 gRPC，但不代表完全不用 message queue。

有些情境還是會用非同步通訊：

### 1. 通知發送

會員註冊成功後，要發歡迎簡訊、Email、推播通知。這些不用即時完成。

```csharp
// Member 模組
public async Task<RegisterResult> RegisterAsync(RegisterRequest request)
{
    // 建立會員
    var member = await _memberRepository.CreateAsync(request);
    
    // 發送 event 到 message queue
    await _messageBus.PublishAsync(new MemberRegisteredEvent
    {
        MemberId = member.Id,
        Phone = member.Phone,
        Email = member.Email,
        RegisteredAt = DateTime.UtcNow
    });
    
    return new RegisterResult { Success = true };
}

// Notification 模組監聽 event
public class MemberRegisteredHandler : IEventHandler<MemberRegisteredEvent>
{
    private readonly ISmsService _smsService;
    private readonly IEmailService _emailService;
    
    public async Task HandleAsync(MemberRegisteredEvent @event)
    {
        // 發送簡訊
        await _smsService.SendAsync(@event.Phone, "歡迎加入！");
        
        // 發送 Email
        await _emailService.SendAsync(@event.Email, "歡迎信範本");
    }
}
```

用 message queue 的好處是：
- Member 模組不用等通知發完才回應
- 如果簡訊或 Email 發送失敗，可以重試
- Notification 模組掛掉也不影響會員註冊

我們用 Azure Service Bus 來做這件事，一個月大概 $10，很便宜。

### 2. 資料同步

有些資料需要同步到其他系統（報表資料庫、搜尋引擎、資料倉儲）。

這些也不用即時，可以用 message queue 慢慢處理。

## 實作細節

### In-Process gRPC 設定

在 .NET 裡，要讓 gRPC 在同一個 process 內工作，要用特殊的設定：

```csharp
// Program.cs

// 註冊 gRPC services（各模組提供的服務）
builder.Services.AddGrpc();

// 註冊 gRPC clients（各模組要呼叫其他模組的 client）
builder.Services.AddGrpcClient<MemberService.MemberServiceClient>(o =>
{
    o.Address = new Uri("http://localhost");
}).ConfigurePrimaryHttpMessageHandler(() =>
{
    return new SocketsHttpHandler
    {
        ConnectCallback = async (context, cancellationToken) =>
        {
            // In-process connection
            var socket = new Socket(SocketType.Stream, ProtocolType.Tcp);
            await socket.ConnectAsync(context.DnsEndPoint, cancellationToken);
            return new NetworkStream(socket, ownsSocket: true);
        }
    };
});

// Map gRPC services
app.MapGrpcService<MemberGrpcService>();
app.MapGrpcService<PointsGrpcService>();
```

這樣設定之後，所有 gRPC call 都在本機內完成，不會真的走網路。

### 錯誤處理

gRPC 有自己的錯誤處理機制，用 `StatusCode` 來表示不同的錯誤：

```csharp
public override async Task<GetMemberResponse> GetMember(
    GetMemberRequest request, ServerCallContext context)
{
    if (string.IsNullOrEmpty(request.MemberId))
    {
        throw new RpcException(new Status(
            StatusCode.InvalidArgument, "會員 ID 不能是空的"));
    }
    
    var member = await _memberRepository.GetByIdAsync(request.MemberId);
    
    if (member == null)
    {
        throw new RpcException(new Status(
            StatusCode.NotFound, $"找不到會員：{request.MemberId}"));
    }
    
    return new GetMemberResponse
    {
        MemberId = member.Id,
        Name = member.Name,
        Level = member.Level,
        IsActive = member.IsActive
    };
}
```

呼叫端可以捕捉 `RpcException` 來處理錯誤：

```csharp
try
{
    var response = await _memberClient.GetMemberAsync(
        new GetMemberRequest { MemberId = memberId });
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
{
    // 會員不存在
    return NotFound();
}
catch (RpcException ex)
{
    // 其他錯誤
    _logger.LogError(ex, "呼叫 Member 服務失敗");
    return StatusCode(500);
}
```

## 反思

選擇技術方案的時候，不能只看「主流」或「潮流」。

Event-driven、message queue 很流行，但不代表所有情境都適用。

我們的原則是：
- **需要即時回應的操作**：用 gRPC（同步）
- **可以延後處理的操作**：用 message queue（非同步）

這樣既保證了使用者體驗，也保留了系統的彈性。

更重要的是，團隊熟悉這套架構，維護起來不會太複雜。

接下來還有六個月的時間，會繼續優化這套通訊機制。特別是當模組越來越多，要確保效能不會因為 call chain 變長而下降。

下週會寫分散式快取的設計，特別是怎麼用 Redis 來減少資料庫查詢。
