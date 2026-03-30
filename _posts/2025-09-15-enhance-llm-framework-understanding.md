---
layout: post
title: "強化 Local LLM 的框架理解 - Prompt Engineering 與微調"
date: 2025-09-15 14:10:00 +0800
categories: [技術筆記, AI]
tags: [Local LLM, Prompt Engineering, Fine-tuning, DeepSeek]
---

## RAG 的局限性

上週把框架文件轉成 RAG 知識庫後,測試效果不錯,但遇到一些問題:

### 問題 1: AI 不理解「潛規則」

例如我們團隊有個不成文的規定:**所有 Entity 的主鍵都用 `{Entity}Id` 格式** (例如 `UserId`, `ProductId`)。

我問 AI:
```
建立一個 Order Entity
```

它生成:
```csharp
public class Order
{
    public string Id { get; set; }  // ❌ 應該是 OrderId
    public string CustomerId { get; set; }
    ...
}
```

雖然文件有寫,但 AI 沒注意到這個細節。

### 問題 2: AI 會「創造」不存在的方法

有次我問:
```
用 BaseRepository 查詢所有 active 的 User
```

它生成:
```csharp
var users = await _userRepository.GetAllActiveAsync();  // ❌ 沒這個方法
```

實際上 BaseRepository 只有 `FirstOrDefaultAsync`, `QueryAsync` 等方法,沒有 `GetAllActiveAsync`。

AI 可能是從別的專案「學到」這個命名模式,就套用過來了。

### 問題 3: 商業邏輯判斷不夠準確

我問:
```
使用者新增時,如果帳號已存在,應該怎麼處理?
```

AI 回答:
```csharp
if (existingUser != null)
    return BadRequest("帳號已存在");  // ❌ 應該 throw BaseException
```

它用了 `BadRequest`,但我們的規範是:**所有錯誤都 throw BaseException,由全域過濾器統一處理**。

## 解決方案:三管齊下

### 1. 改進 Prompt (System Message)

之前的 System Message 很簡單:
```
你是一個 .NET Core 專家,請幫我生成代碼。
```

改成更詳細的指令:

```markdown
你是一個專精 .NET Core 8 的開發專家,熟悉公司的框架規範。

## 核心原則
1. 所有 Controller 必須繼承 SystemBaseController
2. 所有 Service 必須繼承 BaseSystemService
3. 所有 Repository 必須繼承 BaseSystemRepository<TEntity>
4. Entity 主鍵命名格式: {EntityName}Id (例如 UserId, ProductId)
5. 錯誤處理: 使用 throw new BaseException(message, code)
6. 不可自創方法名,只能使用框架提供的方法

## 審計欄位 (必須包含)
- CreateAt, CreateUser, CreateIp
- UpdateAt, UpdateUser, UpdateIp

## 統一回應格式
- 成功: return HandleResult(data)
- 空回應: return ReturnBaseEmpty()
- 錯誤: throw new BaseException(message, code)

## BaseRepository 可用方法
- QueryAsync<TDto>(expression, pageReq)
- FirstOrDefaultAsync(expression)
- InsertAsync(entity)
- UpdateAsync(entity)
- DeleteAsync(entity)

請嚴格遵守以上規範,不要使用未定義的方法或模式。
```

改進後,再測試同樣的問題:

**測試**: 建立 Order Entity

```csharp
public class Order
{
    public string OrderId { get; set; }  // ✅ 正確!
    public string CustomerId { get; set; }
    public decimal Amount { get; set; }
    public string Status { get; set; }
    
    // 審計欄位
    public DateTime CreateAt { get; set; }
    public string CreateUser { get; set; }
    public string CreateIp { get; set; }
    public DateTime UpdateAt { get; set; }
    public string UpdateUser { get; set; }
    public string UpdateIp { get; set; }
}
```

✅ 完美!主鍵名稱對了,審計欄位也有了。

### 2. Few-Shot Prompting (給範例)

除了規則,還要給「好的範例」和「壞的範例」。

```markdown
## 正確範例

❌ 錯誤寫法:
```csharp
if (user == null)
    return NotFound();
```

✅ 正確寫法:
```csharp
if (user == null)
    throw new BaseException("使用者不存在", "9SY07");
