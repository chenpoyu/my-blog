---
layout: post
title: "n8n 實戰案例分享"
date: 2024-05-27 14:50:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Use Cases, 實戰經驗]
---

這週把 n8n 實際應用到幾個業務場景，解決了一些長期困擾的重複性工作。記錄一下這些實戰案例，也許對其他人有參考價值。

## 案例一：自動化客戶問卷回覆處理

**背景**：

每次專案結束後，會發問卷給客戶收集意見。問卷用 Google Forms，回覆會自動記錄到 Google Sheets。以前要人工每天檢查 Sheets，看有沒有新的回覆，然後分類、轉寄給相關部門。

**自動化流程**：

1. **Trigger**：Google Sheets Trigger
   - 監聽特定 Sheet 的新增列
   - 設定為 `On Row Added`

2. **資料處理**：Code Node
   ```javascript
   const response = $input.first().json;
   const satisfaction = parseInt(response.Satisfaction);
   const category = satisfaction >= 4 ? 'positive' : 
                     satisfaction >= 2 ? 'neutral' : 'negative';
   
   return [{
     json: {
       ...response,
       category: category,
       urgent: satisfaction === 1
     }
   }];
   ```

3. **Switch Node**：根據滿意度分類
   - Case 1: `positive` → 感謝信
   - Case 2: `neutral` → 標準回覆
   - Case 3: `negative` → 主管介入

4. **Gmail Node**：自動寄感謝信給高滿意度客戶

5. **Slack Node**：低滿意度案件通知主管

6. **Notion API**：記錄到客戶管理系統

**效果**：

原本每天要花 30 分鐘處理問卷，現在全自動。而且低滿意度案件能即時通知，不會延誤處理。

## 案例二：發票資料同步

**背景**：

會計部門每天要從發票系統（政府的電子發票平台）下載資料，整理後匯入公司的 ERP 系統。手動操作容易出錯，而且浪費時間。

**自動化流程**：

1. **Schedule Trigger**：每天早上 8 點執行

2. **HTTP Request Node**：呼叫電子發票 API
   ```
   GET https://api.einvoice.nat.gov.tw/...
   Headers: API-Key, Date
   ```

3. **Code Node**：解析 XML 回應，轉成 JSON
   ```javascript
   const xml2js = require('xml2js');
   const parser = new xml2js.Parser();
   
   const xml = $input.first().binary.data.toString();
   const result = await parser.parseStringPromise(xml);
   
   const invoices = result.Invoices.Invoice.map(inv => ({
     invoiceNumber: inv.Number[0],
     date: inv.Date[0],
     amount: parseFloat(inv.Amount[0]),
     taxId: inv.BuyerTaxId[0],
     items: inv.Items[0].Item
   }));
   
   return invoices.map(inv => ({ json: inv }));
   ```

4. **資料驗證**：IF Node 檢查必填欄位
   ```javascript
   {{ $json.invoiceNumber && $json.amount > 0 }}
   ```

5. **MySQL Node**：寫入 ERP 資料庫
   ```sql
   INSERT INTO invoices (invoice_number, date, amount, tax_id)
   VALUES (?, ?, ?, ?)
   ON DUPLICATE KEY UPDATE amount = VALUES(amount);
   ```

6. **Email Node**：寄送每日摘要給會計
   - 成功匯入幾筆
   - 有錯誤的發票列表

**遇到的問題**：

一開始 XML 解析一直出錯，因為政府 API 的編碼是 Big5，要先轉成 UTF-8：

```javascript
const iconv = require('iconv-lite');
const buffer = Buffer.from($input.first().binary.data);
const xml = iconv.decode(buffer, 'big5');
```

還有 API 有 rate limit，每分鐘只能呼叫 10 次。加上 retry 機制和延遲。

**效果**：

會計同仁從每天花 2 小時處理發票，減少到只要 10 分鐘檢查報表。錯誤率也降低了。

## 案例三：社群媒體內容排程發布

**背景**：

行銷部門每週要在 Facebook、Instagram、LinkedIn 發布內容。內容都先整理在 Notion，然後手動去各平台發布，很費時。

**自動化流程**：

1. **Notion Trigger**：監聽內容日曆
   - 篩選條件：Status = "Ready to Post"

2. **Code Node**：整理貼文內容
   ```javascript
   const post = $input.first().json;
   
   return [{
     json: {
       content: post.properties.Content.rich_text[0].plain_text,
       imageUrl: post.properties.Image.files[0].file.url,
       platforms: post.properties.Platforms.multi_select.map(p => p.name),
       scheduledTime: post.properties.ScheduledTime.date.start
     }
   }];
   ```

3. **Wait Node**：等到排程時間
   ```javascript
   {{ $json.scheduledTime }}
   ```

4. **Switch Node**：根據平台分流

5. **Facebook API**：發布到 FB
   ```
   POST /me/feed
   {
     "message": "{{ $json.content }}",
     "link": "{{ $json.imageUrl }}"
   }
   ```

6. **Instagram API**：發布到 IG

7. **LinkedIn API**：發布到 LinkedIn

8. **Notion API**：更新狀態為 "Published"

