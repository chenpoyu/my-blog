---
layout: post
title: "那些壓測前沒想到的問題"
date: 2025-04-11 14:20:00 +0800
categories: [效能優化]
tags: [壓力測試, gRPC, Redis, 連線池]
---

## 背景：APP 開啟就要打 12 個 API

在一次壓測的過程中，我發現一件事：**APP 每次開啟首頁，都會同時打 12 個 API。**

這個 APP 不是我們團隊開發的，是客戶找的另一家廠商做的。

他們當初設計的 API 規格就是這樣：

```
會員資料                 /api/member/profile
會員等級                 /api/member/level
點數餘額                 /api/points/balance
點數歷史（最近 10 筆）    /api/points/history
可用優惠券               /api/coupons/available
優惠券數量               /api/coupons/count
進行中的活動             /api/activities/ongoing
最新文章（5 篇）         /api/articles/latest
未讀通知                 /api/notifications/unread
通知數量                 /api/notifications/count
問卷待填                 /api/surveys/pending
系統公告                 /api/announcements
```

「為什麼不合併成一個 API？」我當時問過客戶。

「APP 那邊是另一家廠商做的，他們說這樣比較好維護。」

「現在規格都定了，要改的話要重新談合約。」

好吧，那就這樣。

**我沒有選擇，只能接受這個設計。**

我看著這個清單，心裡有點慌。

我們之前測試都是一支一支 API 測，從來沒想過會有人「同時」打 12 支。

## 第一個問題：gRPC 連線池爆了

下午我們試著用 Postman 同時打這 12 個 API。

然後就看到 Application Insights 噴了一堆錯誤：

```
Grpc.Core.RpcException: Status(StatusCode="Unavailable", Detail="failed to connect to all addresses")
```

什麼？gRPC 連不上？

我去看程式碼，發現我們的 gRPC client 設定太簡單了：

```csharp
services.AddGrpcClient<MemberService.MemberServiceClient>(options =>
{
    options.Address = new Uri("https://localhost:5001");
});
```

這樣的設定，預設連線池太小。當 12 個 API 同時發出去，每個都要呼叫 Member 模組查會員資料，連線池瞬間被用光。

改成這樣：

```csharp
services.AddGrpcClient<MemberService.MemberServiceClient>(options =>
{
    options.Address = new Uri("https://localhost:5001");
})
.ConfigurePrimaryHttpMessageHandler(() =>
{
    var handler = new SocketsHttpHandler
    {
        PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
        KeepAlivePingDelay = TimeSpan.FromSeconds(60),
        KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
        EnableMultipleHttp2Connections = true,  // 關鍵：允許多個 HTTP/2 連線
        MaxConnectionsPerServer = 10            // 每個 server 最多 10 個連線
    };
    return handler;
});
```

重點是 `EnableMultipleHttp2Connections = true`。

HTTP/2 預設會複用同一個 TCP 連線，但在高併發的情況下，一個連線可能不夠用。允許多個連線可以提升併發處理能力。

## 第二個問題：Redis timeout

改完 gRPC 之後，又遇到新問題：

```
StackExchange.Redis.RedisTimeoutException: Timeout performing GET member:M12345
```

Redis 也 timeout 了。

我去看 Redis 的設定：

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = configuration["Redis:ConnectionString"];
});
```

也是太簡單了，什麼參數都沒設。

改成：

```csharp
var redisConnection = ConnectionMultiplexer.Connect(new ConfigurationOptions
{
    EndPoints = { configuration["Redis:ConnectionString"] },
    ConnectTimeout = 5000,        // 連線 timeout: 5 秒
    SyncTimeout = 3000,           // 同步操作 timeout: 3 秒
    AsyncTimeout = 3000,          // 非同步操作 timeout: 3 秒
    ConnectRetry = 3,             // 連線失敗重試 3 次
    KeepAlive = 60,               // 保持連線活著
    AbortOnConnectFail = false,   // 連線失敗不要整個掛掉
});

services.AddSingleton<IConnectionMultiplexer>(redisConnection);
```

重點：
- `ConnectTimeout` 和 `SyncTimeout` 要分開設定
- `AbortOnConnectFail = false` 很重要，不然 Redis 掛掉會拖垮整個系統
- `KeepAlive` 可以避免連線被 idle timeout

## 第三個問題：資料庫連線池

12 個 API 同時打，每個都要查資料庫。

Connection string 本來是：

```
Server=tcp:myserver.database.windows.net;Database=mydb;User ID=admin;Password=xxx;
```

改成：

```
Server=tcp:myserver.database.windows.net;Database=mydb;User ID=admin;Password=xxx;
Max Pool Size=100;Min Pool Size=10;Connect Timeout=30;
```

- `Max Pool Size=100`：連線池最多 100 個連線
- `Min Pool Size=10`：最少保持 10 個連線（避免冷啟動）
- `Connect Timeout=30`：連線 timeout 30 秒

不要小看連線池的設定。預設的 Max Pool Size 通常很小（可能只有 10-20），高併發時很容易用光。

## 測試：同時 50 個使用者開 App

改完之後，我們用 k6 寫了一個簡單的測試：

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
    vus: 50,           // 50 個虛擬使用者
    duration: '1m',    // 跑 1 分鐘
};

export default function () {
    // 模擬開啟 App：同時打 12 個 API
    let responses = http.batch([
        ['GET', 'https://api.example.com/api/member/profile'],
        ['GET', 'https://api.example.com/api/member/level'],
        ['GET', 'https://api.example.com/api/points/balance'],
        ['GET', 'https://api.example.com/api/points/history'],
        ['GET', 'https://api.example.com/api/coupons/available'],
        ['GET', 'https://api.example.com/api/coupons/count'],
        ['GET', 'https://api.example.com/api/activities/ongoing'],
        ['GET', 'https://api.example.com/api/articles/latest'],
        ['GET', 'https://api.example.com/api/notifications/unread'],
        ['GET', 'https://api.example.com/api/notifications/count'],
        ['GET', 'https://api.example.com/api/surveys/pending'],
        ['GET', 'https://api.example.com/api/announcements'],
    ]);

    // 檢查每個回應
    responses.forEach(response => {
        check(response, {
            'status is 200': (r) => r.status === 200,
            'response time < 500ms': (r) => r.timings.duration < 500,
        });
    });

    sleep(1);
}
```

結果：

| 指標 | 數值 |
|-----|-----|
| 平均回應時間 | 180ms |
| P95 回應時間 | 420ms |
| P99 回應時間 | 680ms |
| 錯誤率 | 0.2% |

好多了。

雖然 P99 還是有點高（680ms），但比改之前的「直接爆炸」好太多。

## 開發時想不到的事

這次的經驗讓我發現：

**開發時測試的情境，跟實際使用差很多。**

我們在開發時，通常是：
- 一次測一支 API
- 用 Postman 手動測
- 沒什麼併發

但實際上：
- 前端會「同時」打很多 API
- 使用者不會乖乖一個一個來
- 高併發時，連線池、timeout、retry 這些設定都會變得很重要

下次開發新功能，我應該先問前端：「你會一次打幾支 API？」

不然等到壓測才發現，就太晚了。

## 下一步：真正的壓力測試

這次只是「50 個使用者同時開 App」的簡單測試。

但客戶的要求是：「2000 個同時在線」。

而且最可怕的不是開 App，是**搶優惠券**。

那個流程要打 27 支 API。

光想就頭皮發麻。

下週開始，要來做真正的壓力測試了。
