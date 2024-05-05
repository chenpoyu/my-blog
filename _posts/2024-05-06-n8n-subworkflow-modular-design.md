---
layout: post
title: "n8n Sub-Workflow 與模組化設計"
date: 2024-05-06 13:15:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Sub-Workflow, 架構設計]
---

這週研究了 n8n 的 Sub-Workflow 功能。做了幾個 workflow 後發現，有些邏輯會重複出現，像是發送通知、資料驗證、錯誤處理等。如果每個 workflow 都重寫一次，維護起來很麻煩。

n8n 的 Execute Workflow Node 可以呼叫另一個 workflow，就像函式呼叫一樣。這樣可以把常用的邏輯抽出來，達到模組化和重用。

## Sub-Workflow 的概念

想像一下傳統程式設計，我們會把重複的邏輯寫成函式：

```javascript
function sendNotification(message, recipients) {
  // 發送邏輯
}

// 在多個地方呼叫
sendNotification('訂單已建立', ['sales@company.com']);
sendNotification('庫存不足', ['purchasing@company.com']);
```

n8n 的 Sub-Workflow 就是類似概念。建立一個「通知 Workflow」，讓其他 workflow 都能呼叫它。

## 建立第一個 Sub-Workflow

先建立一個簡單的「發送通知」workflow：

**Workflow: Send Notification**

1. **Webhook Node**（當 trigger）或直接用 **Execute Workflow Trigger**
2. **IF Node**：判斷通知類型（email、slack、sms）
3. **Gmail Node**：如果是 email
4. **Slack Node**：如果是 slack
5. **HTTP Request Node**：如果是 sms

設定 Execute Workflow Trigger：
- 不需要特別設定，只要有這個 node，其他 workflow 就能呼叫

儲存這個 workflow，命名為 `Send Notification`。

## 呼叫 Sub-Workflow

在主要的 workflow 裡，用 Execute Workflow Node 呼叫：

**主 Workflow: Process Order**

1. Webhook：收到訂單資料
2. 處理訂單邏輯
3. **Execute Workflow Node**：
   - Source: Database
   - Workflow: 選擇 `Send Notification`
   - 傳入資料：
   ```json
   {
     "type": "email",
     "to": "sales@company.com",
     "subject": "新訂單通知",
     "message": "訂單編號：{{ $json.orderId }}"
   }
   ```

Execute Workflow Node 會執行 `Send Notification` workflow，並把資料傳過去。

在 `Send Notification` workflow 可以用 `{{ $json.type }}` 存取傳入的資料。

## 傳遞資料

Sub-workflow 就像函式，要傳入參數，也要回傳結果。

**傳入資料**：

在 Execute Workflow Node 設定要傳入的資料：

```json
{
  "orderId": "{{ $json.orderId }}",
  "customer": "{{ $json.customer }}",
  "amount": {{ $json.amount }}
}
```

**回傳資料**：

在 Sub-workflow 的最後一個 node，它的輸出會自動回傳給呼叫方。

比如 `Send Notification` 最後回傳：

```json
{
  "status": "sent",
  "timestamp": "{{ $now }}",
  "recipients": ["sales@company.com"]
}
```

主 workflow 的 Execute Workflow Node 之後就能用 `{{ $json.status }}` 取得結果。

## 實際案例：資料驗證模組

建立一個通用的資料驗證 workflow：

**Workflow: Validate Data**

```javascript
// Code Node
const data = $input.first().json;
const rules = $input.first().json.rules;
const errors = [];

for (const field in rules) {
  const rule = rules[field];
  const value = data[field];
  
  if (rule.required && !value) {
    errors.push(`${field} is required`);
  }
  
  if (rule.type === 'email' && value && !value.includes('@')) {
    errors.push(`${field} must be a valid email`);
  }
  
  if (rule.minLength && value && value.length < rule.minLength) {
    errors.push(`${field} must be at least ${rule.minLength} characters`);
  }
}

return [{
  json: {
    valid: errors.length === 0,
    errors: errors,
    data: data
  }
}];
```

在任何需要驗證資料的 workflow，都可以呼叫這個：

```javascript
// Execute Workflow Node 傳入
{
  "email": "{{ $json.email }}",
  "name": "{{ $json.name }}",
  "rules": {
    "email": { "required": true, "type": "email" },
    "name": { "required": true, "minLength": 2 }
  }
}
```

