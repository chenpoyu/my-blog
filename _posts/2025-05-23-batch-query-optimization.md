---
layout: post
title: "開 App 就打 12 支 API 的災難"
date: 2025-05-23 16:20:00 +0800
categories: [效能優化]
tags: [API 優化, gRPC, 批次查詢, 快取]
---

## 500 併發就開始出錯了

這週我們把併發數從 100 提升到 500。

測試腳本還是一樣，模擬使用者開啟 App，同時打 12 支 API。

結果：

| 指標 | 100 併發 | 500 併發 |
|-----|---------|---------|
| 平均回應時間 | 85ms | 450ms |
| P95 回應時間 | 320ms | 1800ms |
| P99 回應時間 | 580ms | 3500ms |
| 錯誤率 | 0.03% | 2.5% |

**錯誤率從 0.03% 暴增到 2.5%**。

而且 P99 回應時間高達 3.5 秒，完全無法接受。

## 問題在哪裡？

我去看 Application Insights 的 trace，發現大量的 gRPC timeout：

```
Grpc.Core.RpcException: Status(StatusCode="DeadlineExceeded", Detail="Deadline Exceeded")
at MemberGrpcClient.GetMemberAsync(String memberId)
```

DeadlineExceeded = gRPC 呼叫超時。

但為什麼會超時？

我仔細看了一下呼叫流程：

```
使用者開啟 App
├─ API 1: /api/member/profile     → gRPC: GetMember
├─ API 2: /api/member/level       → gRPC: GetMember
├─ API 3: /api/points/balance     → gRPC: GetMember
├─ API 4: /api/points/history     → gRPC: GetMember
├─ API 5: /api/coupons/available  → gRPC: GetMember
├─ API 6: /api/coupons/count      → gRPC: GetMember
├─ API 7: /api/activities/ongoing → gRPC: GetMember
├─ API 8: /api/articles/latest    → gRPC: GetMember (驗證是否會員)
├─ API 9: /api/notifications/unread → gRPC: GetMember
├─ API 10: /api/notifications/count → gRPC: GetMember
├─ API 11: /api/surveys/pending    → gRPC: GetMember
└─ API 12: /api/announcements      → gRPC: GetMember
```

**幹，每個 API 都要查會員資料！**

一個使用者開 App，就對 Member 模組發了 **12 次 gRPC 請求**。

500 個併發使用者，就是 **6000 個 gRPC 請求**，而且是在幾秒鐘內同時打過來。

Member 模組的 gRPC server 根本撐不住。

## 為什麼每個 API 都要查會員？

因為每個 API 都需要驗證：
1. 這個 token 是否有效？
2. 對應的會員是誰？
3. 會員的狀態是什麼（active / suspended）？
4. 會員的等級是什麼（影響權限）？

這些資訊在每個 API 都需要，所以每個都要呼叫 `MemberGrpcClient.GetMemberAsync()`。

但這樣效率太低了。

## 解法 1：在 Gateway 層統一查詢

第一個想法是：**在 API Gateway (YARP) 層統一查詢會員資料**。

流程改成：

```
1. 使用者的 12 個 API 請求都帶 JWT token
2. Gateway 收到請求後，從 JWT 解析出 memberId
3. Gateway 統一呼叫一次 gRPC 取得會員資料
4. 把會員資料放到 HTTP Header 傳給後端 API
5. 後端 API 直接從 Header 讀取，不用再呼叫 gRPC
```

但這有個問題：**12 個 API 是同時發出去的**。

Gateway 收到 12 個請求，還是會發 12 次 gRPC。

除非⋯⋯

## 解法 2：加上快取

我們已經有 Redis 快取會員資料了，但只有「30 分鐘」的 TTL。

問題是：**快取的 key 是什麼？**

原本的 key 設計：

```csharp
var cacheKey = $"member:{memberId}";
```

每次查詢都要先查 Redis，如果沒有再查資料庫，然後寫入 Redis。