```

---

❌ 錯誤寫法:
```csharp
var users = await _context.Users.ToListAsync();
```

✅ 正確寫法:
```csharp
var users = await _userRepository.QueryAsync<GetUserRes>(
    x => x.Status == "active",
    new BasePageReq { PageIndex = 1, PageSize = 100 }
);
```

---

❌ 錯誤寫法:
```csharp
public class Product
{
    public string Id { get; set; }
}
```

✅ 正確寫法:
```csharp
public class Product
{
    public string ProductId { get; set; }
}
```
```

加了這些範例後,AI 犯錯的機率大幅下降。

### 3. 建立「檢查清單」Prompt

生成代碼後,讓 AI 自己檢查:

```markdown
代碼生成後,請檢查以下項目:

1. [ ] Controller 有繼承 SystemBaseController?
2. [ ] Service 有繼承 BaseSystemService?
3. [ ] Repository 有繼承 BaseSystemRepository?
4. [ ] Entity 主鍵命名正確 ({Entity}Id)?
5. [ ] 有包含所有審計欄位?
6. [ ] 錯誤處理用 throw BaseException?
7. [ ] 沒有使用未定義的方法?
8. [ ] AutoMapper Profile 有設定?

如果有任何項目不符,請重新生成代碼。
```

這招很有效!AI 會先生成代碼,然後自己檢查一遍,發現問題後自動修正。

## 進階:Fine-tuning Local LLM (實驗性)

Prompt Engineering 能解決大部分問題,但還有極限。於是我嘗試了 **Fine-tuning DeepSeek**。

### 準備訓練資料

從現有專案中提取 100 組「問題-答案」配對:

```json
[
  {
    "instruction": "建立一個 User Entity,包含帳號、密碼、Email、手機",
    "input": "",
    "output": "public class User\n{\n    public string UserId { get; set; }\n    public string Account { get; set; }\n    public string Password { get; set; }\n    public string Email { get; set; }\n    public string Mobile { get; set; }\n    public string Status { get; set; }\n    \n    // 審計欄位\n    public DateTime CreateAt { get; set; }\n    public string CreateUser { get; set; }\n    public string CreateIp { get; set; }\n    public DateTime UpdateAt { get; set; }\n    public string UpdateUser { get; set; }\n    public string UpdateIp { get; set; }\n}"
  },
  {
    "instruction": "實作 UserRepository,包含查詢使用者列表的方法",
    "input": "",
    "output": "public class UserRepository : BaseSystemRepository<User>, IUserRepository\n{\n    private readonly SystemContext _context;\n    \n    public UserRepository(SystemContext context, IMapper mapper, IHttpContextAccessor httpContextAccessor)\n        : base(context, mapper, httpContextAccessor)\n    {\n        _context = context;\n    }\n    \n    public async Task<PageResult<GetUserRes>> GetUsersAsync(GetUserReq req)\n    {\n        return await QueryAsync<GetUserRes>(req.GetExpression(), req);\n    }\n}"
  }
  // ... 98 more examples
]
```

### 使用 Unsloth 微調

```bash
# 安裝 Unsloth (專為 Fine-tuning 優化的工具)
pip install unsloth

# 微調腳本
python fine_tune.py \
  --base_model "deepseek-ai/deepseek-coder-6.7b-instruct" \
  --data_path "./training_data.json" \
  --output_dir "./models/deepseek-framework-tuned" \
  --num_epochs 3 \
  --learning_rate 2e-5
```

**訓練時間**: 在我的 Mac Mini Pro (M4, 48GB RAM) 上跑了 **6 小時**。

### 測試微調後的模型

用 Ollama 載入微調後的模型:

```bash
ollama create deepseek-framework-tuned -f Modelfile
```

測試同樣的問題:

**問題**: 建立一個 Order Service,實作新增訂單的方法

**原始 DeepSeek 回答**:
```csharp
public class OrderService
{
    public async Task CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order { ... };
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
    }
}
```
❌ 沒繼承 BaseSystemService,直接用 `_context`

