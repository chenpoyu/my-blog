---
layout: post
title: "n8n 資料庫整合與 ETL 流程"
date: 2024-04-22 16:45:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Database, ETL]
---

這週研究了 n8n 的資料庫整合功能。上週用 Google Sheets 做資料處理，但 Sheets 有些限制：資料量大了會慢、不好做複雜查詢、多人協作容易衝突。

如果能把資料同步到資料庫，就能解決這些問題。n8n 支援 MySQL、PostgreSQL、MongoDB 等常見資料庫，剛好可以拿來做 ETL（Extract, Transform, Load）。

## 實際場景

有個實際需求：每天從多個系統匯出資料，整合到一個資料倉儲，供後續分析使用。

資料來源包括：
- Google Sheets（業務資料）
- REST API（訂單系統）
- MySQL（舊系統的資料庫）

目標是把這些資料統一到 PostgreSQL，建立一個整合的資料表。

## 設定資料庫連線

首先在 n8n 設定 PostgreSQL 的 credential：

1. Credentials → New
2. 選 Postgres
3. 填入連線資訊：
   - Host: localhost（或實際的 DB server）
   - Database: analytics
   - User: n8n_user
   - Password: ****
   - Port: 5432

測試連線，確認能正常連接。

對於 MySQL 也是類似的設定，只是選 MySQL credential。

## 建立目標資料表

在 PostgreSQL 建立一個資料表來存放整合後的資料：

```sql
CREATE TABLE daily_sales (
    id SERIAL PRIMARY KEY,
    order_date DATE NOT NULL,
    order_id VARCHAR(50) UNIQUE NOT NULL,
    customer_name VARCHAR(100),
    product VARCHAR(100),
    quantity INTEGER,
    amount DECIMAL(10, 2),
    source VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

這個表會整合來自不同來源的訂單資料。

## ETL Workflow 設計

建立一個 Workflow，每天凌晨 2 點執行：

**Step 1: Schedule Trigger**

設定為每天 2:00 AM 執行：
- Mode: Days
- Hour: 2
- Minute: 0

**Step 2: 從 Google Sheets 抓資料**

用 Google Sheets Node：
- Operation: Get Many
- Range: A2:F1000

**Step 3: 資料轉換**

用 Code Node 把 Sheets 的資料轉成統一格式：

```javascript
const items = $input.all();
const transformed = items.map(item => {
  const data = item.json;
  return {
    json: {
      order_date: data.Date,
      order_id: `SHEET-${data.OrderID}`,
      customer_name: data.Customer,
      product: data.Product,
      quantity: parseInt(data.Quantity),
      amount: parseFloat(data.Amount),
      source: 'google_sheets'
    }
  };
});

return transformed;
```

**Step 4: 從 REST API 抓資料**

用 HTTP Request Node：
- Method: GET
- URL: https://api.example.com/orders
- Authentication: Bearer Token
- Headers: Authorization

API 回傳 JSON 格式的訂單資料。

**Step 5: 轉換 API 資料**

一樣用 Code Node 轉成統一格式：

```javascript
const items = $input.all();
const transformed = items[0].json.orders.map(order => {
  return {
    json: {
      order_date: order.created_at.split('T')[0],
      order_id: `API-${order.id}`,
      customer_name: order.customer.name,
      product: order.items[0].product_name,
      quantity: order.items[0].quantity,
      amount: parseFloat(order.total_amount),
      source: 'api'
    }
  };
});

return transformed;
```

**Step 6: 從舊系統 MySQL 抓資料**

用 MySQL Node：
- Operation: Execute Query
- Query:
```sql
SELECT 
  order_date,
  order_id,
  customer_name,
  product,
  quantity,
  amount
FROM orders
WHERE order_date = CURDATE() - INTERVAL 1 DAY
```

**Step 7: 轉換 MySQL 資料**

```javascript
const items = $input.all();
const transformed = items.map(item => {
  return {
    json: {
      ...item.json,
      order_id: `MYSQL-${item.json.order_id}`,
      source: 'legacy_system'
    }
  };
});

return transformed;
```

## 合併資料

現在有三條分支，各自處理不同來源的資料。要把它們合併起來。

**Step 8: Merge Node**

用 Merge Node 把三個來源的資料合併：
- Mode: Append
- 連接三個資料轉換 Node

這樣會得到一個包含所有資料的陣列。

## 寫入 PostgreSQL

**Step 9: Postgres Node**

把合併後的資料寫入資料庫：
- Operation: Insert
- Table: daily_sales
- Columns: 對應到資料欄位

不過這邊有個問題：如果 order_id 重複（比如重跑 workflow），會違反 UNIQUE 約束。

更好的做法是用 `Upsert`（Insert or Update）：

```sql
INSERT INTO daily_sales (
  order_date, order_id, customer_name, 
  product, quantity, amount, source
) VALUES (
  $1, $2, $3, $4, $5, $6, $7
)
ON CONFLICT (order_id) 
DO UPDATE SET
  quantity = EXCLUDED.quantity,
  amount = EXCLUDED.amount,
  created_at = CURRENT_TIMESTAMP;
