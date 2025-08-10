---
layout: post
title: "Week 2: Local LLM 實驗 - Ollama 與 DeepSeek"
date: 2025-08-11
categories: [AI工程, Local LLM]
tags: [Ollama, DeepSeek, Quantization, Continue.dev]
---

## 為什麼要搞 Local LLM

上週用 GitHub Copilot (Claude Sonnet 4.5) 用得很爽,但想到帳單就笑不出來了。一週我就要花掉 50% ，這樣一個月絕對不夠用。

更重要的是**隱私問題**。我們的充電樁系統有些業者的商業邏輯很敏感,不適合傳到 Claude 的伺服器。法務那邊也一直在問「代碼會不會被拿去訓練模型?」

所以這週的目標很明確:把 AI 搬到本地跑。

## 安裝 Ollama

看了一些教學,Ollama 似乎是最簡單的方案。官網下載安裝包,裝完之後就可以用命令列下載模型。

```bash
ollama pull deepseek-coder-v2:16b
```

等了大概 20 分鐘,模型下載完成。檔案大小 9.8GB,比我想像的小 (本來以為會破 20GB)。

第一次跑起來:

```bash
ollama run deepseek-coder-v2:16b
```

速度出乎意料的快。我的 Mac Mini Pro 是 M4 Pro 晶片 (16 核心) + 48GB RAM,生成代碼的速度大概每秒 35-40 tokens。雖然比不上雲端 API (Claude 大概 50+ tokens/s),但差距沒想像中大。

## 第一個測試:重寫一個簡單的 Service

我丟了一段現有的代碼給它:

```
重構這個 HeartbeatService:
- 改用 async/await
- 加上錯誤處理
- 用 ILogger 記錄
```

DeepSeek 生成的代碼...還行。基本邏輯對的,但有幾個問題:

1. 它把 `async` 方法命名為 `ProcessHeartbeat()` 而不是 `ProcessHeartbeatAsync()`,不符合 .NET 慣例
2. Exception 處理太粗糙,全部 catch 成 `Exception` (應該針對不同例外做處理)
3. Logger 的訊息格式跟我們團隊的風格不太一樣

修改後可以用,但**不如 Claude 生成的那麼「懂」我們的專案**。

## Quantization 的差異測試

DeepSeek 有好幾個量化版本。我下載了兩個來比較:

- `deepseek-coder-v2:16b-lite-instruct-q4_0` (4-bit 量化,檔案 6GB)
- `deepseek-coder-v2:16b-lite-instruct-q8_0` (8-bit 量化,檔案 10GB)

測試同一個任務:「寫一個 OCPI Location 的 CRUD Repository」

**Q4 版本**:
- 速度快 (45 tokens/s)
- 記憶體佔用 8GB
- 但生成的代碼有錯誤:把 `Task<Location>` 寫成 `Task<IEnumerable<Location>>`,型別搞混了

**Q8 版本**:
- 速度慢一點 (38 tokens/s)
- 記憶體佔用 15GB
- 代碼品質好很多,型別都對,連 using 的命名空間都正確

**結論**: 有 48GB RAM 的好處是可以無痛跑 Q8,完全不用擔心記憶體不夠。Q8 比較適合寫代碼,Q4 可能適合做簡單的 code review 或文件生成。

## 串接 VS Code:Continue.dev

Ollama 預設只能在 terminal 用,要接到 IDE 得靠 Continue.dev 這個插件。

裝好後在 `~/.continue/config.json` 加上:

```json
{
  "models": [
    {
      "title": "DeepSeek Local",
      "provider": "ollama",
      "model": "deepseek-coder-v2:16b"
    }
  ]
}
```

重啟 VS Code,按 `Cmd+I` 就可以用了。

測試了幾個場景:

**場景 1: 代碼補全** - 速度可以接受,但建議的代碼比較「制式」,不像 Claude 那麼能理解上下文。

**場景 2: 解釋代碼** - 我選了一段複雜的 LINQ 查詢,問它「這段在做什麼」。解釋得滿清楚的,這個場景 Local LLM 完全夠用。

**場景 3: 重構建議** - 表現普通。它建議把一個 100 行的方法拆成 3 個小方法,方向對,但拆的位置不太理想 (拆斷了商業邏輯)。

## 實戰測試:完全離線寫一個新功能

週五下午我故意把 Wi-Fi 關掉,挑戰用 Local LLM 寫一個新的 API:「查詢充電站的歷史使用記錄」。

