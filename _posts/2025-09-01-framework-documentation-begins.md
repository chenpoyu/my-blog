---
layout: post
title: "建立公司框架文件 - 讓 AI 快速上手專案規範"
date: 2025-09-01 17:20:00 +0800
categories: [技術筆記, AI]
tags: [.NET Core, 框架設計, AI 輔助開發]
---

## 為什麼要建立框架文件?

上個月用 Local LLM 和 RAG 做了不少實驗,但每次開新專案時,還是得花很多時間跟 AI 解釋「我們的專案長什麼樣」。

舉個例子:我跟 GitHub Copilot (Claude Sonnet 4.5) 說「幫我生成一個 User CRUD API」,它生成的代碼:
- 用 Dapper 而不是我們的 EF Core
- 沒有統一的 BaseController
- 錯誤處理格式不對
- 沒有審計欄位 (CreateUser, UpdateIp 那些)

結果生成的代碼要大改,反而更慢。

**問題的根源**: AI 不知道我們的專案規範和架構設計。

## 解決方案:建立一份「框架指南」

靈感來自上週跟 Junior 開發者 onboarding 的經驗。我花了 2 小時解釋專案架構:
- 為什麼要用 N-Layer Architecture?
- BaseController 和 BaseService 怎麼用?
- 新增 API 的標準流程是什麼?

我突然想到:**如果我把這些整理成文件,不就能一次教會 AI 和新人?**

目標很明確:
1. 寫一份完整的框架指南
2. 包含所有常用模式和範例代碼
3. 讓 Local LLM 學會我們的規範
4. 以後開新專案時,丟給 AI 就能生成符合規範的代碼

## 第一步:盤點現有架構

週末花了 4 小時,把現有專案的核心部分整理出來。

### 專案結構
我們用的是標準的 **N-Layer Architecture**:
- **Presentation Layer**: Controllers + Filters
- **Application Layer**: Services + Validators
- **Domain Layer**: Entities + DTOs
- **Infrastructure Layer**: Repository + DbContext

技術棧:
- .NET Core 8.0
- EF Core (資料存取)
- JWT (認證)
- AutoMapper (物件映射)
- FluentValidation (驗證)
- NLog (日誌)
- Redis (快取)

### 統一的代碼模式

**BaseController**:所有 Controller 繼承自 `SystemBaseController`,提供統一的回應格式:

```csharp
protected IActionResult HandleResult<T>(T data) 
    => Ok(new BaseSystemResponse<T>(data));
```

**BaseService**:所有 Service 繼承自 `BaseSystemService`,自動從 JWT Token 解析使用者資訊 (UserId, OrgId, Roles)。

**BaseRepository**:封裝常用的 CRUD 操作,支援分頁、排序、AutoMapper 投影。

**全域例外處理**:用 `SystemExceptionFilter` 攔截所有例外,回傳統一格式:

```json
{
  "code": "40000",
  "message": "必填參數缺少",
  "result": {}
}
```

## 開始撰寫文件

我決定用 **Markdown 格式**,因為:
1. 方便閱讀和維護
2. 容易轉成 RAG 的 Vector Database
3. 可以直接放在 Git 版本控制

文件大綱:
```
1. 專案架構 (架構圖 + 分層說明)
2. 目錄結構 (標準專案結構)
3. 核心元件
   - Web API 層 (BaseController, Filter)
   - Application 層 (BaseService, Validator)
   - Domain 層 (Entity, DTO, Exception)
   - Infrastructure 層 (BaseRepository, DbContext)
4. 常用模式 (錯誤處理、驗證、JWT、快取)
5. 快速開始範本 (完整 CRUD 範例)
```

### 第一個範例:BaseController