```

不過 n8n 的 Postgres Node 不直接支援 upsert，要用 Execute Query：

```javascript
// 在 Code Node 產生 SQL
const items = $input.all();
const values = items.map(item => {
  const d = item.json;
  return `('${d.order_date}', '${d.order_id}', '${d.customer_name}', 
           '${d.product}', ${d.quantity}, ${d.amount}, '${d.source}')`;
}).join(',\n');

const sql = `
INSERT INTO daily_sales (
  order_date, order_id, customer_name, 
  product, quantity, amount, source
) VALUES ${values}
ON CONFLICT (order_id) 
DO UPDATE SET
  quantity = EXCLUDED.quantity,
  amount = EXCLUDED.amount;
`;

return [{ json: { sql } }];
```

然後用 Postgres Node 執行這個 SQL。

## 錯誤處理

ETL 流程容易出錯，要做好錯誤處理。

**Step 10: Error Trigger**

新增一個 Error Trigger Workflow：
- Trigger: On workflow error
- 連接一個 Slack/Email Node 發送通知

這樣任何 Node 出錯，都會收到通知。

也可以在關鍵步驟後加上檢查：

```javascript
// 檢查是否有資料
if ($input.all().length === 0) {
  throw new Error('No data received from source');
}

// 檢查資料格式
const items = $input.all();
for (const item of items) {
  if (!item.json.order_id) {
    throw new Error(`Missing order_id in item: ${JSON.stringify(item)}`);
  }
}

return $input.all();
```

## 增量同步 vs 全量同步

目前的設計是每天全量抓資料。如果資料量大，可以改成增量同步：

只抓昨天新增或修改的資料。這需要來源系統有 `updated_at` 或類似的時間戳欄位。

可以在 n8n 用 Static Data 或外部檔案記錄上次同步的時間：

```javascript
// 讀取上次同步時間
const lastSync = $('HTTP Request').first().json.last_sync_time || '2024-01-01';

// 只查詢新資料
const query = `
SELECT * FROM orders 
WHERE updated_at > '${lastSync}'
`;

// 記錄這次同步時間
const currentTime = new Date().toISOString();
```

下次執行時就從上次的時間點開始抓。

## 資料品質檢查

ETL 的一個重要環節是資料品質檢查。可以在寫入資料庫前加上驗證：

```javascript
const items = $input.all();
const errors = [];
const valid = [];

for (const item of items) {
  const d = item.json;
  
  // 檢查必填欄位
  if (!d.order_id || !d.order_date) {
    errors.push({
      item: d,
      reason: 'Missing required fields'
    });
    continue;
  }
  
  // 檢查資料格式
  if (isNaN(d.amount) || d.amount < 0) {
    errors.push({
      item: d,
      reason: 'Invalid amount'
    });
    continue;
  }
  
  valid.push(item);
}

// 如果有錯誤，發通知
if (errors.length > 0) {
  // 可以寫到錯誤日誌表，或發送通知
  console.log('Data quality issues:', errors);
}

return valid;
```

## 效能優化

處理大量資料時，效能很重要：

**批次處理**：不要一筆一筆寫入，用 batch insert。n8n 的 Postgres Node 預設支援批次。

**平行處理**：如果有多個獨立的資料來源，可以用 Split In Batches 平行處理。

**增量更新**：只處理變動的資料，減少處理量。

**Index 優化**：在資料庫的常用查詢欄位建立索引。

## 監控與日誌

ETL 流程要有完整的監控：

**執行記錄**：n8n 會自動記錄每次執行的結果，可以在 Executions 頁面查看。

**資料量統計**：在 workflow 最後加一個 node 記錄統計資訊：

```javascript
const stats = {
  execution_time: new Date().toISOString(),
  records_processed: $input.all().length,
  sources: {
    google_sheets: $('Google Sheets').all().length,
    api: $('HTTP Request').all().length,
    mysql: $('MySQL').all().length
  }
};

// 寫入統計表
return [{ json: stats }];
```

**告警規則**：如果資料量異常（太多或太少），發送告警。

## 實務心得

用 n8n 做 ETL 比寫傳統的 ETL script 方便很多，視覺化介面讓流程一目了然。不過也有些限制：

**大資料量**：如果單次要處理幾萬筆資料，n8n 可能會記憶體不足。這時候要分批處理，或者改用專門的 ETL 工具。

**複雜轉換**：如果資料轉換邏輯很複雜，Code Node 寫起來還是有點累。可能直接寫 Python script 會更清楚。

**版本控制**：n8n 的 workflow 存在資料庫，不太好做版本控制。要的話可以定期匯出成 JSON 放到 Git。

不過對於中小型的 ETL 需求，n8n 已經很夠用了。而且整合方便，不用自己處理各種 API 和資料庫連線。

下週想研究 n8n 的 Webhook 和 API 觸發，看怎麼讓其他系統主動觸發 workflow，而不是只能用排程。
