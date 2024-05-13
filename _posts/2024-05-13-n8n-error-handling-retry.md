---
layout: post
title: "n8n 錯誤處理與重試機制"
date: 2024-05-13 15:40:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Error Handling, Reliability]
---

這週深入研究了 n8n 的錯誤處理機制。做自動化流程最怕的就是執行到一半掛掉，資料處理了一半、通知發了一半，要收拾殘局很麻煩。

好的錯誤處理機制可以讓 workflow 更穩定，即使出錯也能優雅地恢復或通知。

## 常見的錯誤場景

在實際使用 n8n 時，會遇到各種錯誤：

**網路問題**：API 呼叫失敗、資料庫連不上、timeout

**資料問題**：格式錯誤、必填欄位缺失、資料型態不對

**邏輯錯誤**：除以零、陣列越界、undefined 存取

**外部服務問題**：第三方 API 掛了、rate limit 超過、認證失效

**系統資源**：記憶體不足、磁碟滿了

這些錯誤如果沒處理好，workflow 就會中斷，影響後續流程。

## Node 層級的錯誤處理

n8n 的每個 Node 都可以設定錯誤處理：

點擊 Node → Settings → Continue On Fail

打開這個選項後，即使這個 Node 出錯，workflow 也會繼續執行。錯誤資訊會放在輸出資料的 `error` 欄位。

**使用時機**：

某個 API 呼叫可能失敗，但不想讓整個 workflow 停下來：

```
HTTP Request (Continue On Fail = true)
  ↓
IF Node：檢查是否有 error
  ↓ (有錯誤)
  發送告警
  ↓ (沒錯誤)
  繼續處理
```

## Retry 機制

網路暫時不穩、服務短暫不可用，這種暫時性錯誤可以重試。

在 Node Settings → Retry On Fail：

- **Max Tries**：最多重試幾次
- **Wait Between Tries**：每次重試間隔多久（毫秒）

比如 HTTP Request 可能因為網路問題失敗，設定重試 3 次，間隔 1000ms：

```
Max Tries: 3
Wait Between Tries: 1000
```

這樣第一次失敗會等 1 秒後重試，如果還是失敗，再等 1 秒重試，最多 3 次。

**指數退避（Exponential Backoff）**：

更好的做法是用指數退避，每次重試間隔加倍：

n8n 沒有內建，但可以用 Code Node 模擬：

```javascript
const maxRetries = 3;
let lastError;

for (let i = 0; i < maxRetries; i++) {
  try {
    // 執行操作
    const result = await callAPI();
    return [{ json: result }];
  } catch (error) {
    lastError = error;
    if (i < maxRetries - 1) {
      const delay = Math.pow(2, i) * 1000; // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

throw lastError; // 重試失敗，拋出錯誤
```

## Error Workflow

n8n 可以建立專門處理錯誤的 workflow。

建立一個新 workflow：`Error Handler`

1. **Error Trigger Node**
   - Trigger On: Workflow Error
2. **Code Node**：整理錯誤資訊
3. **Slack Node**：發送告警
4. **Postgres Node**：記錄錯誤到資料庫

Error Trigger 會在任何 workflow 出錯時被觸發，可以拿到：

- `{{ $json.workflow.name }}`：哪個 workflow 出錯
- `{{ $json.execution.id }}`：execution ID
- `{{ $json.error.message }}`：錯誤訊息
- `{{ $json.error.stack }}`：stack trace

整理成通知訊息：

```javascript
const error = $input.first().json;

const message = `
⚠️ Workflow Error Alert

Workflow: ${error.workflow.name}
Execution ID: ${error.execution.id}
Time: ${new Date().toISOString()}
Error: ${error.error.message}

Stack Trace:
${error.error.stack}

View in n8n: http://n8n.company.com/execution/${error.execution.id}
`;

return [{ json: { message } }];
```

然後發送到 Slack 或 email。

## 在特定 Workflow 捕捉錯誤

如果只想處理特定 workflow 的錯誤，不想用全域的 Error Workflow，可以在該 workflow 裡加上錯誤處理。

用 Try-Catch 模式：

1. 主要流程的 Node 都設定 Continue On Fail = true
2. 在最後加上一個 IF Node 檢查是否有錯誤
3. 如果有錯誤，執行錯誤處理分支

```javascript
// IF Node 條件
{{ $('HTTP Request').item.json.error !== undefined }}
```

如果 HTTP Request 有錯誤，走 True 分支處理錯誤。

## 部分失敗處理

有時候 workflow 處理多筆資料，其中一筆失敗不應該影響其他資料。

比如從 Google Sheets 讀取 100 筆訂單，發送 100 封 email。其中 5 封因為 email 格式錯誤失敗，不應該讓其他 95 封也不發。

**解決方法**：

用 Split In Batches Node 搭配 Continue On Fail：

```
Google Sheets (讀取訂單)
  ↓
Split In Batches (每次處理 1 筆)
  ↓
Gmail Node (Continue On Fail = true)
  ↓
Code Node (收集成功和失敗)
  ↓
(迴圈)
  ↓
總結報告
```

