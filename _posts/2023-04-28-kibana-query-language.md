---
layout: post
title: "Kibana 查詢語法：從入門到精通"
date: 2023-04-28 09:00:00 +0800
categories: [Observability, Logging]
tags: [Kibana, KQL, Lucene, Query Language, Search]
---

你在 Kibana 的搜尋框裡輸入什麼？

大部分人只會用簡單的關鍵字搜尋：
```
error
```

但 Kibana 其實支援兩種強大的查詢語法：
1. **KQL（Kibana Query Language）**：簡單易用，適合日常查詢
2. **Lucene Query Syntax**：功能更強大，適合複雜查詢

今天我們來系統性地學習這兩種語法。

## KQL vs Lucene：該用哪一個？

| 特性 | KQL | Lucene |
|------|-----|--------|
| 預設語法 | ✅ Kibana 7.0+ | ❌ 需手動切換 |
| 易用性 | ⭐⭐⭐⭐⭐ 簡單 | ⭐⭐⭐ 進階 |
| 自動補全 | ✅ 支援 | ❌ 不支援 |
| 正則表達式 | ❌ 不支援 | ✅ 支援 |
| 萬用字元 | ✅ 支援 | ✅ 支援 |
| 模糊搜尋 | ❌ 不支援 | ✅ 支援 |

**我的建議**：
- 日常查詢用 KQL（90% 的情況）
- 需要正則表達式時切換到 Lucene

## KQL 基礎語法

### 1. 精確匹配

```
level:ERROR
```

找出 `level` 欄位為 `ERROR` 的所有日誌。

### 2. 包含關鍵字（全文搜尋）

```
message:timeout
```

找出 `message` 欄位包含 `timeout` 的日誌。

### 3. 多個關鍵字（AND）

```
level:ERROR and service:order-service
```

找出同時符合兩個條件的日誌。

### 4. 或條件（OR）

```
level:ERROR or level:FATAL
```

找出 `level` 為 `ERROR` 或 `FATAL` 的日誌。

### 5. 否定（NOT）

```
not level:DEBUG
```

排除 `level` 為 `DEBUG` 的日誌。

### 6. 數字範圍

```
status_code >= 500
duration > 1000
bytes <= 1024
```

### 7. 存在性檢查

```
error_message:*
```

找出有 `error_message` 欄位的日誌（不管內容是什麼）。

```
not exception:*
```

找出沒有 `exception` 欄位的日誌。

### 8. 萬用字元

```
user_id:admin*
```

找出 `user_id` 以 `admin` 開頭的日誌。

```
message:*timeout*
```

找出 `message` 包含 `timeout` 的日誌（等同於 `message:timeout`）。

### 9. 群組查詢

```
(level:ERROR or level:FATAL) and service:order-service
```

## KQL 實戰範例

### 範例 1：找出所有付款失敗的訂單

```
action:"create_order" and result:"failure" and errorType:"PaymentException"
```

### 範例 2：找出慢請求（超過 3 秒）

```
endpoint:"/api/*" and duration > 3000
```

### 範例 3：找出來自中國的異常流量

```
geoip.country_name:"China" and (status_code >= 400 or is_bot:true)
```

### 範例 4：找出特定時間範圍的錯誤

在 Kibana 的時間選擇器中選擇時間範圍，然後：

```
level:ERROR and service:payment-service
```

### 範例 5：排除健康檢查請求

```
not endpoint:"/health" and not endpoint:"/readiness"
```

## Lucene 查詢語法

切換方法：點選搜尋框左側的 **KQL** 按鈕，選擇 **Lucene**。

### 1. 基本查詢（同 KQL）

```
level:ERROR
```

### 2. 萬用字元（更靈活）

```
user_id:admin?  # ? 代表單一字元
user_id:admin*  # * 代表任意字元
```

### 3. 正則表達式

```
user_id:/admin[0-9]+/
```

找出 `user_id` 為 `admin1`, `admin2`, ... 的日誌。

```
message:/timeout|timed out|time out/
```

找出 `message` 包含任一種 timeout 寫法的日誌。

### 4. 模糊搜尋（Fuzzy Search）

```
message:tiemout~1
```

允許 1 個字元的差異，所以 `timeout` 也會被找到（即使你打錯字為 `tiemout`）。

```
message:"order processing"~3
```

允許這兩個字之間有最多 3 個其他字（鄰近搜尋）。

### 5. Boosting（調整權重）

```
level:ERROR^2 OR level:WARN
```

`ERROR` 的權重是 `WARN` 的 2 倍，搜尋結果會優先顯示 `ERROR`。

### 6. 範圍查詢（更靈活）

```
status_code:[500 TO 599]  # 包含邊界
status_code:{400 TO 499}  # 不包含邊界
```

```
@timestamp:[2023-04-28T10:00:00 TO 2023-04-28T11:00:00]
```

### 7. 欄位存在性

```
_exists_:error_message  # 有這個欄位
NOT _exists_:optional_field  # 沒有這個欄位
```

## 進階查詢技巧

### 1. 組合查詢

