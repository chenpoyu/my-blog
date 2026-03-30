---
layout: post
title: "Week 4: MCP 與 AI Agent - 讓 AI 自己做事"
date: 2025-08-25 15:40:00 +0800
categories: [AI工程, Agent]
tags: [MCP, Agent, Function Calling, Automation]
---

## 從「回答問題」到「執行任務」

前三週做的事情,AI 都是「被動」的:
- Week 1: 我下指令,它生成代碼
- Week 2: 我問問題,它回答
- Week 3: 我查資料,它搜尋文件

這週要玩不一樣的:**讓 AI 主動做事**。

舉個例子,我不想再手動執行這些事:
- 每週五產生「系統健康度報告」(查資料庫、統計數據、寫 Markdown)
- 檢查有沒有充電站連續 3 天沒 Heartbeat (掃 Log、發警告)
- 找出代碼裡所有「沒用 async 但應該用」的方法

這些事情重複、耗時,但又不值得寫成完整的自動化腳本。如果 AI 能「自己搞定」就太好了。

## MCP 是什麼?

MCP (Model Context Protocol) 簡單說就是**讓 LLM 可以呼叫外部工具**。

以前 LLM 只能「生成文字」,現在可以:
- 查詢資料庫
- 讀寫檔案
- 呼叫 API
- 執行 shell 命令

它的運作方式類似「Function Calling」:
1. 你告訴 LLM 它可以用哪些工具 (例如:query_database, read_file)
2. LLM 決定要用哪個工具
3. 系統執行工具,把結果回傳給 LLM
4. LLM 根據結果繼續推理或回答

## 第一個實驗:讓 AI 查資料庫

我想問「最近 7 天有哪些充電站 Heartbeat 異常?」,但不想自己寫 SQL。

**Step 1: 定義工具**

```python
tools = [
    {
        "name": "query_database",
        "description": "執行 SQL 查詢,回傳結果",
        "parameters": {
            "type": "object",
            "properties": {
                "sql": {
                    "type": "string",
                    "description": "要執行的 SQL 語法"
                }
            },
            "required": ["sql"]
        }
    }
]
```

**Step 2: 讓 LLM 決定要不要用工具**

```python
prompt = """
你可以使用 query_database 工具來查詢資料庫。
資料表結構:
- ChargingStations (Id, Name, ProviderId, Status)
- HeartbeatLogs (Id, StationId, Timestamp, IsSuccess)

問題: 最近 7 天有哪些充電站 Heartbeat 異常?
"""

response = ollama.generate(
    model='deepseek-coder-v2:16b',
    prompt=prompt,
    tools=tools
)
```

**Step 3: 執行工具**

LLM 回應:
```json
{
  "tool_call": {
    "name": "query_database",
    "arguments": {
      "sql": "SELECT cs.Name, COUNT(*) as FailCount FROM HeartbeatLogs hl JOIN ChargingStations cs ON hl.StationId = cs.Id WHERE hl.IsSuccess = 0 AND hl.Timestamp > DATEADD(day, -7, GETDATE()) GROUP BY cs.Name"
    }
  }
}
```

它自己寫了 SQL!我的程式執行這個查詢,拿到結果後再餵回 LLM,它就能生成人類可讀的報告:

> 最近 7 天有 3 個充電站 Heartbeat 異常:
> - Station-A01: 12 次失敗
> - Station-C05: 8 次失敗
> - Station-B03: 5 次失敗

整個過程我沒寫任何 SQL,只是「問問題」。

## 第二個實驗:自動生成系統報告

週五要產生「本週系統摘要」:
- 總充電次數
- 平均充電時長
- 異常事件數量
- Top 3 繁忙的充電站

以前要手動查資料庫、複製貼上到 Excel、算平均值、寫報告。現在直接問 AI:

```
生成本週系統摘要報告,包含:
1. 總充電次數
2. 平均充電時長
3. 異常事件數量
4. Top 3 繁忙的充電站
```

AI 的執行流程:
1. 呼叫 `query_database` 查充電記錄 → 得到 1,245 次
2. 再呼叫 `query_database` 算平均時長 → 得到 42 分鐘
3. 再呼叫 `query_database` 查異常事件 → 得到 23 件
4. 再呼叫 `query_database` 找 Top 3 → 得到列表
5. 把結果組成 Markdown 報告

最後生成的報告:

```markdown
# 系統週報 (2025-08-18 ~ 2025-08-24)

## 關鍵指標
- 總充電次數: 1,245 次
- 平均充電時長: 42 分鐘
- 異常事件: 23 件 (較上週減少 12%)

## 最繁忙充電站
1. Station-A01: 156 次充電
2. Station-B08: 142 次充電
3. Station-C03: 138 次充電

## 建議
異常事件主要集中在 ProviderB 的充電站,建議檢查其 API 穩定性。
```

全自動,我只要複製貼上給 PM 就好。

## 第三個實驗:代碼審查 Agent

我給它一個任務:「找出專案中所有應該用 async 但沒用的方法」。

