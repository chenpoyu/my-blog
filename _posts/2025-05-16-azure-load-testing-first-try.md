---
layout: post
title: "Azure Load Testing：看起來美好，實際上⋯"
date: 2025-05-16 11:45:00 +0800
categories: [測試]
tags: [Azure Load Testing, k6, 壓力測試]
---

## 第一次用 Azure Load Testing

上週把 k6 腳本寫好之後，這週開始在 Azure Load Testing 上跑測試。

Azure Load Testing 的賣點是：
- **不用自己架測試機器**
- **自動分配 load generator**
- **整合 Azure Monitor**
- **產生漂亮的圖表**

聽起來很完美，對吧？

我照著官方文件，建立了一個 Load Testing resource，上傳 k6 腳本，設定好參數，按下「Run」。

然後⋯⋯等。

## 等了 10 分鐘才開始

Azure Load Testing 的執行流程是這樣：

1. **Provisioning**: 分配測試機器 (~3-5 分鐘)
2. **Configuring**: 設定環境、安裝 k6 (~2-3 分鐘)
3. **Executing**: 執行測試腳本
4. **Done**: 產生報告

我看著 Portal 上的進度條：

```
Provisioning... (3 分鐘)
Configuring... (2 分鐘)
Executing...
```

光是前置作業就花了 5 分鐘。

「如果我在本地跑 k6，5 秒就可以開始測試了⋯⋯」

但想想也合理，畢竟 Azure 要幫你準備機器、安裝環境。

## 第一個測試：Baseline (100 併發)

我們的第一個測試是 Baseline，目標是 100 個併發使用者。

k6 腳本長這樣：

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
    stages: [
        { duration: '1m', target: 100 },   // ramp up
        { duration: '5m', target: 100 },   // stay
        { duration: '1m', target: 0 },     // ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% 的請求要在 500ms 內
        http_req_failed: ['rate<0.01'],    // 錯誤率要低於 1%
    },
};

export default function () {
    // 模擬使用者開啟 App
    let responses = http.batch([
        ['GET', 'https://api.example.com/api/member/profile'],
        ['GET', 'https://api.example.com/api/points/balance'],
        ['GET', 'https://api.example.com/api/coupons/available'],
        ['GET', 'https://api.example.com/api/activities/ongoing'],
    ]);

    // 檢查回應
    responses.forEach(res => {
        check(res, {
            'status is 200': (r) => r.status === 200,
            'response time < 500ms': (r) => r.timings.duration < 500,
        });
    });

    sleep(Math.random() * 3 + 2);  // 隨機等待 2-5 秒
}
```

按下「Run」，等了 5 分鐘，測試終於開始跑。

## 結果看起來不錯

7 分鐘後（測試本身跑 7 分鐘），Azure Load Testing 產生了報告：

| 指標 | 數值 |
|-----|-----|
| 總請求數 | 120,543 |
| 平均回應時間 | 85ms |
| P95 回應時間 | 320ms |
| P99 回應時間 | 580ms |
| 錯誤率 | 0.03% |

**P95 在 320ms，符合我們的 threshold (500ms)**。

而且 Azure Load Testing 還自動產生了很多圖表：
- Response time 隨時間變化
- Throughput (RPS)
- Error rate
- 每個 API endpoint 的分布

這些圖表很好看，可以直接拿去給客戶看。

## 但有個問題

在看報告的時候，我注意到一個奇怪的現象：

**前 1 分鐘的回應時間特別高。**

| 時間區間 | 平均回應時間 |
|---------|------------|
| 0-1 min | 850ms |
| 1-2 min | 120ms |
| 2-7 min | 85ms |

為什麼前 1 分鐘這麼慢？

我去看 Application Insights 的 trace，發現：
- **前 1 分鐘有大量的 cold start**
- App Service 的 instance 從 1 個自動 scale 到 3 個
- 新的 instance 啟動需要時間

「喔，原來是 auto-scaling 的問題。」

這在實際使用上不是問題（使用者不會突然從 0 暴增到 100），但在測試時會影響結果。

## Unit Test vs Scenario

Azure Load Testing 支援兩種測試模式：

### 1. Unit Test (Quick Test)
- 簡單的 API 測試
- 固定的 URL 和參數
- 不用寫 k6 腳本
- 適合快速驗證單一 API

### 2. Scenario (k6 Script)
- 複雜的使用情境
- 可以寫 JavaScript logic
- 支援多個 API、有狀態的操作
- 適合真實的使用者行為

我們用的是 Scenario，因為要模擬真實的使用者行為（開 App → 看優惠券 → 看活動 → 點進去⋯⋯）。

但如果只是想快速測試單一 API，Unit Test 會更快。

## 第一次的成本

測試跑完後，我去看 Azure 的帳單。

這次測試的成本：
- **Engine hours**: $0.80（跑了 12 分鐘，包含前置時間）
- **Data transfer**: $0.05（測試流量）
- **Total**: $0.85

還蠻便宜的。

如果一個月跑 20 次測試，也才 $17。

比自己架機器便宜多了。

## 遇到的第一個坑

週四我們要跑第二次測試（200 併發），結果 Azure Load Testing 跳出錯誤：

```
Failed to provision load testing engines. 
Error: Capacity not available in region 'East Asia'.
Please try again later or use a different region.
```

什麼？容量不足？

我試了幾次，都是同樣的錯誤。

去查了一下，原來 Azure Load Testing 是共享資源，如果同一時間有太多人在用，就會沒有足夠的 engine。

「那怎麼辦？」

後來我換了一個時間（晚上 10 點），就成功了。

「看來 Azure Load Testing 也有尖峰和離峰⋯⋯」

這是我沒想到的問題。

## 小結

Azure Load Testing 的優點：
- ✅ 不用自己架機器
- ✅ 自動產生漂亮的圖表
- ✅ 整合 Azure Monitor
- ✅ 成本不貴

但也有缺點：
- ⚠️ 前置時間長（5 分鐘）
- ⚠️ 可能遇到容量不足
- ⚠️ 無法完全控制測試環境

對於我們的需求來說，Azure Load Testing 還是值得用的。

雖然有些小問題，但整體體驗不錯。

## 下一步：增加併發數

這週測試了 100 併發，下週要測 500、1000、2000。

真正的挑戰才剛開始。

尤其是那個「搶券流程」，27 支 API⋯⋯

光想就覺得可怕。
