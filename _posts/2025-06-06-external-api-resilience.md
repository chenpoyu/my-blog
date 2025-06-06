---
layout: post
title: "外部 API 拖垮整個系統"
date: 2025-06-06 14:40:00 +0800
categories: [架構設計]
tags: [Circuit Breaker, Resilience, Timeout, 外部整合]
---

## 週五晚上又掛了

上週五晚上 8 點，正準備下班，手機又響了。

監控 alert：「API error rate > 10%」。

我打開 Application Insights，發現大量的 timeout：

```
System.Threading.Tasks.TaskCanceledException: 
A task was canceled.
at PaymentService.ProcessPaymentAsync(...)
```

而且不只付款 API，連帶的結帳、訂單查詢、優惠券使用⋯⋯全部都掛了。

「又是金流 API 的問題。」

## 金流 API 的不穩定

我們用的金流服務叫「XXPay」（為了避免爭議，隱藏真實名稱）。

這個 API 的特性是：
- **大部分時候很快**：200-500ms
- **偶爾會很慢**：3-10 秒
- **有時候會完全無回應**：timeout 30 秒

而且最糟的是：**我們無法控制它**。

因為它是第三方服務，我們只能呼叫它的 API，無法改善它的效能。

但問題是：**它的問題會影響我們整個系統**。

## 連鎖反應

當金流 API 變慢時，會發生這種情況：

```
使用者 A 結帳 → 呼叫金流 API → 等待 30 秒 → timeout
使用者 B 結帳 → 呼叫金流 API → 等待 30 秒 → timeout
使用者 C 結帳 → 呼叫金流 API → 等待 30 秒 → timeout
...
```

每個請求都要等 30 秒才 timeout，而在這 30 秒內：
- **占用一個 HTTP connection**
- **占用一個 thread**（或 async task）
- **占用資料庫連線**（transaction 還沒結束）

如果同時有 50 個人在結帳，就會有 50 個 thread 卡在那裡等。

App Service 的 thread pool 被用光，**連其他正常的 API 都會受影響**。

這就是典型的「雪崩效應」。

## 解法 1：設定更短的 timeout

最直接的方式是：**不要等 30 秒，設定 5 秒就 timeout**。

```csharp
var httpClient = new HttpClient
{
    Timeout = TimeSpan.FromSeconds(5)
};
```

這樣至少不會讓 thread 卡太久。

但這也有問題：**金流 API 真的需要 6 秒怎麼辦？**

使用者明明付款成功了，我們卻因為 timeout 而判定失敗。

## 解法 2：改成非同步處理

更好的做法是：**不要同步等待金流 API 的結果**。

改成這樣：

```
1. 使用者送出付款請求
   ↓
2. 我們先建立訂單（狀態：pending）
   ↓
3. 非同步呼叫金流 API
   ↓
4. 立即回應使用者「處理中，請稍候」
   ↓
5. 背景 job 定期查詢金流 API 的結果
   ↓
6. 查到結果後，更新訂單狀態，並通知使用者
```

這樣的好處：
- **不會占用 thread 等待**
- **使用者不會看到 timeout**
- **即使金流 API 很慢，也不影響系統**

但壞處是：
- **使用者要等待**（不是即時知道結果）
- **前端要實作 polling 或 WebSocket**
- **訂單狀態變複雜**（pending → processing → success/failed）

## 解法 3：Circuit Breaker

但還有一個問題：**如果金流 API 完全掛掉怎麼辦？**

例如它連續 100 次都 timeout，我們還要繼續呼叫它嗎？

這時候就需要 **Circuit Breaker（斷路器）** 模式。

概念很簡單：

```
1. Closed（正常狀態）
   - 正常呼叫 API
   - 如果失敗次數超過閾值（例如 10 次），進入 Open

2. Open（斷路狀態）
   - 直接回傳錯誤，不呼叫 API
   - 經過一段時間（例如 30 秒）後，進入 Half-Open

3. Half-Open（半開狀態）
   - 嘗試呼叫 API
   - 如果成功，回到 Closed
   - 如果失敗，回到 Open
```