**微調後的 DeepSeek 回答**:
```csharp
public class OrderService : BaseSystemService, IOrderService
{
    private readonly IOrderRepository _orderRepository;
    
    public OrderService(IHttpContextAccessor httpContextAccessor, IOrderRepository orderRepository)
        : base(httpContextAccessor)
    {
        _orderRepository = orderRepository;
    }
    
    public async Task PostOrderAsync(PostOrderReq req)
    {
        var order = new Order
        {
            OrderId = Guid.NewGuid().ToString("N"),
            CustomerId = req.CustomerId,
            Amount = req.Amount,
            Status = "pending",
            CreateAt = DateTime.UtcNow,
            CreateUser = UserId ?? "system",
            CreateIp = GetClientIp()
        };
        
        await _orderRepository.InsertAsync(order);
    }
}
```
✅ 完全符合規範!

## 效果對比

做了 50 個測試案例,對比三種方法:

| 方法 | 準確度 | 速度 | 成本 |
|------|--------|------|------|
| RAG 檢索 (無 Prompt 優化) | 70% | 3 秒 | 低 |
| RAG + 優化 Prompt | 88% | 3 秒 | 低 |
| RAG + Prompt + Fine-tuning | 95% | 2.5 秒 | 中 (一次性訓練成本) |

**結論**: 
- **Prompt Engineering 是性價比最高的方法**,從 70% 提升到 88%
- **Fine-tuning 能再提升 7%**,但需要準備訓練資料和訓練時間
- 速度差異不大,主要影響在準確度

## 實際應用場景

現在我的工作流程:

### 場景 1: 快速生成標準 CRUD

```
@framework-guide 建立一個 Invoice (發票) 模組
包含:
- Entity: 發票編號、客戶ID、金額、狀態、開立日期
- 完整 CRUD API (查詢支援日期範圍篩選)
- Repository, Service, Controller 都要
```

**AI 生成時間**: 30 秒
**代碼符合規範**: 95%
**我需要調整的部分**: 5% (主要是商業邏輯細節)

### 場景 2: 代碼審查

把同事寫的代碼貼給 AI:

```
@framework-guide 檢查這段代碼是否符合框架規範:

[貼上代碼]
```

AI 會指出:
- ❌ Controller 沒繼承 SystemBaseController
- ❌ 錯誤處理用了 `return BadRequest()` 而不是 `throw BaseException`
- ✅ Repository 使用正確
- ✅ Entity 審計欄位完整

### 場景 3: 重構建議

```
@framework-guide 這段代碼有什麼可以改進的地方?

[貼上代碼]
```

AI 會建議:
- 可以把重複的查詢邏輯抽到 BaseRepository
- 錯誤碼應該統一定義在 ErrorCodes 常數類別
- 複雜的查詢條件可以用 Specification Pattern

## 遇到的挑戰

### 挑戰 1: Fine-tuning 資料不夠多

100 組訓練資料還是太少,有些罕見情境 AI 還是會出錯。

**計畫**: 從現有 10 個專案中提取更多範例,目標 **500-1000 組**。

### 挑戰 2: 框架持續演進

上週我們把 `BaseService` 加了一個新方法 `GetClientIp()`,但 AI 不知道。

**解法**: 
1. 更新 RAG 知識庫 (每次框架改動都更新文件)
2. 定期重新 Fine-tuning (每季一次)

### 挑戰 3: 團隊其他人怎麼用?

我的設定在本機,其他同事要用的話,得重複一次設定流程。

**計畫**: 
- 建立 Docker Image,包含 Ollama + 微調模型 + RAG API
- 團隊成員只要 `docker run` 就能用
- 或者架設內部 API Server,所有人共用

## 小結

這週的核心改進:
1. **優化 System Prompt** - 明確規範和範例,準確度從 70% → 88%
2. **Few-Shot Prompting** - 給「正確」和「錯誤」範例,讓 AI 學習模式
3. **Fine-tuning (實驗性)** - 用公司代碼訓練模型,準確度達 95%

現在 AI 不只是「代碼生成器」,更像是「懂公司規範的助手」。它知道:
- 我們的命名慣例
- 標準的錯誤處理方式
- 哪些方法可以用,哪些不能用
- 常見的重構模式

**下一步**: 實戰測試!用 AI 從頭建立一個完整的微服務模組,看看能節省多少時間。