但如果 12 個 API 同時打過來，第一個 API 查 Redis 發現沒有資料，就去查資料庫。

**但其他 11 個 API 也會同時發現 Redis 沒有資料，所以也去查資料庫。**

這就是經典的 **Cache Stampede** 問題。

## 解法 3：批次查詢

我們改了一下 gRPC 的 API，新增一個批次查詢的方法：

```protobuf
service MemberService {
    rpc GetMember(GetMemberRequest) returns (GetMemberResponse);
    
    // 新增：批次查詢
    rpc GetMemberBatch(GetMemberBatchRequest) returns (GetMemberBatchResponse);
}

message GetMemberBatchRequest {
    repeated string member_ids = 1;
}

message GetMemberBatchResponse {
    repeated MemberDto members = 1;
}
```

然後在 Gateway 層改成：

```csharp
public class MemberCacheMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        var memberId = GetMemberIdFromToken(context);
        
        // 先查 Redis
        var cacheKey = $"member:{memberId}";
        var cached = await _cache.GetStringAsync(cacheKey);
        
        if (cached != null)
        {
            context.Items["Member"] = JsonSerializer.Deserialize<MemberDto>(cached);
        }
        else
        {
            // 沒有 cache，呼叫 gRPC（但用 lock 避免重複查詢）
            var member = await GetMemberWithLock(memberId);
            context.Items["Member"] = member;
            
            // 寫入 cache
            await _cache.SetStringAsync(cacheKey, 
                JsonSerializer.Serialize(member),
                new DistributedCacheEntryOptions 
                { 
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30) 
                });
        }
        
        await _next(context);
    }
    
    private async Task<MemberDto> GetMemberWithLock(string memberId)
    {
        var lockKey = $"lock:member:{memberId}";
        
        // 用 Redis lock 確保同一時間只有一個請求去查 gRPC
        if (await _cache.LockAsync(lockKey, TimeSpan.FromSeconds(5)))
        {
            try
            {
                return await _memberGrpcClient.GetMemberAsync(memberId);
            }
            finally
            {
                await _cache.UnlockAsync(lockKey);
            }
        }
        else
        {
            // 如果拿不到 lock，等一下再查 cache
            await Task.Delay(100);
            var cached = await _cache.GetStringAsync($"member:{memberId}");
            if (cached != null)
            {
                return JsonSerializer.Deserialize<MemberDto>(cached);
            }
            
            // 還是沒有，只好再試一次
            return await _memberGrpcClient.GetMemberAsync(memberId);
        }
    }
}
```

這個 middleware 做了幾件事：
1. **先查 Redis cache**
2. **如果沒有，用 distributed lock 確保只有一個請求去查 gRPC**
3. **其他請求等待，然後從 cache 讀取**

## 改完之後的效果

重新跑 500 併發測試：

| 指標 | 改之前 | 改之後 |
|-----|-------|--------|
| 平均回應時間 | 450ms | 120ms |
| P95 回應時間 | 1800ms | 380ms |
| P99 回應時間 | 3500ms | 650ms |
| 錯誤率 | 2.5% | 0.1% |
| gRPC 呼叫次數（每個使用者）| 12 次 | 0.5 次 (大部分走 cache) |

**好太多了！**

錯誤率從 2.5% 降到 0.1%，P99 從 3.5 秒降到 650ms。

而且看 Member 模組的監控，gRPC 的 QPS 從 6000 降到 250（減少了 96%）。

## 代價

這個優化也不是沒有代價：

1. **程式碼變複雜了**：要處理 lock、cache miss、timeout
2. **多了一層快取**：會員資料可能不是最新的（但我們可以接受 30 分鐘的延遲）
3. **依賴 Redis**：如果 Redis 掛了，所有請求都會打到 gRPC

但這些代價是值得的。

## 下一個挑戰

開 App 的問題解決了，但還有一個更可怕的：

**搶券流程**。

那個流程要打 27 支 API，而且很多是寫入操作（不能只靠快取解決）。

下週要來挑戰這個了。

想到就頭痛。