需求:
- GET /api/stations/{id}/usage-history
- 支援日期範圍篩選
- 回傳每小時的充電次數和總電量

**過程**:

1. 我用 Continue.dev 問:「用 EF Core 寫一個查詢,統計充電站的每小時使用量」
2. 它生成了基本的 LINQ 查詢,但忘記處理時區問題 (我們的資料是 UTC,要轉成當地時間)
3. 我補充要求:「加上時區轉換,用 TimeZoneInfo」
4. 它重新生成,這次對了
5. 我又問:「加上快取,用 IMemoryCache」
6. 它加上快取邏輯,但 cache key 的命名很隨意

來回大概 6 次對話,花了 40 分鐘完成。比用 Copilot (Claude Sonnet 4.5) 慢一點,但**在沒網路的情況下還能寫代碼,這件事本身就很酷**。

## Local vs Cloud 的對比

用了一週 Local LLM,我的心得:

| 項目 | Cloud (Claude) | Local (DeepSeek) |
|------|----------------|------------------|
| **速度** | 很快 (50+ tokens/s) | 不錯 (35-40 tokens/s) |
| **品質** | 很懂專案架構 | 需要更多提示 |
| **成本** | $150/月/人 | 免費 (電費忽略) |
| **隱私** | 代碼上雲端 | 完全本地 |
| **Context** | 200K tokens | 依模型,通常較小 |

**適合 Local 的場景**:
- 簡單的重構、代碼補全
- 閱讀代碼、寫註解
- 生成測試案例
- 敏感專案的開發

**適合 Cloud 的場景**:
- 複雜的架構設計
- 需要深度理解 codebase 的重構
- 時間緊迫的任務

## 遇到的技術問題

**問題 1: 記憶體充足的優勢**

跑 Q8 的 16B 模型時,記憶體會吃到 15GB 左右。還好 Mac Mini Pro 有 48GB RAM,同時開 Chrome + VS Code + Docker 都完全沒壓力。Activity Monitor 顯示還有 20GB+ 可用記憶體。

**心得**: 這時候才感受到當初加購 RAM 的價值。如果只有 16GB,跑 Local LLM 會很吃力。48GB 讓我可以無痛使用 Q8 甚至測試更大的模型。

**問題 2: 模型不懂我們的專案慣例**

Claude 用久了會「學會」我們的 coding style (因為有 codebase indexing),但 Local LLM 每次都是全新對話,沒有記憶。

**解法**: 寫一個 `project-guidelines.md` 放在專案根目錄,每次對話先丟給它:
```
請遵循這些規範:
- async 方法後面加 Async
- Exception 要細分類型處理
- Logger 格式: [Service] Message (UserId, RequestId)
```

效果有改善,但還是沒有 Cloud 那麼自然。

**問題 3: Context Window 太小**

DeepSeek 的 context 只有 16K tokens,大一點的檔案就塞不下。我試著丟一個 500 行的 Service 給它分析,結果它只處理了前半部。

**解法**: 把大檔案拆成小段,分批處理。或是等以後換更大的模型 (聽說 33B 的版本 context 更大,以我的 Mac 規格應該跑得動,下次可以試試)。

## 團隊導入的考量

週五跟幾個 Senior 工程師聊了一下,問他們要不要試 Local LLM。

**支持的理由**:
- 隱私安全,法務比較放心
- 不用擔心 API 費用爆表
- 可以客製化 (之後可以微調模型)

**擔心的點**:
- 品質不如 Claude,會不會浪費時間?
- 每個人的電腦規格不同,有些人的 MacBook Pro 只有 16GB RAM 可能跑不動
- 學習成本 (要會裝 Ollama、設定 Continue)

我的建議是:**混合使用**。簡單任務用 Local,複雜任務用 Cloud。這樣既能控制成本,又能保持效率。

## 下週計畫

**目標: RAG - 讓 AI 讀懂專案文件**

這週發現 Local LLM 不懂我們專案的 domain knowledge。下週要研究 RAG (Retrieval-Augmented Generation),讓 AI 能查詢:
- 專案的技術文件
- 過去的 PR 討論
- OCPI 規範文件

技術上要搞定:
1. 文件切分 (Chunking)
2. Embedding (把文字轉成向量)
3. 向量資料庫 (ChromaDB 或 pgvector)
4. 查詢整合 (讓 LLM 能自動去查相關文件)

如果成功,Local LLM + RAG 的組合應該能接近 Cloud LLM 的效果。
