---
layout: post
title: "SQL Database 規格怎麼選才對"
date: 2025-04-25 15:50:00 +0800
categories: [資料庫]
tags: [Azure SQL Database, 效能調校, DTU]
---

## DTU 又爆了

繼上週 Redis 升級之後，這週又輪到 SQL Database 出問題了。

週一早上測試團隊回報：「API 變很慢，每個都要等 5-10 秒。」

我打開 Application Insights，發現 SQL query 的執行時間異常地高：

```
SELECT * FROM Members WHERE Id = @id
執行時間：8000ms
```

一個簡單的 primary key 查詢要 8 秒？這不合理。

我去看 Azure Portal 的 SQL Database 監控，**DTU 使用率 100%**，而且已經持續 2 小時了。

又是規格不夠的問題。

## 什麼是 DTU？

Azure SQL Database 有兩種計價模型：
- **DTU (Database Transaction Unit)**：簡單，但彈性較低
- **vCore**：複雜，但可以細調

我們用的是 DTU 模型。

DTU 是 Azure 自己定義的「資料庫效能單位」，包含 CPU、記憶體、I/O 的綜合指標。

聽起來很抽象，對吧？我也這麼覺得。

但選擇 DTU 的原因很簡單：**不用想太多，照著 tier 選就對了**。

我們現在的規格是：**S2 (50 DTU)**

| Tier | 規格 | DTU | 儲存空間 | 價格/月 |
|------|------|-----|---------|---------|
| Basic | B | 5 | 2 GB | $5 |
| Standard | S0 | 10 | 250 GB | $15 |
| Standard | S1 | 20 | 250 GB | $30 |
| Standard | S2 | 50 | 250 GB | $75 |
| Standard | S3 | 100 | 250 GB | $150 |
| Premium | P1 | 125 | 500 GB | $465 |
| Premium | P2 | 250 | 500 GB | $930 |

S2 (50 DTU) 在開發初期還夠用，但現在測試流量增加，就不夠了。

問題是：**要升級到哪個規格？**

## 升到 S3 還是 P1？

我先看了 S3 (100 DTU) 和 P1 (125 DTU) 的差異：

| 項目 | S3 (Standard) | P1 (Premium) |
|-----|---------------|--------------|
| DTU | 100 | 125 |
| 價格/月 | $150 | $465 |
| IOPS | ~400 | ~1000 |
| Log write speed | ~4 MB/s | ~12 MB/s |
| 記憶體 | ~1 GB | ~3 GB |

P1 的效能明顯比 S3 好很多，但價格也貴 3 倍。

我猶豫了。

「到底該選哪個？」

## 先看瓶頸在哪裡

與其亂猜，不如先看**到底是什麼把 DTU 吃滿的**。

Azure Portal 的 Query Performance Insight 可以看到哪些 SQL 最耗資源：

```
TOP 5 queries by DTU consumption:

1. SELECT * FROM CouponUsages WHERE MemberId = @memberId AND Status = 'Active'
   執行次數: 15,234
   平均 DTU: 12.5
   總 DTU: 190,425 (佔 45%)

2. SELECT * FROM Points WHERE MemberId = @memberId ORDER BY CreatedAt DESC
   執行次數: 12,456
   平均 DTU: 8.3
   總 DTU: 103,385 (佔 24%)

3. SELECT * FROM Activities WHERE Status = 'Active' AND StartDate <= GETDATE()
   執行次數: 8,901
   平均 DTU: 5.2
   總 DTU: 46,285 (佔 11%)
```

前兩個 query 就佔了快 70% 的 DTU。

而且我注意到：**這兩個 query 都沒有用到 index**。

## 先優化 SQL 再升級

我檢查了一下 table schema：

```sql
-- CouponUsages table
CREATE TABLE CouponUsages (
    Id INT PRIMARY KEY,
    MemberId NVARCHAR(50),    -- 沒有 index！
    CouponId INT,
    Status NVARCHAR(20),      -- 沒有 index！
    UsedAt DATETIME,
    ...
)

-- Points table
CREATE TABLE Points (
    Id INT PRIMARY KEY,
    MemberId NVARCHAR(50),    -- 沒有 index！
    Amount INT,
    CreatedAt DATETIME,       -- 沒有 index！
    ...
)
```

