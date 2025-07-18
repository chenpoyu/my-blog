---
layout: post
title: "DR Plan：備份不是做心安的"
date: 2025-07-18 10:30:00 +0800
categories: [維運]
tags: [災難恢復, 備份, Azure, 高可用性]
---

## 「如果 Azure 掛了怎麼辦？」

上週的上線前會議,客戶問了一個問題:

「如果 Azure East Asia 整個 region 掛了,我們的服務要多久才能恢復？」

我愣了一下。

說實話,我們還沒認真想過這個問題。

「呃⋯⋯我們有做備份⋯⋯」我試著回答。

「備份在哪裡？」

「SQL Database 有 geo-redundant backup,Redis 也有 persistence⋯⋯」

「那要多久才能切換到備份？」

「⋯⋯我不確定。」

客戶看著我:「上線前,我需要一份 DR plan,包含 RTO 和 RPO。」

## RTO 和 RPO 是什麼

回到公司,我查了一下:

### RTO (Recovery Time Objective)

**系統掛掉後,多久要恢復正常？**

例如:
- RTO = 1 小時:掛掉後 1 小時內要恢復
- RTO = 4 小時:掛掉後 4 小時內要恢復

### RPO (Recovery Point Objective)

**最多可以損失多少資料？**

例如:
- RPO = 0:完全不能損失資料(需要即時備份)
- RPO = 1 小時:可以損失最近 1 小時的資料

這兩個指標會影響備份策略和成本:

| RTO | RPO | 成本 | 複雜度 |
|-----|-----|------|--------|
| 24 小時 | 24 小時 | 低 | 簡單 |
| 4 小時 | 1 小時 | 中 | 中 |
| 1 小時 | 15 分鐘 | 高 | 高 |
| < 5 分鐘 | 0 | 非常高 | 非常高 |

## 跟客戶確認需求

我回頭問客戶:「你們可以接受的 RTO 和 RPO 是多少？」

客戶想了想:「這是會員系統,不是金融交易。我覺得：」

- **RTO: 4 小時**（半天內恢復就可以）
- **RPO: 1 小時**（損失 1 小時內的資料可以接受）

「但如果能做到更好,當然更好。」

好,至少有個目標了。

## 災難情境分析

我列出了可能的災難情境:

### 情境 1: 單一 App Service instance 掛了

**影響**: 無(因為我們有 2 個 instance)

**處理**: Azure 會自動切換到另一個 instance

**RTO**: < 1 分鐘

**RPO**: 0

### 情境 2: SQL Database 掛了

**影響**: 所有寫入操作失敗,讀取也可能失敗

**處理**: 

1. 短暫故障(< 1 分鐘):自動重試
2. 長時間故障:切換到 geo-replica

**RTO**: 
- 自動 failover:< 5 分鐘
- 手動 failover:< 30 分鐘

**RPO**: 
- 如果用 geo-replication:< 5 分鐘
- 如果用 geo-redundant backup:1-24 小時

### 情境 3: Redis 掛了

**影響**: 效能變差(要查資料庫),但不影響核心功能

**處理**: 重啟 Redis,資料從資料庫重建

**RTO**: < 15 分鐘

**RPO**: N/A(cache 資料可以重建)

### 情境 4: 整個 East Asia region 掛了

**影響**: 所有服務無法使用

**處理**: 切換到另一個 region

**RTO**: 
- 如果有 hot standby:< 1 小時
- 如果沒有:4-8 小時

**RPO**: 取決於備份策略

## SQL Database 的備份策略

Azure SQL Database 預設就有備份:

### 自動備份

- **Full backup**: 每週一次
- **Differential backup**: 每 12-24 小時一次
- **Transaction log backup**: 每 5-10 分鐘一次

保留期限:
- **Basic/Standard**: 7-35 天
- **Premium**: 7-35 天(我們設定 35 天)

這些備份是 **geo-redundant**,會複製到 paired region(East Asia → Southeast Asia)。

### Geo-Replication

除了自動備份,我們還可以開啟 **Active Geo-Replication**:

- 在另一個 region(例如 Southeast Asia)建立一個 read-only replica
- 資料會即時複製過去(延遲 < 5 秒)
- 如果 primary region 掛了,可以手動 failover 到 replica

成本:
- **每個 replica 的成本 = primary database 的成本**
- 我們的 P2 是 $930/月,加一個 replica 就變 $1860/月

