---
layout: post
title: "終於撐住 2000 併發了"
date: 2025-06-20 15:10:00 +0800
categories: [效能優化]
tags: [壓力測試, 效能調校, 成功案例]
---

## 第三輪測試

上週測完 2000 併發之後，客戶說：「可以再測一次嗎？我想確認結果是穩定的。」

好吧，那就再測一次。

這次我們加碼：**2000 併發，跑 30 分鐘**（之前只跑 10 分鐘）。

而且要測兩個情境：
1. **一般使用**：開 App、瀏覽活動、查看優惠券
2. **搶券尖峰**：500 人同時搶券

## 一般使用情境

測試腳本：

```javascript
export let options = {
    stages: [
        { duration: '2m', target: 2000 },   // 2 分鐘爬升到 2000
        { duration: '30m', target: 2000 },  // 維持 30 分鐘
        { duration: '2m', target: 0 },
    ],
};

export default function () {
    // 模擬一般使用者行為
    
    // 開啟 App（12 個 API）
    http.batch([
        ['GET', `${BASE_URL}/api/member/profile`],
        ['GET', `${BASE_URL}/api/member/level`],
        ['GET', `${BASE_URL}/api/points/balance`],
        ['GET', `${BASE_URL}/api/points/history`],
        ['GET', `${BASE_URL}/api/coupons/available`],
        ['GET', `${BASE_URL}/api/coupons/count`],
        ['GET', `${BASE_URL}/api/activities/ongoing`],
        ['GET', `${BASE_URL}/api/articles/latest`],
        ['GET', `${BASE_URL}/api/notifications/unread`],
        ['GET', `${BASE_URL}/api/notifications/count`],
        ['GET', `${BASE_URL}/api/surveys/pending`],
        ['GET', `${BASE_URL}/api/announcements`],
    ]);
    
    sleep(Math.random() * 5 + 3);  // 停留 3-8 秒
    
    // 點進活動頁（4 個 API）
    let activityId = Math.floor(Math.random() * 10) + 1;
    http.get(`${BASE_URL}/api/activities/${activityId}`);
    http.get(`${BASE_URL}/api/coupons/applicable?activityId=${activityId}`);
    http.get(`${BASE_URL}/api/articles?activityId=${activityId}`);
    http.get(`${BASE_URL}/api/comments?activityId=${activityId}`);
    
    sleep(Math.random() * 5 + 2);
    
    // 查看優惠券（2 個 API）
    http.get(`${BASE_URL}/api/coupons/mine`);
    http.get(`${BASE_URL}/api/coupons/history`);
    
    sleep(Math.random() * 3 + 2);
}
```

結果：

| 指標 | 數值 |
|-----|-----|
| 總請求數 | 8,650,000+ |
| 平均 RPS | 4,800 |
| 平均回應時間 | 380ms |
| P50 回應時間 | **336ms** |
| P90 回應時間 | **570ms** |
| P95 回應時間 | 720ms |
| P99 回應時間 | **680ms** |
| 錯誤率 | 0.25% |

**P50: 336ms, P90: 570ms, P99: 680ms**。

而且這是跑 30 分鐘的結果，證明系統的穩定性。

## 搶券情境

第二個測試是搶券，我們用 spike test 模擬：

```javascript
export let options = {
    stages: [
        { duration: '10s', target: 0 },
        { duration: '10s', target: 500 },   // 突然衝到 500 人
        { duration: '3m', target: 500 },    // 維持 3 分鐘
        { duration: '10s', target: 0 },
    ],
};

export default function () {
    // 完整的搶券流程（27 個 API）
    
    // 1. 進入活動頁（5 個 API）
    // 2. 點擊領券（1 個 API + queue）
    // 3. 等待領券成功（WebSocket 推播）
    // 4. 更新畫面（5 個 API）
    // 5. 結帳流程（16 個 API，包含金流）
    
    // ... (完整腳本太長，省略)
}
```

結果：

| 指標 | 數值 |
|-----|-----|
| 平均回應時間 | 950ms |
| P50 回應時間 | 680ms |
| P90 回應時間 | 1800ms |
| P99 回應時間 | 2800ms |
| 成功領券率 | 98.5% |
| 錯誤率 | 1.5% |

搶券的回應時間比一般使用慢（因為流程複雜），但還是在可接受範圍。

而且成功率 98.5%，很不錯了。

## 資源使用情況

測試的同時，我們監控了 Azure 資源的使用狀況：

### App Service (P1V3 × 2 instances)

