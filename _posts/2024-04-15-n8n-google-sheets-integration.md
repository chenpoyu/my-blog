---
layout: post
title: "用 n8n 整合 Google Sheets 做資料處理"
date: 2024-04-15 10:20:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Google Sheets, 資料處理]
---

上週把 n8n 跑起來了，這週來試試整合 Google Sheets。很多企業還是習慣用 Excel 或 Google Sheets 管理資料，如果能自動化處理這些資料，會方便很多。

## 實際需求

想做的事情是這樣：每天自動從 Google Sheets 讀取訂單資料，檢查庫存，如果庫存不足就發 email 通知採購。

原本這件事要人工每天打開 Sheets 檢查，很容易忘記。用 n8n 自動化後，設定好就不用管了。

## 設定 Google Sheets 認證

要讓 n8n 存取 Google Sheets，首先要設定認證。

1. 進入 Google Cloud Console（https://console.cloud.google.com）
2. 建立一個新專案（或用既有的）
3. 啟用 Google Sheets API 和 Google Drive API
4. 建立 OAuth 2.0 認證資訊
5. 設定 Authorized redirect URIs：`http://localhost:5678/rest/oauth2-credential/callback`
6. 下載 credentials（Client ID 和 Client Secret）

回到 n8n：

1. 點 Credentials → New
2. 選 Google Sheets OAuth2 API
3. 填入 Client ID 和 Client Secret
4. 點 Connect my account
5. 會跳到 Google 授權頁面，同意授權
6. 完成後會自動跳回 n8n

這樣就設定好了，之後 Google Sheets Node 可以選擇這個 credential。

## 讀取 Google Sheets 資料

建立一個新的 Workflow：

**Step 1: Schedule Trigger**

用 Schedule Trigger Node 設定每天早上 9 點執行：
- Trigger Interval: Days
- Days Between Triggers: 1
- Hour: 9
- Minute: 0

**Step 2: Google Sheets Node**

從 Sheets 讀取訂單資料：
- Operation: Get Many
- Document: 選擇你的 Google Sheets 檔案
- Sheet: Sheet1
- Range: A2:D100（假設標題在第一行）

執行後會拿到一個陣列，每筆資料包含訂單編號、產品、數量等。

## 處理資料

拿到資料後，要檢查庫存。假設庫存資料在另一個 Sheet。

**Step 3: 再加一個 Google Sheets Node**

讀取庫存資料：
- Operation: Get Many
- Document: 同一個檔案
- Sheet: Stock
- Range: A2:C50

現在有兩組資料：訂單和庫存。要比對哪些產品庫存不足。

**Step 4: Code Node**

用 Code Node 寫邏輯：

```javascript
// 從前面的 Google Sheets Node 拿資料
const orders = $('Google Sheets').all();  // 訂單資料
const stock = $('Google Sheets1').all();  // 庫存資料

// 建立庫存對照表
const stockMap = {};
for (const item of stock) {
  stockMap[item.json.ProductName] = parseInt(item.json.Stock);
}

// 檢查哪些訂單的產品庫存不足
const lowStockItems = [];
for (const order of orders) {
  const product = order.json.Product;
  const quantity = parseInt(order.json.Quantity);
  const currentStock = stockMap[product] || 0;
  
  if (currentStock < quantity) {
    lowStockItems.push({
      orderNumber: order.json.OrderNumber,
      product: product,
      ordered: quantity,
      available: currentStock,
      shortage: quantity - currentStock
    });
  }
}

return lowStockItems.map(item => ({ json: item }));
```

這段 code 會比對訂單和庫存，找出缺貨的項目。

## 發送通知

如果有缺貨，發 email 通知採購。

**Step 5: IF Node**

先判斷是否有缺貨：
- Condition: `{{ $json.length > 0 }}`

如果有資料（缺貨），走 True 分支；沒有就結束。

**Step 6: Gmail Node**

在 True 分支加上 Gmail Node：
- Operation: Send
- To: purchasing@example.com
- Subject: 庫存不足警示 - {{ $now.format('YYYY-MM-DD') }}
- Message: 寫一個表格列出缺貨項目

Message 可以用 HTML 格式：

{% raw %}
```html
<h2>以下產品庫存不足</h2>
<table border="1" cellpadding="5">
  <tr>
    <th>訂單編號</th>
    <th>產品</th>
    <th>訂購數量</th>
    <th>可用庫存</th>
    <th>缺少數量</th>
  </tr>
  {{ $json.items.map(item => `
  <tr>
    <td>${item.orderNumber}</td>
    <td>${item.product}</td>
    <td>${item.ordered}</td>
    <td>${item.available}</td>
    <td>${item.shortage}</td>
  </tr>
  `).join('') }}
</table>
```
{% endraw %}

不過 n8n 的 expression 語法有點複雜，實務上可能要在 Code Node 先把 HTML 組好，再傳給 Gmail Node。

## 寫回 Google Sheets

除了發 email，也可以把結果寫回 Google Sheets，留個紀錄。

**Step 7: Google Sheets Node**

在 email 之後加一個 Google Sheets Node：
- Operation: Append
- Document: 同一個檔案
- Sheet: Alerts（另外建一個 sheet 記錄告警）
- Columns: Date, Product, Shortage

這樣每次檢查的結果都會記錄下來，方便追蹤。

## 測試執行

設定好後，點 Execute Workflow 手動測試一次。

可以在每個 Node 上看到執行結果，確認資料有正確傳遞。如果哪個 Node 出錯，會顯示錯誤訊息。

常見的問題：
- **權限不足**：Google Sheets API 的權限範圍要設對
- **資料格式**：Sheets 裡的資料格式要一致，不然解析會出錯
- **Range 設定**：A2:D100 要根據實際資料調整

測試沒問題後，點 Active 啟用，之後就會每天自動執行。

## 進階應用

**處理多個 Sheets**：可以用 Loop 處理多個檔案，但要注意 Google API 的 rate limit。

**資料驗證**：在讀取資料後，先檢查格式是否正確，避免後續處理出錯。

**錯誤處理**：用 Error Trigger 捕捉錯誤，發通知給管理員。

**資料轉換**：n8n 有 Item Lists Node，可以做一些陣列操作，像 filter、map、sort。

## 實際心得

Google Sheets 整合很實用，很多企業流程都能用這個自動化。不過要注意：

**API Quota**：Google Sheets API 有使用限制，每天每個使用者 500 次請求。如果頻繁存取，可能會超過。

**效能問題**：如果 Sheet 資料很多（幾千筆），讀取會很慢。可以考慮只讀需要的 range，或者改用資料庫。

**權限管理**：OAuth token 要妥善保管，不要外洩。n8n 會加密存放 credential，但伺服器本身的安全要做好。

**協作問題**：多人同時編輯 Sheets，可能會有資料衝突。要設計好流程，避免覆蓋資料。

下週想研究怎麼整合資料庫，看能不能把 Sheets 的資料同步到 MySQL 或 PostgreSQL，這樣查詢和報表會更方便。