這樣可以避免在 API 掛掉的時候，還一直浪費資源去呼叫它。

## 實作：用 Polly

.NET 有一個很好用的 library 叫 **Polly**，可以實作各種 resilience patterns。

安裝：

```bash
dotnet add package Polly
```

設定 Circuit Breaker：

```csharp
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<TaskCanceledException>()
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 10,  // 失敗 10 次就斷路
        durationOfBreak: TimeSpan.FromSeconds(30)  // 斷路 30 秒
    );

var timeoutPolicy = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(5));  // 5 秒 timeout

var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => 
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

// 組合三個 policy
var resilientPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy, timeoutPolicy);

// 使用
try
{
    var result = await resilientPolicy.ExecuteAsync(async () =>
    {
        var response = await httpClient.PostAsync(paymentApiUrl, content);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    });
}
catch (BrokenCircuitException)
{
    // 斷路了，直接回傳錯誤給使用者
    return new PaymentResult 
    { 
        Success = false, 
        Message = "Payment service is temporarily unavailable" 
    };
}
```

這樣就有了：
- **5 秒 timeout**
- **失敗自動 retry 3 次**
- **連續失敗 10 次就斷路 30 秒**

## 測試：模擬金流 API 掛掉

我們寫了一個測試，模擬金流 API 完全無回應：

```javascript
// k6 測試腳本
export default function () {
    let res = http.post('https://api.example.com/api/orders/checkout', 
        JSON.stringify({ amount: 1000, paymentMethod: 'credit_card' }));
    
    check(res, {
        'status is not 500': (r) => r.status !== 500,  // 不要整個掛掉
        'response time < 10s': (r) => r.timings.duration < 10000,  // 不要卡很久
    });
}
```

結果：

| 指標 | 沒有 Circuit Breaker | 有 Circuit Breaker |
|-----|---------------------|-------------------|
| 平均回應時間 | 28s | 0.8s |
| P99 回應時間 | 30s | 5.2s |
| 錯誤率 | 100% | 100% (但很快就失敗) |
| Thread pool 耗盡 | 是 | 否 |

雖然錯誤率都是 100%（因為金流 API 確實掛了），但有 Circuit Breaker 的版本：
- **不會卡 thread**
- **快速失敗**（0.8 秒就回傳錯誤）
- **不會影響其他 API**

## 但使用者還是會失敗啊？

對，Circuit Breaker 不能讓失敗的 API 變成功，它只是**讓失敗來得更快、更優雅**。

真正的解決方案是：

1. **改成非同步處理**（不要同步等金流 API）
2. **提供降級方案**（例如改用其他金流服務、或記錄下來稍後處理）
3. **通知使用者**（「付款系統暫時無法使用，請稍後再試」）

Circuit Breaker 只是其中一環，不是萬靈丹。

## 另一個問題：物流 API

除了金流，我們還整合了物流 API（查詢貨態、預估送達時間）。

這個 API 也很不穩定。

但好消息是：**物流 API 不是關鍵路徑**。

使用者下單時不一定要立刻知道送達時間，我們可以之後再查、再推播給他。

所以物流 API 的處理比較簡單：
- **全部改成非同步**
- **用背景 job 定期查詢**
- **有結果再推播給使用者**

這樣就算物流 API 掛了，也不會影響結帳流程。

## 心得

跟外部 API 整合，最大的挑戰是：**你無法控制它的品質**。

它可能：
- 很慢
- 掛掉
- 回傳錯誤資料
- 突然改 API 格式（然後沒通知你）

所以一定要做好 resilience：
- **Timeout**：不要無限等待
- **Retry**：自動重試（但要小心 retry storm）
- **Circuit Breaker**：快速失敗，避免拖垮系統
- **Fallback**：提供降級方案
- **監控**：第一時間知道外部 API 出問題

這些都是血淚經驗。

下週要測試 2000 併發了，希望不會再有新的問題⋯⋯

（但我知道一定會有）