**挑戰**：

各平台的 API 不太一樣，Instagram 特別麻煩，要先上傳圖片拿 media_id，再發布貼文。

Facebook 和 Instagram Business API 需要 Page Access Token，要先在 Facebook Developer 設定。

**效果**：

行銷同仁只要在 Notion 準備好內容，設定發布時間，系統會自動發布到各平台。省下不少時間，而且不會忘記發文。

## 案例四：伺服器監控與自動修復

**背景**：

公司有幾台伺服器，偶爾會因為記憶體不足或某個服務掛掉而出問題。以前都是等使用者反應才知道，處理不夠及時。

**自動化流程**：

1. **Schedule Trigger**：每 5 分鐘執行

2. **HTTP Request Node**：檢查各服務是否正常
   ```
   GET https://app1.company.com/health
   GET https://app2.company.com/health
   GET https://app3.company.com/health
   ```

3. **Code Node**：檢查回應
   ```javascript
   const checks = $input.all();
   const failed = checks.filter(c => 
     c.json.statusCode !== 200 || 
     c.json.responseTime > 5000
   );
   
   return failed.map(f => ({ 
     json: {
       service: f.json.url,
       statusCode: f.json.statusCode,
       error: f.json.error
     }
   }));
   ```

4. **IF Node**：如果有服務異常

5. **SSH Node**：連到伺服器，嘗試重啟服務
   ```bash
   docker restart app1_container
   systemctl restart nginx
   ```

6. **Wait Node**：等 30 秒

7. **再次檢查**：確認服務是否恢復

8. **Slack Alert**：
   - 如果自動修復成功：通知已處理
   - 如果還是失敗：緊急通知 on-call 工程師

**安全考量**：

SSH 連線要用 key-based authentication，不要用密碼。

而且只允許執行特定指令，不要給 root 權限。

**效果**：

大部分的服務異常都能自動恢復，減少 downtime。真正需要人工介入的問題，也能即時通知。

## 案例五：定期報表自動產生

**背景**：

每週一早上要產生營運報表，寄給管理層。報表包含：
- 上週營收統計
- 新增客戶數
- 系統使用情況
- 待處理問題

以前要從多個系統撈資料，用 Excel 整理，花 1-2 小時。

**自動化流程**：

1. **Schedule Trigger**：每週一早上 7 點

2. **多個 MySQL Node**：從不同資料庫查詢
   ```sql
   -- 營收統計
   SELECT 
     DATE(created_at) as date,
     SUM(amount) as daily_revenue
   FROM orders
   WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
   GROUP BY DATE(created_at);
   
   -- 新增客戶
   SELECT COUNT(*) as new_customers
   FROM customers
   WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY);
   ```

3. **Google Analytics API**：網站流量資料

4. **Jira API**：待處理 ticket 數量

5. **Code Node**：整理成報表格式
   ```javascript
   const revenue = $('MySQL').all();
   const customers = $('MySQL1').first().json.new_customers;
   const traffic = $('Google Analytics').first().json;
   const tickets = $('Jira').first().json.total;
   
   const report = {
     period: `${lastWeekStart} - ${lastWeekEnd}`,
     totalRevenue: revenue.reduce((sum, r) => sum + r.json.daily_revenue, 0),
     avgDailyRevenue: totalRevenue / 7,
     newCustomers: customers,
     pageViews: traffic.pageviews,
     openTickets: tickets
   };
   
   return [{ json: report }];
   ```

6. **生成圖表**：用 QuickChart API 產生圖表
   ```
   https://quickchart.io/chart?c={type:'line',data:{labels:...,datasets:...}}
   ```

7. **HTML Template**：組合成 HTML email

8. **Gmail Node**：寄送報表

**進階功能**：

如果營收下降超過 10%，在信件標題加上 ⚠️ 警示。

報表同時存一份到 Google Drive，方便查閱歷史資料。

**效果**：

管理層每週一早上就能收到報表，不用等人工整理。而且格式一致，不會遺漏資料。

## 共通的經驗教訓

**錯誤處理很重要**：每個外部 API 呼叫都可能失敗，要做好 retry 和錯誤通知。

**測試很重要**：正式啟用前要充分測試，包括各種邊界情況。

**監控很重要**：要知道 workflow 有沒有正常執行，失敗率多少。

**文件化**：每個 workflow 要加註解說明用途、參數、聯絡人。不然過幾個月自己都忘了。

**漸進式部署**：不要一次改太多流程。先從簡單的開始，逐步擴展。

## 下一步

n8n 還有很多進階功能沒用到，像是：

- **Custom Nodes**：開發自己的 node
- **API 整合**：讓其他系統呼叫 n8n 的 API
- **AI 整合**：接入 OpenAI、Claude 等 AI 服務
- **更複雜的資料處理**：用 Python 或其他語言處理

不過目前這些功能已經夠用了，幫公司省下不少人力和時間。自動化真的是值得投資的方向。

經過這幾個月的學習和實踐，對 n8n 有了比較完整的理解。它確實是個很實用的工具，雖然有些限制，但對中小型企業的自動化需求來說已經很夠用了。
