---
layout: post
title: "n8n Webhook 與 API 觸發機制"
date: 2024-04-29 09:55:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Webhook, API]
---

前幾週都是用排程觸發 workflow，每天或每小時固定執行。但很多場景需要即時觸發，比如：訂單成立時馬上發通知、使用者註冊時自動寄歡迎信、有新的 issue 時建立工單等。

這週研究了 n8n 的 Webhook 和 API 觸發機制，讓外部系統能主動呼叫 workflow。

## Webhook Trigger 基礎

Webhook 是一種「反向 API」的概念。傳統 API 是我們主動去呼叫；Webhook 是對方有事件發生時，主動呼叫我們提供的 URL。

n8n 的 Webhook Node 會產生一個 URL，讓外部系統可以 POST 資料過來，觸發 workflow 執行。

建立一個簡單的 Webhook workflow：

1. 新增 Webhook Node
2. Method 選 POST
3. Path 填 `/order-created`
4. 點 `Listen For Test Event`

會看到產生的 URL：`http://localhost:5678/webhook/order-created`

測試一下：

```bash
curl -X POST http://localhost:5678/webhook/order-created \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "12345",
    "customer": "John Doe",
    "amount": 1500
  }'
```

在 n8n 介面會看到收到的資料，可以用 `{{ $json.orderId }}` 這樣存取。

## 實際應用：訂單通知系統

假設電商網站用 PHP 寫的，每當有新訂單，要通知多個部門（客服、倉庫、財務）。

以前可能是在 PHP code 裡寫一堆發信、發 Slack、寫日誌的邏輯。現在可以改成呼叫 n8n webhook，讓 n8n 處理所有通知。

**PHP 端改動**：

```php
// 訂單建立後
function notifyNewOrder($orderData) {
    $webhookUrl = 'https://n8n.company.com/webhook/order-created';
    
    $ch = curl_init($webhookUrl);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($orderData));
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
}
```

**n8n Workflow**：

1. **Webhook Node**：接收訂單資料
2. **Code Node**：整理要通知的內容
3. **Split Out Node**：分成多個分支
4. **Gmail Node**：寄信給財務
5. **Slack Node**：通知客服頻道
6. **HTTP Request Node**：呼叫倉庫系統 API
7. **Postgres Node**：記錄到日誌表

這樣所有通知邏輯都在 n8n，PHP 只要呼叫一個 webhook 就好。之後要調整通知內容或新增通知管道，不用改 PHP code。

## Webhook 的回應

Webhook Node 可以設定回應內容：

- **Response Mode**：
  - `On Received`：收到請求馬上回應（預設）
  - `Last Node`：workflow 執行完才回應
  - `Using 'Respond to Webhook' Node`：在特定位置回應

如果 workflow 執行時間短（幾秒內），可以用 `Last Node`，讓呼叫方知道執行結果。

如果執行時間長，應該用 `On Received` 馬上回應 200 OK，避免呼叫方 timeout。

設定回應內容：

```javascript
// 在 Respond to Webhook Node
{
  "status": "success",
  "message": "Order notification sent",
  "orderId": "{{ $json.orderId }}"
}
```

## 安全性考量

Webhook URL 如果被人知道，任何人都能呼叫，會有安全風險。要加上驗證機制。

**方法 1：在 URL 加上 secret token**

```
https://n8n.company.com/webhook/order-created?token=abc123xyz
```

在 Webhook Node 用 IF Node 檢查：

```javascript
{{ $('Webhook').item.json.query.token === 'abc123xyz' }}
```

不對就回應 401 Unauthorized。

**方法 2：檢查 Header**

很多服務（GitHub、Stripe）會在 header 加上簽章：

```javascript
// 檢查 X-Signature header
const receivedSignature = $('Webhook').item.json.headers['x-signature'];
const secret = 'your-secret-key';
const payload = JSON.stringify($('Webhook').item.json.body);

const crypto = require('crypto');
const expectedSignature = crypto
  .createHmac('sha256', secret)
  .update(payload)
  .digest('hex');

if (receivedSignature !== expectedSignature) {
  throw new Error('Invalid signature');
}
```

**方法 3：IP 白名單**

如果呼叫方的 IP 固定，可以在 Load Balancer 或防火牆層級限制。

## 同步 vs 非同步

有些場景需要馬上知道執行結果（同步），有些只要觸發就好（非同步）。

**同步模式**：