難怪這麼慢。每次查詢都要 full table scan。

我加了幾個 index：

```sql
-- CouponUsages
CREATE INDEX IX_CouponUsages_MemberId_Status 
ON CouponUsages(MemberId, Status);

-- Points
CREATE INDEX IX_Points_MemberId_CreatedAt 
ON Points(MemberId, CreatedAt DESC);
```

改完之後，重新測試。

**DTU 使用率從 100% 降到 35%**。

原來不是規格不夠，是 SQL 沒優化好。

## 但還是要升級

雖然加了 index 之後，DTU 使用率降下來了，但我們還是決定升級。

原因：
1. **現在只是測試環境**，實際上線後流量會更大
2. **客戶要求 2000 個同時在線**，現在測試只有 50 個
3. **S2 (50 DTU) 的 margin 不夠**，稍微流量大一點就會爆

經過計算，我們預估：
- 測試環境 50 個併發 → DTU 35%（優化後）
- 正式環境 2000 個併發 → DTU 估計 1400（35% × 40 倍）

所以至少需要 **P2 (250 DTU)** 以上。

但等一下⋯⋯

250 DTU 應付 1400 的需求？這不合理啊。

## 重新計算

我仔細想了一下，這個計算方式有問題。

**不是所有使用者都會同時查詢資料庫**。

實際上：
- 50 個併發測試，資料庫的 QPS (Query Per Second) 大約是 500
- 換算成 2000 個併發，QPS 大約是 20,000
- 但不是所有 query 都一樣重

我們需要的是：**RPS (Requests Per Second) × 平均每個 request 的 DTU = 總 DTU 需求**

根據 Query Performance Insight 的數據：
- 平均每個 query 消耗 0.8 DTU
- 預估 QPS: 1000（保守估計，peak 時可能 2000）
- 總 DTU 需求: 1000 × 0.8 = 800 DTU

但要保留一些 buffer，所以選：

**P2 (250 DTU)**？不夠。

**P4 (500 DTU)**？比較保險。

但 P4 一個月要 $1,860，太貴了。

## 最後的選擇：P2 (250 DTU)

經過一番掙扎，我們還是選了 **P2 (250 DTU)**。

原因：
1. **現在是測試階段**，不需要直接上 P4
2. **可以隨時升級**，Azure SQL 升級只需要幾秒鐘
3. **再搭配優化**，應該可以撐住

升級完成後，我們繼續跑壓測：

| 指標 | S2 (50 DTU) | P2 (250 DTU) |
|-----|-------------|--------------|
| DTU 使用率 | 100% | 45% |
| 平均 query 時間 | 850ms | 35ms |
| P99 query 時間 | 8000ms | 120ms |

好很多了。

而且 P2 還有一些額外優點：
- **更高的 IOPS**：從 ~200 提升到 ~1000
- **更快的 log write**：有助於 transaction 效能
- **更多記憶體**：可以 cache 更多資料

## 下一步的優化

雖然升級到 P2 了，但還有一些可以優化的地方：

1. **Connection pooling**：確保連線池設定正確
2. **Query optimization**：檢查還有哪些 slow query
3. **Read replica**：考慮用 read-only replica 分流查詢
4. **Partitioning**：大 table 考慮做 partition

但這些都是後話了。

現在至少可以繼續測試，不用擔心資料庫隨時爆掉。

## 選規格的心得

這次經驗讓我學到：

1. **不要憑感覺選規格**：要看數據
2. **不要一開始就選最好的**：浪費錢
3. **不要只靠升級解決問題**：先優化 code 和 SQL
4. **保留 buffer**：不要讓資源使用率長期在 80% 以上

雲端的好處就是：可以隨時調整。

不用一次到位，慢慢測試、慢慢調整就好。

下週開始，要進入真正的壓力測試了。

2000 個併發，我來了。
