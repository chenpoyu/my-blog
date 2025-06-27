---
layout: post
title: "App Service P1V3 的選擇"
date: 2025-06-27 11:30:00 +0800
categories: [雲端架構]
tags: [Azure App Service, 成本優化, 規格選擇]
---

## 為什麼不選 P2V3？

上週壓測報告出來之後，團隊裡有人提出質疑：

「我們選 P1V3 × 2，但如果選 P2V3 × 1，價格一樣，效能更好啊？」

這是個好問題。

| 方案 | vCPU | RAM | 價格/月 |
|------|------|-----|---------|
| P1V3 × 2 | 2 × 2 = 4 | 2 × 8 GB = 16 GB | $420 |
| P2V3 × 1 | 4 | 16 GB | $420 |

看起來 CPU 和記憶體總量一樣，價格也一樣。

那為什麼選 2 個小的，而不是 1 個大的？

## High Availability

最重要的原因是：**High Availability（高可用性）**。

如果只有 1 個 instance：
- **硬體故障**：機器掛了，整個服務就掛了
- **部署中斷**：部署新版本時，服務會暫時不可用
- **Azure 維護**：Azure 進行維護時，instance 會重啟

如果有 2 個 instance：
- 一個掛了，另一個還能服務
- 部署時可以 rolling update（一個一個更新）
- Azure 維護時不會同時重啟所有 instance

**對 production 環境來說，HA 比效能更重要。**

## Auto Scaling

2 個 instance 還有另一個好處：**更靈活的 auto-scaling**。

App Service 的 auto-scaling 設定：

```
Min instances: 2
Max instances: 5
Scale-out rule: CPU > 70% for 5 minutes
Scale-in rule: CPU < 30% for 10 minutes
```

如果是 P2V3 × 1：
- 流量暴增時，只能 scale 到 P2V3 × 2、× 3⋯⋯
- 每次 scale 都是增加 4 vCPU

如果是 P1V3 × 2：
- 流量暴增時，可以 scale 到 × 3、× 4、× 5⋯⋯
- 每次 scale 只增加 2 vCPU，**更細緻**

這樣可以更精準地控制成本。

## 部署風險

部署新版本時，如果只有 1 個 instance：

```
1. 停止 instance
2. 部署新版本
3. 啟動 instance
4. 測試
5. 如果有問題，rollback
```

**在步驟 1-3 之間，服務是不可用的。**

如果有 2 個 instance，可以 rolling update：

```
1. 部署到 instance 1
2. 測試 instance 1
3. 如果 OK，instance 1 開始服務
4. 部署到 instance 2
5. 測試 instance 2
6. 如果 OK，instance 2 開始服務
```

**整個過程中，至少有 1 個 instance 在服務使用者。**

## 實際使用情況

壓測的時候，2000 併發，P1V3 × 2 的資源使用情況：

| Instance | CPU | RAM | Requests/sec |
|----------|-----|-----|-------------|
| Instance 1 | 58% | 72% | ~2400 |
| Instance 2 | 62% | 74% | ~2400 |

**每個 instance 的 CPU 使用率都在 60% 左右**，很健康。

如果改用 P2V3 × 1：

| Instance | CPU | RAM | Requests/sec |
|----------|-----|-----|-------------|
| Instance 1 | 30% | 36% | ~4800 |

看起來 CPU 只用了 30%，很浪費對吧？

但問題是：**沒有 redundancy**。

一旦這個 instance 掛了，整個服務就掛了。

## 成本考量

有人會說：「那我們用 P2V3 × 2，不是更好？」

對，但那樣成本會變成 **$840/月**（雙倍）。

而我們的測試顯示：**P1V3 × 2 已經足夠應付 2000 併發**。

多花錢買更高規格，不一定能帶來對等的效益。

## 什麼時候該升級到 P2V3？

如果未來流量成長，什麼時候該升級？

我的判斷標準：

1. **CPU 長期 > 70%**：表示資源不夠了
2. **Response time 開始變慢**：P95 > 1 秒
3. **Scale out 到 4-5 個 instance**：太多小 instance 不好管理

目前我們的 CPU 使用率是 60%，還有 buffer。

而且可以先試著 scale out 到 P1V3 × 3，成本只增加 50%（從 $420 到 $630）。

如果 × 3 還不夠，再考慮升級到 P2V3。

## P1V3 vs B 系列

還有人問：「為什麼不用 B 系列（Basic）？更便宜。」

| 規格 | vCPU | RAM | 價格/月 |
|------|------|-----|---------|
| B1 | 1 | 1.75 GB | $55 |
| B2 | 2 | 3.5 GB | $110 |
| B3 | 4 | 7 GB | $220 |
| **P1V3** | **2** | **8 GB** | **$210** |

看起來 B3 (4 vCPU) 比 P1V3 (2 vCPU) 便宜又強？

但 B 系列有個致命缺點：**CPU 是 shared 的**。

B 系列的 CPU 不是專用的，是跟其他客戶共用實體機器的 CPU。

在**低流量**時，B 系列很划算。

但在**高流量**時，如果其他客戶也在用，你的 CPU 效能會被擠壓。

**Production 環境不適合用 B 系列。**

## P1V3 vs S 系列

S 系列（Standard）呢？

| 規格 | vCPU | RAM | 價格/月 |
|------|------|-----|---------|
| S1 | 1 | 1.75 GB | $70 |
| S2 | 2 | 3.5 GB | $140 |
| S3 | 4 | 7 GB | $280 |
| **P1V3** | **2** | **8 GB** | **$210** |

S 系列的 CPU 是專用的（不像 B 系列是 shared）。

但 S 系列的硬體比較舊，P 系列用的是 **Dv3 (Intel Xeon Platinum 8370C)**，效能更好。

而且 P 系列支援：
- **Premium SSD**：更快的磁碟 I/O
- **Faster scale-up/scale-out**：擴展更快
- **Better network throughput**：網路頻寬更高

所以 P1V3 雖然貴一點，但 CP 值更高。

## 最終決策

經過這些分析，我們最終選擇：

**P1V3 × 2，搭配 auto-scaling (max 5 instances)**

理由：
1. ✅ High Availability
2. ✅ 靈活的 auto-scaling
3. ✅ Rolling update 不中斷服務
4. ✅ 成本合理（$420/月）
5. ✅ 足夠應付 2000 併發

而且如果未來流量成長：
- 先 scale out 到 × 3、× 4（增加成本 50%-100%）
- 再考慮升級到 P2V3（增加成本 100%）

**雲端的好處就是：隨時可以調整，不用一開始就做完美的選擇。**

## 下週：Key Vault 和 API 安全

壓測和規格選擇都搞定了，下週開始要處理安全性。

第一個要做的是：**把所有機密資訊搬到 Key Vault**。

現在 connection string、API key 都還放在 Azure Configuration，這不是好做法。

該認真處理安全性了。
