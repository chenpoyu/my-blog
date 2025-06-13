---
layout: post
title: "Load Testing Engine 塞車事件"
date: 2025-06-13 09:20:00 +0800
categories: [測試]
tags: [Azure Load Testing, 壓力測試, 基礎設施]
---

## 終於要測 2000 併發了

經過一個多月的優化，我們終於準備好測 2000 併發了。

這是客戶的需求：「系統要能同時承受 2000 個使用者在線」。

我信心滿滿地打開 Azure Load Testing，設定好參數：

```
Virtual Users: 2000
Duration: 10 minutes
Ramp-up: 2 minutes (從 0 到 2000)
Engine Instances: Auto
```

按下「Run」。

然後⋯⋯等。

## 「Provisioning engines...」

Azure Load Testing 的執行流程是這樣：

1. **Provisioning**: 分配測試機器
2. **Configuring**: 設定環境
3. **Executing**: 執行測試
4. **Done**: 產生報告

之前測 100、500、1000 併發，Provisioning 通常 3-5 分鐘就好了。

但這次，等了 10 分鐘，還是在 Provisioning。

「奇怪⋯⋯」

我去看 Azure Portal 的 Activity Log：

```
Status: Provisioning
Message: Waiting for available capacity...
```

又等了 5 分鐘。

```
Status: Failed
Error: Unable to provision load testing engines. 
Capacity not available in region 'East Asia'. 
Please try again later or reduce the load.
```

「幹，又來了。」

## Azure Load Testing 的容量限制

上次遇到容量不足，是因為在尖峰時段（工作日下午）測試。

這次我特地選在週六早上 9 點，應該比較少人用才對。

結果還是一樣。

我去查了一下 Azure Load Testing 的文件，發現：

> Load testing engines are shared resources across all Azure customers in the same region. 
> If capacity is not available, the test run will fail or be queued.

**共享資源**。

這表示：
- 你不能保證一定有機器可以用
- 如果同一時間很多人在測試，你可能要排隊
- 排隊等不到，就會失敗

「那我要怎麼測 2000 併發？」

## 選項 1：換 region

第一個想法是：換一個比較少人用的 region。

我們的 App Service 在 **East Asia**（東亞），但 Azure Load Testing 可以選其他 region。

例如：
- **Southeast Asia**（東南亞）
- **Australia East**（澳洲東部）
- **West US**（美國西部）

但這有個問題：**測試機器跟 App Service 的網路延遲**。

如果測試機器在美國，App Service 在東亞，光是網路延遲就 150-200ms。

這樣測出來的結果根本不準。

## 選項 2：減少 engine instances

Azure Load Testing 會自動決定要用幾台 engine（測試機器）。

2000 併發，它可能會分配 4-8 台 engine。

但如果容量不夠，可以手動設定減少 engine 數量：

```
Engine Instances: 2  // 手動指定 2 台
```

但這也有風險：**2 台 engine 可能跑不動 2000 併發**。

每台 engine 要模擬 1000 個使用者，負載很重，可能會影響測試準確性。

## 選項 3：分批測試

既然一次跑 2000 有問題，那就分批測試：

1. 先測 1000 併發
2. 再測另外 1000 併發
3. 加總起來推算 2000 的效能

但這個方法也不完美，因為：
- **資源競爭不一樣**：1000 + 1000 ≠ 2000
- **快取狀態不一樣**：第一批測試會把資料放進 cache
- **無法驗證真正的 2000 併發情境**

## 最後的解法：自己架 k6 agent

既然 Azure Load Testing 不穩定，那就自己架 k6。

我在 Azure 開了兩台 VM：
- **VM 規格**: Standard D8s v3 (8 vCPUs, 32 GB RAM)
- **數量**: 2 台
- **位置**: East Asia（跟 App Service 同 region）

每台 VM 可以跑 1000-1500 併發，兩台合起來足夠跑 2000。

在每台 VM 上安裝 k6：

```bash
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

然後寫一個腳本，讓兩台 VM 同時執行測試：

```bash
# vm1.sh
k6 run --vus 1000 --duration 10m test.js

# vm2.sh
k6 run --vus 1000 --duration 10m test.js
```

用 SSH 同時執行：

```bash
ssh vm1 'bash vm1.sh' &
ssh vm2 'bash vm2.sh' &
wait
```

## 測試結果

這次終於成功跑完 2000 併發的測試：

| 指標 | 數值 |
|-----|-----|
| 總請求數 | 2,850,432 |
| 平均回應時間 | 420ms |
| P50 回應時間 | 336ms |
| P90 回應時間 | 570ms |
| P95 回應時間 | 850ms |
| P99 回應時間 | 1200ms |
| 錯誤率 | 1.2% |

**P50: 336ms, P90: 570ms, P99: 1200ms**。

符合客戶的需求（P95 < 1000ms）。

但錯誤率還是有 1.2%，主要是：
- 資料庫偶爾 timeout（0.5%）
- 金流 API 失敗（0.4%）
- Redis connection timeout（0.3%）

## 改進：再跑一次

我們針對這些錯誤做了一些優化：
1. **資料庫 timeout**: 調整 connection pool 和 index
2. **金流 API**: 加強 circuit breaker 和 retry
3. **Redis timeout**: 增加 connection pool

改完後再跑一次：

| 指標 | 第一次 | 第二次 |
|-----|-------|--------|
| P50 | 336ms | 336ms |
| P90 | 570ms | 570ms |
| P99 | 1200ms | 680ms |
| 錯誤率 | 1.2% | 0.3% |

**P99 從 1200ms 降到 680ms，錯誤率從 1.2% 降到 0.3%**。

這個結果我們很滿意了。

## 搶券情境

除了一般使用情境，我們還測試了搶券：

```
Scenario: 500 人同時搶 100 張優惠券
Duration: 3 分鐘
```

結果：

| 指標 | 數值 |
|-----|-----|
| P50 | 850ms |
| P90 | 1500ms |
| P99 | 2300ms |
| 成功領券 | 100 人 |
| 失敗（搶光）| 380 人 |
| 失敗（timeout）| 20 人 |

搶券的回應時間比較慢（P50: 850ms），但還在可接受範圍。

而且因為改成 queue 處理，沒有出現資料庫死鎖的問題。

## 最終規格

經過這一個多月的測試和優化，我們確定了正式環境的規格：

| 服務 | 規格 | 成本/月 |
|-----|------|---------|
| App Service | P1V3 (2 instances) | $420 |
| SQL Database | P2 (250 DTU) | $930 |
| Redis | Standard C2 (2.5 GB) | $146 |
| Front Door | Standard | $35 + 流量費 |
| Service Bus | Standard | $10 |
| **Total** | | **~$1,600** |

客戶看了之後說：「成本比我們預期的低，效能也符合需求。可以。」

終於通過了。

## 心得

這次壓測的經驗讓我學到：

1. **雲端服務也有容量限制**：即使是 Azure 這種大廠，也不能保證隨時有資源
2. **不要太依賴單一工具**：Azure Load Testing 不穩定，就要有 Plan B（自己架 k6）
3. **測試環境要接近正式環境**：網路延遲、地理位置都會影響結果
4. **壓測不是一次就好**：要反覆測試、優化、再測試

下週開始，要進入安全性和上線準備的階段了。

壓測算是告一段落。