AI 的執行流程:
1. 呼叫 `list_files("*.cs")` 取得所有 C# 檔案
2. 對每個檔案呼叫 `read_file` 讀內容
3. 分析哪些方法有 I/O 操作但沒用 async
4. 生成報告

結果它找到 7 個方法:

```markdown
# Async/Await 改善建議

1. **OrderService.cs - GetOrderById()**
   - 問題: 呼叫了 DbContext.Find() 但沒用 async
   - 建議: 改用 FindAsync()

2. **PaymentService.cs - ProcessPayment()**
   - 問題: 呼叫外部 API 但沒用 async
   - 建議: 改用 HttpClient.PostAsync()

...
```

這種「掃全專案找問題」的任務,以前要人工做很痛苦,現在 AI 5 分鐘搞定。

## Function Calling 的實作細節

我用的是 Ollama + 自己寫的簡單框架。核心邏輯:

```python
def run_agent(task):
    messages = [{"role": "user", "content": task}]
    
    while True:
        # LLM 決定下一步
        response = ollama.chat(
            model='deepseek-coder-v2:16b',
            messages=messages,
            tools=available_tools
        )
        
        # 如果不需要呼叫工具,就結束
        if not response.get('tool_calls'):
            return response['message']['content']
        
        # 執行工具
        for tool_call in response['tool_calls']:
            result = execute_tool(tool_call)
            messages.append({
                "role": "tool",
                "content": result
            })
```

`available_tools` 包含:
- `query_database(sql)`: 執行 SQL
- `read_file(path)`: 讀檔案
- `list_files(pattern)`: 列出檔案
- `run_command(cmd)`: 執行 shell 命令

理論上可以加無限多工具,但要小心安全性 (不能讓 AI 隨便執行危險命令)。

## 遇到的挑戰與限制

**挑戰 1: AI 會犯錯**

有次我讓它「刪除 7 天前的 Log」,它生成的 SQL 是:
```sql
DELETE FROM Logs WHERE Timestamp < GETDATE() - 7
```

看起來對,但這會刪掉「7 天前那一天」的 Log,不是「超過 7 天」的。正確應該是 `< DATEADD(day, -7, GETDATE())`。

**解法**: 重要操作一定要人工確認。我加了一個「預覽模式」,AI 產生 SQL 後先給我看,我按確認才真的執行。

**挑戰 2: Context 會爆掉**

如果任務很複雜,AI 可能呼叫 10+ 次工具。每次的輸入輸出都要放進 Context,最後會超過 16K tokens 的限制。

**解法**: 只保留最近 3 次的工具呼叫記錄,舊的丟掉。或是用更大 Context 的模型 (但會變慢)。

**挑戰 3: 執行速度慢**

一個「生成報告」的任務,AI 要呼叫 4 次資料庫。每次都要:
1. LLM 推理 (2 秒)
2. 執行工具 (0.5 秒)
3. 回傳結果給 LLM (2 秒)

總共 4 * 4.5 = 18 秒。如果用傳統腳本,1 秒就跑完。

**權衡**: 如果是「常跑的任務」,還是寫成腳本比較好。AI Agent 適合「臨時的、一次性的、邏輯會變的」任務。

## 安全性考量

讓 AI 執行 SQL、讀檔案、跑命令,風險很大。我加了這些限制:

1. **唯讀權限**: 預設只能 SELECT,不能 INSERT/UPDATE/DELETE
2. **白名單**: 只能讀 `docs/` 和 `src/` 資料夾,不能讀系統檔案
3. **命令限制**: 不能執行 `rm`、`sudo` 等危險命令
4. **人工確認**: 寫入操作要人工批准

這樣至少不會把系統搞壞。但還是建議在**測試環境**先玩,穩定後再用在 production。

## 團隊的反應

週五我 demo 了「AI 自動生成週報」,大家反應兩極:

**正面**:
- PM: 「這樣我不用每週催你交報告了」
- 資深工程師: 「可以拿來做代碼品質檢查,自動 review」

**負面**:
- DBA: 「你確定要讓 AI 直接查資料庫?萬一它跑了個超慢的查詢怎麼辦?」
- Junior 開發者: 「這樣我們是不是快失業了...」

我的看法:**AI Agent 是工具,不是取代人**。它適合做「重複但邏輯簡單」的任務,複雜的決策還是要人來做。

## 四週回顧與心得

從 Week 1 到 Week 4,我從「完全不懂 AI 開發」到「建了一套能自動執行任務的 AI Agent」。

**技術收穫**:
- **Vibe Coding** (Week 1): 學會用 AI 快速產生代碼
- **Local LLM** (Week 2): 解決隱私和成本問題
- **RAG** (Week 3): 讓 AI 讀懂專案文件
- **Agent** (Week 4): 讓 AI 自動執行任務

**思維轉變**:
- AI 不是「會寫代碼的工具」,而是「可以思考的助手」
- 重點不是「AI 能不能做」,而是「哪些任務適合 AI 做」
- 人的價值從「執行任務」變成「定義任務、審查結果」

技術主管的角色正在改變。以前是「寫代碼最強的人」,未來是「最會用 AI 的人」。