```javascript
// Workflow 設定 Response Mode = Last Node
// 在最後加一個 Respond to Webhook Node
{
  "success": true,
  "processedItems": {{ $json.count }},
  "errors": {{ $json.errors }}
}
```

呼叫方會等 workflow 執行完，拿到結果。

**非同步模式**：

```javascript
// Workflow 設定 Response Mode = On Received
// 馬上回應
{
  "status": "accepted",
  "message": "Processing started"
}
```

呼叫方馬上拿到回應，workflow 在背景執行。

如果要通知執行結果，可以讓呼叫方提供一個 callback URL，執行完後再呼叫回去。

## 批次處理

如果短時間內收到很多 webhook 呼叫，逐一處理可能會過載。可以用 Queue 模式。

n8n 本身沒有內建 queue，但可以整合外部的 message queue：

**Workflow 1：接收 webhook**

1. Webhook Node
2. Redis Node：把資料放到 Redis list

**Workflow 2：處理 queue**

1. Schedule Trigger：每分鐘執行
2. Redis Node：從 list 取出資料（LPOP）
3. 處理資料
4. Loop：持續處理直到 queue 空了

或者用 RabbitMQ、Kafka 這些專門的 message queue。

## Production Webhook 與 Test Webhook

n8n 的 webhook 有兩種模式：

**Test Webhook**：在編輯 workflow 時，點 `Listen For Test Event` 產生的 URL。只有在編輯模式下才能用，用來測試。

**Production Webhook**：workflow 啟用（Active）後，會有一個固定的 production URL。這才是要給外部系統呼叫的。

Production URL 的格式：
- `http://your-domain/webhook/<path>`
- 或 `http://your-domain/webhook-test/<path>`（如果 workflow 沒啟用）

要記得把 workflow 啟用，不然 production webhook 不會運作。

## 整合第三方服務的 Webhook

很多 SaaS 服務支援 webhook，可以在特定事件發生時通知你：

**GitHub Webhook**：repo 有新的 push、issue、PR 時通知

在 GitHub repo settings → Webhooks → Add webhook：
- Payload URL: 你的 n8n webhook URL
- Content type: application/json
- 選擇要監聽的事件

n8n workflow 收到後可以：
- 發 Slack 通知
- 建立 Jira ticket
- 觸發 CI/CD

**Stripe Webhook**：有新的付款、退款時通知

**Shopify Webhook**：訂單建立、庫存變動時通知

這些服務的 webhook 通常會包含簽章，要驗證才安全。

## 錯誤處理

Webhook workflow 一定要做好錯誤處理，不然外部系統會一直重試，或者收不到正確回應。

```javascript
try {
  // 處理邏輯
  const result = processOrder($json);
  
  return [{
    json: {
      status: 'success',
      data: result
    }
  }];
} catch (error) {
  // 記錄錯誤
  console.error('Error processing webhook:', error);
  
  // 發送告警
  // ...
  
  // 回應錯誤（但不要讓呼叫方無限重試）
  return [{
    json: {
      status: 'error',
      message: error.message
    }
  }];
}
```

如果是暫時性錯誤（網路問題、服務暫時不可用），可以回應 5xx，讓呼叫方重試。

如果是永久性錯誤（資料格式錯誤、business logic 問題），回應 4xx，告訴呼叫方不要再重試了。

## Webhook 監控

Production 環境要監控 webhook 的狀況：

- 收到多少請求
- 成功率
- 回應時間
- 錯誤類型

n8n 的 Executions 頁面可以看到每次執行的記錄，但如果量大，要整合到專門的監控系統（Prometheus、Grafana）。

可以在 workflow 最後加一個 node 記錄 metrics：

```javascript
const metrics = {
  timestamp: new Date().toISOString(),
  webhook_path: '/order-created',
  execution_time: Date.now() - $execution.startTime,
  status: 'success',
  items_processed: $json.length
};

// 發送到監控系統
```

## 實務經驗

Webhook 模式很適合事件驅動的場景，即時性比排程好很多。不過要注意：

**冪等性**：webhook 可能會重複呼叫（網路問題、重試機制），要確保重複執行不會有副作用。

**順序問題**：如果有多個 webhook 同時進來，執行順序可能不是你預期的。如果有順序要求，要加上 queue 或 lock 機制。

**流量控制**：如果 webhook 被大量呼叫，n8n 可能撐不住。要做好 rate limiting，或者在前面加一層 API Gateway。

下週想研究 n8n 的 Sub-workflow 和模組化設計，看怎麼把常用的邏輯包裝成可重用的元件。