在 Code Node 記錄每筆的結果：

```javascript
const item = $input.first().json;

// 檢查是否有錯誤
if (item.error) {
  $execution.customData.failed = $execution.customData.failed || [];
  $execution.customData.failed.push({
    orderId: item.orderId,
    error: item.error.message
  });
} else {
  $execution.customData.success = $execution.customData.success || [];
  $execution.customData.success.push({
    orderId: item.orderId
  });
}

return [$input.first()];
```

最後產生報告：

```javascript
const summary = {
  total: $execution.customData.success.length + $execution.customData.failed.length,
  success: $execution.customData.success.length,
  failed: $execution.customData.failed.length,
  failedItems: $execution.customData.failed
};

return [{ json: summary }];
```

## Dead Letter Queue

對於無法處理的錯誤資料，可以放到 Dead Letter Queue（DLQ），之後手動處理。

建立一個「錯誤資料」資料表：

```sql
CREATE TABLE error_queue (
    id SERIAL PRIMARY KEY,
    workflow_name VARCHAR(100),
    error_type VARCHAR(50),
    error_message TEXT,
    failed_data JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'pending'
);
```

在錯誤處理分支：

```javascript
// Code Node
const failedItem = $input.first().json;

return [{
  json: {
    workflow_name: $workflow.name,
    error_type: 'validation_error',
    error_message: failedItem.error.message,
    failed_data: failedItem,
    status: 'pending'
  }
}];
```

然後寫到資料庫。

定期檢查 DLQ，手動修正資料後重新處理。

## 監控與告警

除了被動處理錯誤，更重要的是主動監控。

**健康檢查 Workflow**：

建立一個定期檢查的 workflow：

1. Schedule Trigger（每 5 分鐘）
2. HTTP Request：檢查關鍵服務是否正常
3. Postgres：檢查資料庫連線
4. 檢查最近的 execution 失敗率
5. 如果異常，發送告警

**失敗率告警**：

```javascript
// 查詢最近 1 小時的 execution
const executions = $('HTTP Request').first().json;

const total = executions.length;
const failed = executions.filter(e => e.finished === false).length;
const failureRate = failed / total;

if (failureRate > 0.1) { // 失敗率超過 10%
  return [{
    json: {
      alert: true,
      message: `Warning: Failure rate is ${(failureRate * 100).toFixed(1)}%`,
      total: total,
      failed: failed
    }
  }];
}

return [{ json: { alert: false } }];
```

## Circuit Breaker 模式

如果外部服務持續失敗，不應該一直重試，會浪費資源。可以用 Circuit Breaker 模式。

在 Redis 或資料庫記錄服務狀態：

```javascript
// 檢查 circuit 狀態
const circuitKey = 'circuit:external-api';
const circuitState = await redis.get(circuitKey); // 'open', 'closed', 'half-open'

if (circuitState === 'open') {
  // Circuit 斷開，不呼叫 API
  throw new Error('Circuit breaker is open');
}

try {
  // 嘗試呼叫 API
  const result = await callAPI();
  
  // 成功，reset circuit
  await redis.set(circuitKey, 'closed');
  
  return result;
} catch (error) {
  // 失敗，記錄失敗次數
  const failCount = await redis.incr(`${circuitKey}:fails`);
  
  if (failCount > 5) {
    // 連續失敗 5 次，打開 circuit
    await redis.set(circuitKey, 'open');
    await redis.expire(circuitKey, 60); // 1 分鐘後自動恢復到 half-open
  }
  
  throw error;
}
```

## Idempotency（冪等性）

錯誤重試時要注意冪等性：同樣的操作執行多次，結果應該一樣。

**不具冪等性的操作**：

```javascript
// 每次執行都會新增一筆
INSERT INTO orders (order_id, amount) VALUES ('12345', 1000);
```

如果重試，會插入多筆相同的訂單。

**改成冪等性**：

```javascript
// 先檢查是否已存在
INSERT INTO orders (order_id, amount) VALUES ('12345', 1000)
ON CONFLICT (order_id) DO NOTHING;
```

或者用 Upsert，或者在執行前先檢查。

## 實務建議

**分級處理**：不是所有錯誤都一樣嚴重。分成 Critical、Warning、Info 不同等級。

**告警去重**：同樣的錯誤不要一直發通知。可以設定 5 分鐘內同樣錯誤只發一次。

**保留現場**：錯誤發生時，把相關資料都記下來，方便事後分析。

**漸進式重試**：不要無限重試，設定最大次數。而且重試間隔要漸進增加。

**優雅降級**：如果某個非關鍵功能失敗，讓 workflow 繼續執行，不要整個停掉。

經過這週的研究，對 n8n 的穩定性更有信心了。雖然還是會遇到錯誤，但至少知道怎麼處理和恢復。

下週想研究 n8n 的進階部署，像是 cluster 模式、queue mode、監控整合等，讓正式環境更穩定。