```csharp
[ApiController]
[AllowAnonymous]
public class SystemBaseController : ControllerBase
{
    /// <summary>
    /// 統一回應格式
    /// </summary>
    protected IActionResult HandleResult<T>(T data) 
        where T : class
        => Ok(new BaseSystemResponse<T>(data));

    /// <summary>
    /// FluentValidation 驗證
    /// </summary>
    protected async Task ValidateRequestAsync<T>(
        T request, 
        IValidator<T> validator)
    {
        var result = await validator.ValidateAsync(request);
        
        if (!result.IsValid)
        {
            var missingFields = result.Errors
                .Where(e => e.ErrorMessage.Contains("必填"))
                .ToList();
                
            if (missingFields.Any())
                throw new BaseException("必填參數缺少", "40000");
            
            throw new BaseException("格式錯誤或填寫不正確", "40001");
        }
    }
}
```

關鍵是**加入充足的註解和使用情境說明**,讓 AI 知道這個 class 的用途。

### 完整 CRUD 範例

文件最重要的部分是「快速開始範本」。我寫了一個完整的 Product CRUD 範例,從頭到尾包含:

**Step 1**: 建立 Entity
**Step 2**: 建立 DTOs (Request/Response)
**Step 3**: 建立 Repository (Interface + 實作)
**Step 4**: 建立 Service (Interface + 實作)
**Step 5**: 建立 Controller
**Step 6**: 建立 AutoMapper Profile
**Step 7**: 註冊到 DI Container
**Step 8**: 更新 DbContext

每一步都有完整代碼,不是片段。這樣 AI 才能理解完整的開發流程。

## 遇到的挑戰

### 1. 文件太長,AI 會不會看不完?

我的文件寫到 **6000+ 行**,擔心 Local LLM 的 Context Window 不夠。

**解法**: 用 RAG 切分文件。每個章節獨立儲存到 Vector Database,需要時才檢索相關部分。

### 2. 範例代碼要寫多詳細?

太簡單 → AI 不知道細節
太複雜 → AI 學習成本高

**折衷方案**: 核心邏輯寫完整,周邊功能用註解說明。

例如:BaseRepository 的 `QueryAsync` 方法,我把分頁、排序、投影邏輯都寫出來,但複雜的 Expression 組合用註解說明即可。

### 3. 如何讓文件保持更新?

框架會持續演進,文件可能過時。

**計畫**: 
- 每次改 BaseController / BaseService 時,同步更新文件
- 每季 Review 一次,確保文件和實際代碼一致

## 初步成果

寫完第一版文件後,我做了個簡單測試:

把文件丟給 GitHub Copilot (Claude Sonnet 4.5),然後說:
```
依照框架指南,幫我生成一個 Order CRUD API
包含:
- Entity: Order (訂單編號、客戶ID、金額、狀態)
- 完整的 CRUD 端點
- 分頁查詢 (支援狀態篩選)
```

**結果**:
- ✅ 生成的代碼結構 100% 符合規範
- ✅ BaseController, BaseService 都用對了
- ✅ 審計欄位都有加
- ✅ AutoMapper Profile 正確
- ⚠️ 有些商業邏輯判斷還是要手動調整

整體節省了 **60-70% 的重複工作**。以前要 2 小時寫完的 CRUD,現在 30 分鐘搞定。

## 下一步:讓 Local LLM 學會框架

現在文件是給 GitHub Copilot (雲端 AI) 用的,每次都要把文件丟上去,有點慢。

下週計畫:
1. 將文件切分成小塊,存入 ChromaDB
2. 用 Continue.dev 整合 Local LLM
3. 實作「框架知識庫」,讓 DeepSeek 能快速檢索規範
4. 測試在完全離線環境下,AI 能不能正確生成代碼

**目標**: 不依賴網路,用 Local LLM 就能快速開發符合公司規範的模組。

## 小結

建立框架文件的過程,其實是**把隱性知識顯性化**。

以前這些規範都在腦袋裡,或散落在各個專案的代碼中。現在統一整理成文件,好處是:
- 新人 onboarding 更快 (給他看文件就好)
- AI 能學習公司規範
- 團隊有統一的開發標準

寫文件很花時間 (整整一個週末),但長期來看絕對值得。尤其當 AI 能真正理解這份文件時,開發效率會大幅提升。

期待下週把這份文件「餵」給 Local LLM,看看效果如何。
