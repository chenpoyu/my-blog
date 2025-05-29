---
layout: post
title: "搶券活動的 27 支 API 地獄"
date: 2025-05-30 10:00:00 +0800
categories: [效能優化]
tags: [壓力測試, API 優化, Queue, 搶購]
---

## 最可怕的流程

上週優化完「開 App」的流程，這週要面對更可怕的：**搶券活動**。

我讓前端同事帶我走一遍完整的搶券流程，並且用 Chrome DevTools 記錄所有的 API 呼叫。

結果：**從進入活動頁到領券成功，總共打了 27 支 API**。

我傻眼了。

「為什麼會這麼多？」

前端同事說：「因為要檢查很多東西啊⋯⋯」

她打開流程圖給我看：

```
1. 進入活動頁
   └─ GET /api/activities/{id}          # 取得活動資訊
   └─ GET /api/member/profile           # 確認會員身份
   └─ GET /api/member/level             # 檢查會員等級（影響能領幾張券）
   └─ GET /api/points/balance           # 顯示目前點數
   └─ GET /api/coupons/available        # 取得目前已有的優惠券

2. 點擊「領取優惠券」按鈕
   └─ POST /api/coupons/claim           # 領券（這是重點）
   
3. 領券成功後，畫面要更新
   └─ GET /api/member/profile           # 重新取得會員資料
   └─ GET /api/points/balance           # 更新點數（領券可能扣點）
   └─ GET /api/coupons/available        # 更新優惠券清單
   └─ GET /api/coupons/count            # 更新優惠券數量
   └─ GET /api/activities/{id}          # 更新活動資訊（剩餘張數）

4. 如果要結帳使用優惠券
   └─ GET /api/cart                     # 取得購物車
   └─ GET /api/coupons/applicable       # 查詢可用的優惠券
   └─ POST /api/cart/apply-coupon       # 套用優惠券
   └─ GET /api/cart                     # 重新取得購物車（顯示折扣）
   └─ POST /api/orders/preview          # 預覽訂單
   └─ GET /api/member/addresses         # 取得配送地址
   └─ GET /api/payments/methods         # 取得付款方式
   └─ POST /api/orders                  # 建立訂單
   
5. 訂單建立後
   └─ GET /api/orders/{id}              # 取得訂單詳情
   └─ POST /api/payments/process        # 處理付款（外部金流 API）
   └─ GET /api/payments/{id}/status     # 查詢付款狀態
   └─ POST /api/orders/{id}/confirm     # 確認訂單
   └─ GET /api/points/balance           # 更新點數（訂單可能賺點）
   └─ GET /api/coupons/available        # 更新優惠券（已使用）
   └─ POST /api/notifications/send      # 發送通知
```

「天啊⋯⋯」

這還不包括中間可能的錯誤重試、polling、timeout。

## 測試：1000 人同時搶券

客戶說搶券活動預計會有 500-1000 人同時參加。

我們先測 500 人：

```javascript
export let options = {
    stages: [
        { duration: '10s', target: 500 },  // 10 秒內衝到 500 人
        { duration: '3m', target: 500 },   // 維持 3 分鐘
        { duration: '10s', target: 0 },
    ],
};

export default function () {
    // 完整的搶券流程
    
    // 1. 進入活動頁
    http.get(`${BASE_URL}/api/activities/${ACTIVITY_ID}`);
    http.get(`${BASE_URL}/api/member/profile`);
    http.get(`${BASE_URL}/api/member/level`);
    http.get(`${BASE_URL}/api/points/balance`);
    http.get(`${BASE_URL}/api/coupons/available`);
    
    sleep(1);  // 使用者看一下頁面
    
    // 2. 點擊領券
    let claimRes = http.post(`${BASE_URL}/api/coupons/claim`, 
        JSON.stringify({ activityId: ACTIVITY_ID }));
    
    if (claimRes.status === 200) {
        // 3. 領券成功，更新畫面
        http.get(`${BASE_URL}/api/member/profile`);
        http.get(`${BASE_URL}/api/points/balance`);
        http.get(`${BASE_URL}/api/coupons/available`);
        http.get(`${BASE_URL}/api/coupons/count`);
        http.get(`${BASE_URL}/api/activities/${ACTIVITY_ID}`);
        
        sleep(2);  // 使用者決定要不要結帳
        
        // 4. 結帳流程（50% 的人會繼續）
        if (Math.random() < 0.5) {
            http.get(`${BASE_URL}/api/cart`);
            http.get(`${BASE_URL}/api/coupons/applicable`);
            // ... 後面還有一堆
        }
    }
    
    sleep(5);
}
```

