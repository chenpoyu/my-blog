---
layout: post
title: "n8n 初探：工作流程自動化工具"
date: 2024-04-08 14:35:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Workflow, Automation]
---

最近公司內部有很多重複性的工作，像是每天要從不同系統撈資料、整理後寄信給相關人員、定期備份資料等等。這些事情雖然不難，但很瑣碎，而且容易出錯。

研究了一下工作流程自動化工具，發現 n8n 這個開源專案蠻有意思的。它是用視覺化的方式設計工作流程，不用寫太多程式碼就能把各種系統串接起來。

## 什麼是 n8n

n8n（念作 "nodemation"）是一個開源的工作流程自動化平台。概念類似 Zapier 或 Make（以前叫 Integromat），但因為是開源的，可以自己架設，資料不用放到第三方平台。

基本概念是這樣：設計一個 Workflow，裡面有很多 Node，每個 Node 執行一個動作（抓資料、處理資料、發送通知等），Node 之間可以傳遞資料，形成一個自動化流程。

比如說：
1. 定時觸發（每天早上 9 點）
2. 從 Google Sheets 讀取資料
3. 呼叫 API 取得額外資訊
4. 整理成報表
5. 寄信給相關人員

以前要寫一支程式才能做到，現在用拖拉的方式就能完成。

## 為什麼選 n8n

市面上類似的工具不少，為什麼選 n8n？

**開源自架**：資料在自己的伺服器上，不用擔心隱私問題。企業的敏感資料不適合放到第三方平台。

**整合豐富**：支援超過 400 個整合（Node），包括常見的 Google、Microsoft、Slack、Database、HTTP API 等。

**自訂彈性**：內建 Code Node，可以寫 JavaScript 處理複雜邏輯。如果內建 Node 不夠用，還能自己開發。

**價格優勢**：如果用 Zapier，每個月的費用會隨著 task 數量增加。n8n 自架的話，只有伺服器成本。

**社群活躍**：GitHub 上有 3 萬多顆星，文件完整，遇到問題容易找到解答。

## 快速安裝

n8n 提供 Docker 映像檔，測試環境用 Docker 跑最快：

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

等個幾秒鐘，開瀏覽器連 `http://localhost:5678`，就能看到 n8n 的介面。

第一次進入會要求設定管理員帳號密碼，設定完就能開始用了。

## 第一個 Workflow

來做個簡單的測試。建立一個新的 Workflow：

1. 點左側的 `+` 建立 Workflow
2. 從右側的 Node 面板拖一個 `Webhook` Node 到畫布上
3. Webhook Node 會產生一個 URL
4. 再拖一個 `Set` Node，用來設定回傳的資料
5. 把 Webhook 連到 Set
6. 設定 Set Node 回傳 `{ "message": "Hello from n8n" }`

點右上角的 `Save` 儲存，然後 `Activate` 啟用。

用 curl 測試：

```bash
curl http://localhost:5678/webhook-test/hello
```

應該會收到 `{"message": "Hello from n8n"}`。

簡單吧！這只是最基本的，但已經能看出 n8n 的運作方式。

## 實用的範例

來做一個實際一點的自動化：監控網站，有異常就發 Slack 通知。

**Workflow 設計**：

1. **Cron Node**：每 5 分鐘執行一次
2. **HTTP Request Node**：檢查網站是否正常
3. **IF Node**：判斷 HTTP status code
4. **Slack Node**：如果不是 200，發送告警訊息

設定 Cron Node：
- Expression: `*/5 * * * *`（每 5 分鐘）

設定 HTTP Request Node：
- Method: GET
- URL: `https://example.com`

設定 IF Node：
- Condition: `{{ $json.statusCode }} === 200`
- True 分支：不做事
- False 分支：連到 Slack Node

設定 Slack Node：
- 需要先設定 Slack credential（拿 Webhook URL 或 OAuth token）
- Message: `網站異常！Status Code: {{ $json.statusCode }}`

啟用後，n8n 會每 5 分鐘自動檢查，有問題就通知。

## 一些觀察

n8n 的介面很直覺，拖拉 Node、連線、設定參數，邏輯清楚。不過要熟悉各個 Node 的參數和資料格式，需要一些時間。

Node 之間傳遞的資料是 JSON 格式，用 `{{ }}` 這種 expression 語法來存取。比如前一個 Node 的輸出是 `{ "name": "John" }`，下一個 Node 可以用 `{{ $json.name }}` 取得。

有些 Node 會回傳陣列，要用 `{{ $json.items[0].value }}` 這樣存取。一開始可能會搞混，多試幾次就習慣了。

內建的 Node 已經很夠用，但如果要做比較複雜的資料處理，還是要用 Code Node 寫 JavaScript。好在 JavaScript 蠻好寫的，而且可以用 Node.js 的 library。

## 下一步

這週只是把 n8n 跑起來，做了一些簡單測試。下週想研究怎麼整合常用的系統，像是 Google Sheets、Gmail、資料庫等，看看能不能把一些日常工作自動化。

另外也想了解 n8n 的部署方式，測試環境用 Docker 很方便，但正式環境可能要考慮高可用、備份、監控等問題。

目前看起來 n8n 是個很實用的工具，如果能善用，應該可以省下不少時間。