「這個成本有點高⋯⋯」我跟客戶說。

「那如果不做 geo-replication,RTO 會是多久？」客戶問。

「如果 region 掛了,要從 geo-redundant backup 還原,可能要 4-8 小時。」

「而且 RPO 會是最近一次 backup 的時間,可能 1-24 小時。」

客戶想了想:「先不做 geo-replication,但要定期演練備份還原。」

「如果未來真的有需要,再開。」

## Redis 的備份策略

Azure Cache for Redis 的備份選項:

### RDB Persistence

- 定期把記憶體資料存到磁碟
- 頻率:15 分鐘、30 分鐘、60 分鐘(可選)
- 只有 **Premium tier** 支援

我們用的是 Standard C2,**不支援 persistence**。

但 Redis 對我們來說只是 cache,掛了可以重建,所以沒關係。

## 演練:備份還原

光有備份還不夠,要實際演練過才知道能不能用。

### SQL Database 還原測試

我們建立了一個測試環境,從 backup 還原:

```bash
# 1. 列出可用的備份
az sql db list-restorable-dropped-databases \
  --resource-group my-rg \
  --server my-server

# 2. 還原到新的資料庫
az sql db restore \
  --resource-group my-rg \
  --server my-server \
  --name my-db-restored \
  --source-database my-db \
  --restore-point-in-time "2025-07-17T10:00:00Z"
```

**實測時間**: 15 分鐘(還原一個 50 GB 的資料庫)

### App Service 重新部署測試

如果整個 region 掛了,我們要在另一個 region 重新部署。

步驟:
1. 在新 region 建立 App Service Plan
2. 建立 App Service
3. 部署 code(從 GitHub)
4. 設定環境變數(從 Key Vault)
5. 更新 DNS(指向新的 Front Door)

**實測時間**: 45 分鐘(手動操作)

可以寫成自動化腳本,縮短到 10-15 分鐘。

## DR Checklist

最後我整理了一份 DR checklist:

### 每日檢查
- ✅ 自動備份是否成功
- ✅ 備份檔案是否可讀取
- ✅ 監控系統是否正常

### 每週檢查
- ✅ 測試環境還原備份
- ✅ 檢查 Key Vault 的 audit log
- ✅ 檢查 Application Insights 的 alert

### 每月演練
- ✅ 模擬 SQL Database 故障,從備份還原
- ✅ 模擬 App Service 掛掉,切換到備用 instance
- ✅ 檢查 RTO 和 RPO 是否符合目標

### 每季演練
- ✅ 模擬整個 region 掛掉,切換到另一個 region
- ✅ 全員參與,包含客戶方的人員
- ✅ 記錄問題,改進流程

## 文件化

我把所有步驟寫成 runbook:

```markdown
# 災難恢復手冊

## 情境 1: SQL Database 無法連線

### 檢查步驟
1. 開啟 Azure Portal,檢查 SQL Database 狀態
2. 檢查 Azure Service Health,是否有 region 故障
3. 檢查 connection string 是否正確

### 恢復步驟
如果是短暫故障(<5 分鐘):
- 等待自動恢復
- 監控 Application Insights

如果是長時間故障(>5 分鐘):
1. 執行 `restore-db.sh` 從最近的備份還原
2. 更新 App Service 的 connection string
3. 重啟 App Service
4. 驗證服務是否恢復

預估時間: 30 分鐘
```

這份文件要讓團隊的每個人都能看懂,包含新人。

## 客戶的回饋

我把 DR plan 交給客戶,客戶的 IT 主管看了之後說:

「很完整。但我有一個要求:每個月要實際演練一次,並提供演練報告。」

「這是我們的計畫,」我說,「DR plan 不是寫好就放著,要定期演練才有意義。」

「很好,」他點頭,「我看過太多公司的 DR plan 都是做心安的,真的發生災難時才發現根本沒用。」

「你們這份很務實。」

這是應該的。

**DR 是生產系統的基本要求,不是加分項。**

## 下週:上線前 Checklist

DR plan 搞定了,下週要整理上線前的 checklist。

從監控、alert、文件、培訓⋯⋯林林總總還有一堆事要確認。

上線日期定在下個月底。

終於要結束這個專案了。

(但我知道上線只是另一個開始)