拿到結果後檢查 `{{ $json.valid }}`，如果是 false 就停止流程。

## 錯誤處理模組

建立一個統一的錯誤處理 workflow：

**Workflow: Handle Error**

1. **Execute Workflow Trigger**
2. **Code Node**：記錄錯誤到資料庫
3. **Slack Node**：發送錯誤通知給開發團隊
4. **HTTP Request Node**：呼叫監控系統 API

在其他 workflow 的 Error Trigger：

```
Error Trigger → Execute Workflow (Handle Error)
```

傳入錯誤資訊：

```json
{
  "workflow": "{{ $workflow.name }}",
  "error": "{{ $json.error.message }}",
  "execution_id": "{{ $execution.id }}",
  "timestamp": "{{ $now }}"
}
```

這樣所有 workflow 的錯誤都會用同一套機制處理，不用每個都寫一次。

## 設定管理

建立一個「設定中心」workflow，存放常用的設定：

**Workflow: Get Config**

```javascript
// Code Node
const configs = {
  email: {
    from: 'noreply@company.com',
    replyTo: 'support@company.com'
  },
  slack: {
    channels: {
      alerts: '#alerts',
      sales: '#sales',
      tech: '#tech'
    }
  },
  api: {
    baseUrl: 'https://api.company.com',
    timeout: 30000
  }
};

const configKey = $input.first().json.key;
const value = configKey.split('.').reduce((obj, key) => obj[key], configs);

return [{
  json: {
    key: configKey,
    value: value
  }
}];
```

其他 workflow 要用設定時：

```
Execute Workflow (Get Config) 
傳入: { "key": "slack.channels.alerts" }
拿到: { "value": "#alerts" }
```

這樣設定集中管理，要改的話只要改一個地方。

## 遞迴呼叫

Sub-workflow 也可以呼叫自己（遞迴），但要小心避免無限迴圈。

比如處理階層式的資料結構：

```javascript
// Workflow: Process Tree Node
const node = $input.first().json;
const results = [];

// 處理當前節點
results.push(processNode(node));

// 如果有子節點，遞迴處理
if (node.children && node.children.length > 0) {
  for (const child of node.children) {
    // Execute Workflow (自己)
    const childResult = executeWorkflow('Process Tree Node', child);
    results.push(...childResult);
  }
}

return results;
```

記得要設定終止條件，不然會一直跑下去。

## 版本管理

Sub-workflow 被多個 workflow 使用，修改時要特別小心。

**建議做法**：

1. 測試環境先測試修改
2. 用版本號命名：`Send Notification v2`
3. 新版本穩定後，再讓其他 workflow 切換過去
4. 保留舊版本一段時間，確認沒問題再刪除

或者在 workflow 名稱加上 `[Stable]`、`[Beta]` 這種標籤。

## 效能考量

Sub-workflow 的執行會有一點 overhead，因為要啟動新的 execution context。

如果是很簡單的邏輯（幾個 node），直接寫在主 workflow 可能更快。

Sub-workflow 適合用在：
- 邏輯複雜，有 10 個以上的 node
- 被多個 workflow 重複使用
- 需要獨立測試和維護

## 依賴管理

當 workflow 之間有依賴關係，要注意：

**避免循環依賴**：A 呼叫 B，B 又呼叫 A，會造成無限迴圈。

**文件化依賴**：在 workflow 的 Notes 寫清楚依賴哪些 sub-workflow。

**測試完整性**：修改 sub-workflow 後，要測試所有呼叫它的 workflow。

## 實務經驗

模組化設計讓 workflow 維護容易很多。幾個建議：

**單一職責**：一個 sub-workflow 只做一件事，不要包太多邏輯。

**命名清楚**：用描述性的名稱，一看就知道做什麼。`Send Email` 比 `Util1` 好。

**參數驗證**：sub-workflow 開頭要驗證傳入的參數是否正確。

**錯誤處理**：sub-workflow 要處理好錯誤，不要讓錯誤傳播到呼叫方。

**文件註解**：在 workflow 加上 Sticky Note，說明用途、參數、回傳值。

用了 sub-workflow 後，發現維護效率提升不少。以前要改一個邏輯，要去十幾個 workflow 改；現在只要改一個 sub-workflow 就好。

下週想研究 n8n 的進階功能，像是 Custom Node 開發、Webhook 進階應用、與 CI/CD 整合等。