| 指標 | 數值 |
|-----|-----|
| CPU 使用率 | 55-65% |
| 記憶體使用率 | 70-75% |
| HTTP Requests/sec | 4800 |
| HTTP Response Time | 380ms (avg) |

**CPU 和記憶體都還有 buffer**，不會爆掉。

而且因為是 2 個 instance，如果其中一個掛了，另一個還可以撐著。

### SQL Database (P2 - 250 DTU)

| 指標 | 數值 |
|-----|-----|
| DTU 使用率 | 60-70% |
| CPU 使用率 | 45-55% |
| Data IO 使用率 | 40-50% |
| Log IO 使用率 | 30-40% |
| Connection count | 80-120 |

**DTU 使用率 60-70%，很健康**。

而且我們已經優化過 SQL，加了該加的 index，沒有 slow query。

### Redis (Standard C2 - 2.5 GB)

| 指標 | 數值 |
|-----|-----|
| CPU 使用率 | 35-45% |
| 記憶體使用率 | 55-60% |
| Connected Clients | 150-200 |
| Commands/sec | 8000-12000 |

**Redis 很輕鬆，完全沒壓力**。

Standard C2 對我們來說綽綽有餘。

### Azure Service Bus (Standard)

| 指標 | 數值 |
|-----|-----|
| Incoming Messages/sec | 150 (搶券時) |
| Outgoing Messages/sec | 150 |
| Queue Length | 0-50 |

**Queue 處理速度很快**，沒有積壓。

## 成本總結

最終的正式環境成本：

| 服務 | 規格 | 成本/月 (USD) |
|-----|------|--------------|
| App Service Plan | P1V3 × 2 | $420 |
| SQL Database | P2 (250 DTU) | $930 |
| Redis Cache | Standard C2 | $146 |
| Front Door | Standard | $35 + 流量費 |
| Service Bus | Standard | $10 |
| Application Insights | Pay-as-you-go | ~$50 |
| Load Testing (開發期間) | 測試完就關掉 | $0 |
| **總計** | | **~$1,600/月** |

一年約 $19,200，折合台幣約 60 萬。

客戶說：「這個成本可以接受，比我們預期的低。」

## 為什麼是 P1V3 而不是 P2V3？

有人問：「為什麼 App Service 選 P1V3，不選效能更好的 P2V3？」

| 規格 | vCPU | RAM | 價格/月 (單 instance) |
|------|------|-----|---------------------|
| P1V3 | 2 | 8 GB | $210 |
| P2V3 | 4 | 16 GB | $420 |
| P3V3 | 8 | 32 GB | $840 |

選 P1V3 × 2 的原因：

1. **2000 併發用不到 4 vCPU**：測試顯示 2 vCPU 的 CPU 使用率只有 55-65%
2. **High Availability**：2 個 instance 可以做到 HA，一個掛了另一個還在
3. **成本考量**：P1V3 × 2 = $420，P2V3 × 1 = $420（價格一樣，但 2 個 instance 更穩）

如果未來流量成長，可以考慮：
- 先加到 P1V3 × 3 或 × 4
- 或是升級到 P2V3 × 2

反正雲端隨時可以調整。

## 為什麼是 P2 而不是 S3？

SQL Database 為什麼選 Premium P2 而不是 Standard S3？

| 規格 | DTU | IOPS | 價格/月 |
|------|-----|------|---------|
| S3 | 100 | ~400 | $150 |
| P2 | 250 | ~1000 | $930 |

看起來 S3 便宜很多（$150 vs $930），但：

1. **IOPS 差很多**：S3 只有 400 IOPS，P2 有 1000 IOPS
2. **搶券時 S3 撐不住**：我們測過，S3 在搶券時 DTU 會飆到 95%
3. **Log write speed**：P2 的 log write 速度是 S3 的 3 倍（對 transaction 很重要）

客戶對這個成本有疑慮，但我解釋：

「資料庫是整個系統的核心，如果資料庫慢，什麼都慢。而且 P2 還有更好的 SLA 和備份機制。」

「這是不能省的錢。」

客戶最後同意了。

## 下一步

壓測算是完成了，接下來要處理：

1. **安全性**：Key Vault、API 認證、DR plan
2. **監控**：設定 alert、dashboard
3. **上線準備**：checklist、rollback plan、on-call 輪值

預計下個月就要上線了。

有點緊張，但也有點期待。

這個專案從 1 月開始，到現在快半年了。

終於要看到成果了。
