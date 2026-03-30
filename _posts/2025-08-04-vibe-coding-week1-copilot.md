---
layout: post
title: "Week 1: GitHub Copilot 初體驗與充電樁系統重構"
date: 2025-08-04 11:00:00 +0800
categories: [AI工程, Vibe Coding]
tags: [GitHub Copilot, Claude Sonnet 4.5, OCPI, IoT, Charging Station]
---

## 第一次真正用 GitHub Copilot 寫代碼

這週正式開始實驗 AI 輔助開發。在 VS Code 裝了 GitHub Copilot (使用 Claude Sonnet 4.5 模型),把公司的充電樁管理系統專案匯入進去,想說從小地方開始試。

沒想到第一次嘗試就讓我印象深刻。

## 實際案例:OCPI 2.1 的多業者串接邏輯

我們的系統要對接多個充電業者,每家的 API 細節都不一樣,但都宣稱遵循 OCPI 2.1 規範。實際上每家的實作都有微妙差異——有的 Location ID 格式不同、有的 Heartbeat 頻率要求不一樣、有的錯誤碼自己亂定義。

之前的做法是每接一家業者,就寫一堆 if-else 判斷。代碼越來越亂,維護成本很高。上個月 PM 說要再接兩家新業者,我就知道不能再這樣搞了,得重構成 Strategy Pattern。

**傳統做法**:我會花半天時間畫 UML、寫技術文件,然後指派給 Senior 開發者去實作。來回溝通加上開發測試,大概要 3-4 天。

**用 GitHub Copilot 的實驗**:我打開 Copilot Chat (選擇 Claude Sonnet 4.5 模型),直接描述需求:

```
重構 OCPI 業者串接邏輯,使用 Strategy Pattern:

目前系統:
- ChargingStationService 有一堆 if-else 判斷業者類型
- 每家業者的 Location、Session、Heartbeat 處理邏輯都不同

要做的:
1. 建立 IOcpiProvider 介面
2. 每家業者一個實作類別 (ProviderA, ProviderB, ProviderC)
3. 用 Factory Pattern 根據業者代碼取得對應的實作
4. ChargingStationService 改用介面,不要有任何 if-else

技術棧: .NET 8, EF Core, Dependency Injection
```

它花了大概 20 秒,生成了:
- `IOcpiProvider` 介面 (包含 SendHeartbeat, UpdateLocation, StartSession 等方法)
- 3 個業者的實作類別
- `OcpiProviderFactory` (用 Dictionary 管理實例)
- 更新了 `Program.cs` 的 DI 註冊
- 連 `ChargingStationService` 的重構都做了

我檢查了一下代碼,基本架構是對的。但有幾個地方需要調整:

**問題 1**: 它把業者的 API Key 直接寫死在代碼裡。我告訴它「API Key 要從 appsettings.json 讀取,用 IOptions<OcpiSettings>」。它立刻修正,還加上了設定檔的範本。

**問題 2**: Heartbeat 的錯誤處理太簡單,只是 log 錯誤。我說「Heartbeat 失敗要重試 3 次,間隔 5 秒,還是失敗就標記充電樁為離線」。它用 Polly 寫了 Retry Policy,連 Circuit Breaker 都加上了。

整個過程我只花了 **40 分鐘**,包含審查和調整。如果用傳統方式,光是跟開發者解釋清楚需求就要半小時。

## 意外發現:Codebase Indexing 的威力

改完之後我好奇問了一句:「哪些地方還在直接呼叫業者 API,沒走 Strategy?」

GitHub Copilot 掃了整個專案,列出 4 個地方:
- `BillingService` 還在直接呼叫 ProviderA 的 GetTariff API
- `ReportingService` 的統計邏輯有硬編碼的業者名稱
- 有個測試案例還在用舊的 if-else 邏輯
- `appsettings.Development.json` 有一組過期的 API Key

前兩個我完全忘記了,如果不是 AI 提醒,重構後肯定會出 bug。這種「語義搜尋」的能力,比 Visual Studio 的 Find All References 強太多。

## 踩到的第一個坑:安全性問題

隔天 DBA 看了代碼,發現一個嚴重問題:Copilot 生成的 SQL 查詢用了字串插值。

```csharp
var query = $"SELECT * FROM ChargingStations WHERE ProviderId = {providerId}";
```

這是典型的 SQL Injection 漏洞。我當下有點傻眼,因為代碼看起來很正常,我差點就 merge 了。

趕快回去改成參數化查詢:
```csharp
var query = "SELECT * FROM ChargingStations WHERE ProviderId = @ProviderId";
// ... 用 Dapper 的 DynamicParameters
```

這件事讓我警覺:**AI 生成的代碼不能無腦信任**。它可以提高速度,但 Code Review 絕對不能省。