按下 Run。

## 災難

測試跑了 30 秒之後，**系統直接掛了**。

Application Insights 顯示：

```
SqlException: Timeout expired. The timeout period elapsed prior to completion.

Grpc.Core.RpcException: Status(StatusCode="ResourceExhausted")

RedisConnectionException: No connection is available to service this operation
```

資料庫、gRPC、Redis 全部爆炸。

錯誤率：**85%**。

幹。

## 問題 1：資料庫鎖死

首先是資料庫的問題。

領券的時候，我們要做這些事：
1. 檢查優惠券是否還有庫存
2. 檢查使用者是否已經領過
3. 建立一筆 `CouponUsage` 記錄
4. 更新優惠券的剩餘數量
5. 記錄領券的 log

這些操作都在一個 transaction 裡面。

而且為了避免超領，我們用了 `SELECT FOR UPDATE` 來鎖定優惠券的那一列：

```sql
BEGIN TRANSACTION;

SELECT * FROM Coupons WHERE Id = @couponId FOR UPDATE;
-- 檢查庫存
-- 檢查是否已領過
INSERT INTO CouponUsages (...);
UPDATE Coupons SET Quantity = Quantity - 1 WHERE Id = @couponId;

COMMIT;
```

這樣的做法，在正常流量下沒問題。

但在搶券情境，會有大量的併發請求。

我們擔心 **500 人同時搶同一張券，會有 500 個 transaction 在排隊等這個 row lock**。

所以要測試看看 Azure SQL Database P2 能不能撐住。

## 解法：優化 SQL 查詢

我們先優化了 SQL 查詢：

1. **加上適當的 index**：在 CouponId、MemberId 上建 index
2. **減少 transaction 時間**：把不必要的查詢移到 transaction 外面
3. **使用 Optimistic Concurrency**：用 RowVersion 避免長時間 lock

改完之後，單次領券的 transaction 時間從 80ms 降到 30ms。

這樣可以提升 2-3 倍的吞吐量。

## 問題 2：外部金流 API 很慢

搶券流程的最後是結帳，要呼叫外部的金流 API。

但金流 API 的回應時間很不穩定：
- 正常：200-500ms
- 慢的時候：3-5 秒
- 更慢的時候：timeout (30 秒)

這會拖垮整個流程。

而且我們不能控制外部 API 的效能。

## 解法：設定合理的 timeout

我們設定了比較短的 timeout，並且加上 retry：

```csharp
var httpClient = new HttpClient
{
    Timeout = TimeSpan.FromSeconds(5)  // 5 秒 timeout
};

var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<TaskCanceledException>()
    .WaitAndRetryAsync(3, retryAttempt => 
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));  // 2s, 4s, 8s

await retryPolicy.ExecuteAsync(async () =>
{
    var response = await httpClient.PostAsync(paymentApiUrl, content);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync();
});
```

而且如果金流 API timeout，我們會先建立訂單（狀態設為「待付款」），然後用背景 job 定期檢查付款狀態。

這樣至少不會讓使用者一直等。

## 重新測試

改完之後，重新測 500 人搶券：

| 指標 | 改之前 | 改之後 |
|-----|-------|--------|
| 錯誤率 | 12% | 2% |
| 平均回應時間 | 2.8s | 0.9s |
| P99 回應時間 | 8.5s | 2.1s |
| 成功領券率 | 88% | 98% |

好多了！

但還是有 2% 的失敗率。主要是：
- 優惠券真的搶完了（正常現象）
- 金流 API timeout（已經改成非同步處理，不影響使用者）

**原本擔心的資料庫 lock 問題，其實沒有想像中嚴重。**

SQL 優化做好，P2 的規格足夠應付這個流量。

## 下一步

搶券流程算是穩定了，但我們還沒測到 2000 併發。

下週要繼續增加流量，看系統的極限在哪裡。

**這次經驗讓我學到：不要過度設計。**

本來以為要改成 async queue，結果只要優化 SQL 就夠了。

先測試，找到真正的瓶頸，再決定要不要改架構。