```
(level:ERROR OR level:FATAL) AND service:order-service AND NOT endpoint:"/health"
```

### 2. 跨欄位搜尋

```
_all:timeout
```

搜尋所有欄位中包含 `timeout` 的日誌（效能較差，不建議用）。

### 3. 精確片語匹配

```
message:"connection timeout"
```

只匹配完整片語，不匹配 `timeout connection` 或 `connection to host timeout`。

### 4. 多值欄位

假設 `tags` 是陣列 `["error", "payment", "timeout"]`：

```
tags:error  # 匹配（只要陣列中有一個值符合）
tags:(error AND payment)  # 匹配（陣列中同時有這兩個值）
```

### 5. Nested 物件查詢

假設 `user` 是巢狀物件：

```json
{
  "user": {
    "name": "John",
    "age": 30
  }
}
```

查詢：

```
user.name:John AND user.age:30
```

## 常見錯誤與陷阱

### 陷阱 1：特殊字元需要跳脫

如果你的欄位值包含特殊字元（如 `+`, `-`, `=`, `&&`, `||`, `!`, `(`, `)`, `{`, `}`, `[`, `]`, `^`, `"`, `~`, `*`, `?`, `:`, `\`, `/`），需要跳脫：

```
# 錯誤
message:C++

# 正確
message:"C++"
```

### 陷阱 2：大小寫敏感性

**Keyword 類型**：大小寫敏感

```
level:error  # 找不到（如果實際是 ERROR）
level:ERROR  # 正確
```

**Text 類型**：大小寫不敏感

```
message:error  # 可以找到 "Error", "ERROR", "error"
```

### 陷阱 3：萬用字元效能問題

```
# 慢（前置萬用字元）
message:*timeout

# 快（後置萬用字元）
message:timeout*
```

**前置萬用字元**會導致 Elasticsearch 掃描整個索引，非常慢。

### 陷阱 4：數字類型的比較

如果欄位類型是 `keyword`（字串），數字比較會失效：

```json
{
  "status_code": "500"  # 字串類型
}
```

查詢：
```
status_code >= 400  # 無法正常運作（字串比較）
```

解決方案：確保欄位類型正確（在 Logstash 中用 `convert`）。

## Kibana Discover 進階技巧

### 1. 快速過濾（Filter）

在 Discover 頁面：
1. 找到任一筆日誌
2. 點選欄位值旁的 `+`（包含）或 `-`（排除）
3. 自動產生 Filter

這比手動輸入查詢快很多。

### 2. 儲存查詢

點選搜尋框右側的 **Save**，可以儲存常用查詢：

- **名稱**：`Production Errors`
- **查詢**：`level:ERROR and environment:production`

下次直接從 **Saved Queries** 選擇。

### 3. 欄位統計

點選左側的欄位名稱，會顯示：
- **Top 5 值**
- **值的分佈**

這對快速了解數據分佈很有幫助。

### 4. 建立視覺化

在 Discover 找到有用的查詢後：
1. 點選 **Visualize**
2. 選擇圖表類型（Bar chart, Pie chart, Line chart 等）
3. 加入到 Dashboard

## 查詢效能優化

### 1. 縮小時間範圍

```
# 慢（查詢 30 天）
過去 30 天的所有錯誤

# 快（查詢 1 小時）
過去 1 小時的所有錯誤
```

時間範圍越小，查詢越快。

### 2. 使用精確欄位（Keyword）

```
# 慢（Text 欄位，需要分詞）
message:"order processing failed"

# 快（Keyword 欄位，精確匹配）
errorType:"OrderProcessingException"
```

### 3. 避免前置萬用字元

```
# 慢
message:*timeout

# 快
message:timeout*
```

### 4. 限制返回的欄位

在 Discover 中，只顯示需要的欄位，不要顯示所有欄位。

### 5. 使用 Filter 而非 Query

Filter 會被快取，Query 不會。

```
# 慢（每次都重新計算）
Query: level:ERROR

# 快（結果會被快取）
Filter: level is ERROR
```

## 實戰：調查生產環境問題

### 場景：使用者回報「付款一直失敗」

**步驟 1：找出相關日誌**

```
service:payment-service and (level:ERROR or level:WARN)
```

**步驟 2：縮小時間範圍**

使用者說是「剛剛」，所以設定「過去 15 分鐘」。

**步驟 3：找出特定使用者的日誌**

```
service:payment-service and userId:"john@example.com"
```

**步驟 4：分析錯誤類型**

在左側欄位中點選 `errorType`，看看 Top 5 錯誤是什麼。

發現：`PaymentGatewayTimeoutException` 佔了 80%。

**步驟 5：確認外部 API 狀態**

```
service:payment-service and externalApi:"stripe" and duration > 30000
```

發現：Stripe API 的回應時間從平常的 500ms 暴增到 30 秒以上。

**結論**：問題在 Stripe，不是我們的系統。立即切換到備用支付服務。

---

**掌握 Kibana 查詢語法，你就能在幾秒鐘內找到關鍵資訊，而不是盲目地翻閱日誌。**