後來我在專案根目錄建了一個 `.github/copilot-instructions.md` 檔案,寫了一些安全規範:

```markdown
## Security Rules
- 所有 SQL 查詢必須用參數化 (@param)
- API Key 不可寫死在代碼,要從設定檔讀取
- DateTime 一律用 UTC
- 不可在 Log 記錄敏感資料 (API Key, Token)
```

之後生成的代碼就會自動遵守這些規則。

## Prompt 的學習曲線

一開始我的 Prompt 很隨意,像「優化這段代碼」、「加個錯誤處理」。結果 AI 常常會過度發揮,把整個邏輯都改了。

後來發現**越具體越好**。比如:

❌ 不好的 Prompt:
```
優化 Heartbeat 處理邏輯
```

✅ 好的 Prompt:
```
優化 SendHeartbeat 方法:
1. 加上 Retry 機制 (失敗重試 3 次,間隔 5 秒)
2. 超過 3 次失敗,標記充電樁為 Offline 狀態
3. 記錄詳細 log (含 ProviderId, StationId, ErrorMessage)
4. 不要改變現有的商業邏輯
```

有明確的約束條件,AI 才不會亂改。

## 另一個實驗:快速產出充電站狀態監控 Dashboard API

PM 上週突然說要做一個「即時監控所有充電站狀態」的 Dashboard。需要一個 API 回傳:
- 所有充電站的即時狀態 (在線/離線/充電中/故障)
- 按業者分組統計
- 最近 1 小時的 Heartbeat 異常記錄

以前我會叫 Junior 開發者去做,來回討論加開發大概 2 天。這次我試著讓 Copilot 直接生成。

```
建立 ChargingStationMonitor API:

GET /api/monitor/dashboard
回傳:
- 各業者的充電站數量與狀態統計
- 在線率 (Online / Total)
- 最近 1 小時 Heartbeat 失敗的充電站清單

技術要求:
- 用 EF Core 查詢,注意 N+1 問題
- 回應時間要在 500ms 內
- 加 Redis 快取 (30 秒過期)
```

它生成的代碼包含:
- DTO 設計合理 (DashboardResponse, StationStatus, HeartbeatFailure)
- 用一條 SQL 完成查詢 (JOIN + GROUP BY)
- 自動加上 Redis 快取層
- 連 Unit Test 都寫好了

我只改了一個地方:快取的 Key 命名規則改成符合我們團隊的慣例 (`monitor:dashboard:v1`)。整個過程 **15 分鐘**搞定。

## 思維模式的轉換

用了一週 GitHub Copilot,最大的感受是:我的角色從「寫代碼的人」變成「審查代碼的人」。

以前要花 2 小時寫一個 Service,現在只要花 20 分鐘寫清楚需求,AI 生成後再花 20 分鐘審查調整。效率確實提升了,但**責任感要更強**——因為代碼不是自己一行行寫的,更容易忽略潛在問題。

另一個體會是:AI 很擅長「套用設計模式」,但不太理解「業務邏輯的細節」。像是 OCPI 規範中,不同業者對 Session.status 的定義微妙不同,這種 domain knowledge 還是要人工補充。

有時候我會懷疑:這樣下去,Junior 開發者會不會失去學習機會?但轉念一想,**與其花時間寫重複的 CRUD,不如讓他們學習架構設計、Code Review、效能優化**。工具進步了,工作內容本來就會改變。

## Token 消耗的成本問題

這週用得滿兇的,GitHub Copilot (Claude Sonnet 4.5 模型) 的 Usage 報告顯示我消耗了大約 50% 的 token。

這讓我開始思考:是不是每個情境都適合用 Copilot 的 Claude 模型?

我的初步結論:
- **複雜的重構/架構調整**: 值得用 Claude Sonnet 4.5 (省下的人力成本遠大於 API 費用)
- **簡單的 Bug 修復**: 不需要,直接手動改更快
- **學習新技術**: 可以用免費的 Gemini 先查資料,確認方向後再用 Copilot 實作

下週要研究 Local LLM,就是想解決這個成本問題。如果能在本地跑 DeepSeek,代碼又不用上傳到雲端,一舉兩得。

## 下週計畫

**目標: 用 Local LLM 取代雲端 API**

具體要做的:
1. 裝 Ollama,跑 DeepSeek-V2.5 (聽說 coding 能力接近 GPT-4)
2. 測試不同量化版本的品質差異 (Q4 vs Q8)
3. 用 Continue.dev 把 Local LLM 串到 VS Code
4. 對比雲端 vs 本地的速度和代碼品質

在完全斷網的情況下,能用 Local LLM 寫出一個可以編譯執行的 Service。
