---
layout: post
title: "理想與現實的妥協：為什麼我們選擇了微模組而非微服務"
date: 2025-01-09 10:15:00 +0800
categories: [架構設計]
tags: [微服務, 模組化架構, 技術選型, .NET Core]
---

## 從技術理想到現實考量

去年底規劃新專案的時候，團隊裡有個年輕的工程師很興奮地提案：「這次我們來做微服務吧！」

他的理由很充分：這次的會員系統範圍很廣，包括會員管理、點數、優惠券、活動、文章、問卷、通知⋯⋯林林總總加起來超過十個功能模組。「這麼多模組，不做微服務要等什麼時候？」

我看著他的架構圖，心裡有點感慨。十年前的我大概也是這樣，對新技術充滿熱情，覺得微服務就是答案。

但十三年的開發經驗告訴我：技術選擇不能只看「理想」，更要看「現實」。

我問了幾個問題：

「團隊有幾個人？」

「算你在內三個後端。」

「預算多少？」

「⋯⋯客戶給的預算很緊。」

「上線時間？」

「九個月。」

「那就不做微服務。」

「可是⋯⋯」他還想說什麼。

「我知道微服務很酷，但現在不是時候。我們先把微模組架構做好,未來真的有需要再拆。」

## 跟客戶的那場對話

其實在做技術決策之前，我先跟客戶開了一次會。

我很直接地問：「你們在意的是什麼？系統穩定？開發速度？還是未來的擴展性？」

客戶想了想說：「我們希望九個月能上線，先把核心功能做出來。未來如果使用者變多，再考慮擴展。但現在最重要的是**快**和**穩**。」

這句話給了我答案。

客戶不在乎你用微服務還是單體，他們在乎的是：
- 能不能按時交付
- 系統會不會動不動就出問題
- 維護成本會不會太高

微服務在這三點上都沒有優勢。特別是在團隊規模小、時程緊的情況下。

## 微模組：務實的選擇

回到辦公室後，我跟團隊提出一個方案：**微模組架構（Modular Monolith）**。

這不是妥協，是根據現實條件做出的最佳選擇。

我們要的只是：
- 模組之間邊界清楚，不要互相糾纏
- 可以獨立開發和測試
- 未來有需要的時候可以拆出去

這些目標不一定要用微服務才能達成。

其實我們要的只是：
- 模組之間邊界清楚，不要互相糾纏
- 可以獨立開發和測試
- 未來有需要的時候可以拆出去

這不就是「模組化單體」的概念嗎？

之前在寫 .NET Core 的時候，我就有用過 Clean Architecture 的分層結構。每個功能都有自己的資料夾，domain、application、infrastructure 分得清清楚楚。為什麼不把這個概念再延伸一下，把每個業務功能都當成一個獨立的模組？

於是我們的架構變成這樣：

```
MyApp.Api/                    # API 入口
├── Modules/
│   ├── Member/              # 會員模組
│   │   ├── Domain/
│   │   ├── Application/
│   │   └── Infrastructure/
│   ├── Points/              # 點數模組
│   │   ├── Domain/
│   │   ├── Application/
│   │   └── Infrastructure/
│   ├── Coupon/              # 優惠券模組
│   ├── Activity/            # 活動模組
│   ├── Article/             # 文章模組
│   ├── Survey/              # 問卷模組
│   └── Notification/        # 通知模組
└── Shared/                  # 共用元件
    ├── Infrastructure/
    └── Contracts/
```

每個模組都是一個獨立的 namespace，有自己的 domain model、business logic、data access。模組之間不能直接 reference，只能透過 Shared.Contracts 定義的介面溝通。

## 實際做法

### 1. 嚴格的邊界

最重要的規則：**模組之間不能直接相依**。

舉例來說，Coupon（優惠券）模組需要知道會員資訊，但它不能直接 reference Member 模組。必須透過介面：

```csharp
// Shared.Contracts/IMemberQuery.cs
public interface IMemberQuery
{
    Task<MemberDto> GetMemberAsync(string memberId);
    Task<bool> IsMemberActiveAsync(string memberId);
}

// Member 模組實作這個介面
// Member/Application/Queries/MemberQueryHandler.cs
internal class MemberQueryHandler : IMemberQuery
{
    private readonly MemberDbContext _context;
    
    public async Task<MemberDto> GetMemberAsync(string memberId)
    {
        // 實作細節
    }
}

// Coupon 模組透過介面使用
// Coupon/Application/Services/CouponService.cs
public class CouponService
{
    private readonly IMemberQuery _memberQuery;
    
    public async Task AssignCouponAsync(string memberId, string couponId)
    {
        var member = await _memberQuery.GetMemberAsync(memberId);
        if (!member.IsActive) 
        {
            throw new BusinessException("會員未啟用");
        }
        // 發放優惠券邏輯
    }
}
```

用介面隔離以後，將來要把 Member 拆成獨立服務，只要改實作就好，Coupon 模組的程式碼完全不用動。

### 2. 獨立的資料庫 Schema

雖然暫時還是用同一個 SQL Database，但每個模組都有自己的 schema：

```sql
-- Member 模組
CREATE SCHEMA Member;
CREATE TABLE Member.Members (...);
CREATE TABLE Member.MemberProfiles (...);

-- Points 模組
CREATE SCHEMA Points;
CREATE TABLE Points.Accounts (...);
CREATE TABLE Points.Transactions (...);

-- Coupon 模組
CREATE SCHEMA Coupon;
CREATE TABLE Coupon.Coupons (...);
CREATE TABLE Coupon.Assignments (...);
```
### 3. gRPC：高效能的模組通訊

有些情況需要跨模組呼叫，比如 Coupon 模組要查詢會員資訊、Points 模組要確認會員等級⋯⋯

一開始團隊有人建議用 event-driven 的方式，透過 message queue 來通訊。

我否決了這個提案。

原因很簡單：**我們的 API 需要高度即時**。

舉例來說，使用者要使用優惠券結帳，這個操作需要：
1. 檢查會員是否有效
2. 確認優惠券是否可用
3. 計算折扣後的金額
4. 扣除點數（如果有用點數折抵）

這些步驟必須在**幾百毫秒內**完成，不能用非同步的方式慢慢處理。

所以我們選擇了 **gRPC**。

```csharp
// Member 模組提供 gRPC service
public class MemberGrpcService : MemberService.MemberServiceBase
{
    private readonly IMemberQuery _memberQuery;
    
    public override async Task<GetMemberResponse> GetMember(
        GetMemberRequest request, ServerCallContext context)
    {
        var member = await _memberQuery.GetMemberAsync(request.MemberId);
        
        return new GetMemberResponse
        {
            MemberId = member.Id,
            Name = member.Name,
            Level = member.Level,
            IsActive = member.IsActive
        };
    }
}

// Coupon 模組呼叫 Member 的 gRPC service
public class CouponService
{
    private readonly MemberService.MemberServiceClient _memberClient;
    
    public async Task<bool> CanUseCouponAsync(string memberId, string couponId)
    {
        // 透過 gRPC 查詢會員資訊
        var response = await _memberClient.GetMemberAsync(
            new GetMemberRequest { MemberId = memberId });
        
        if (!response.IsActive)
        {
            return false;
        }
        
        // 其他優惠券邏輯...
        return true;
    }
}
```

gRPC 的好處：
- **效能好**：用 binary protocol（Protobuf），比 JSON 快很多
- **強型別**：有 .proto 定義，不會搞錯資料格式
- **雙向串流**：雖然我們暫時用不到，但未來有需要可以用

而且因為都在同一個 process 裡，gRPC 的呼叫基本上就是 in-process call，延遲幾乎可以忽略。

未來如果真的要拆成微服務，gRPC 也是標準的服務間通訊方式，不用重寫。

Member 模組不需要知道有哪些模組在監聽這個事件，完全解耦。

EventBus 目前是用 in-process 的 MediatR 實作，未來如果真的要拆成微服務，換成 RabbitMQ 或 Azure Service Bus 就行了。

## 這樣做的好處

### 1. 開發效率高

不用處理分散式系統的複雜問題：
- 不用處理網路延遲
- 不用處理部分失敗
- 不用處理分散式事務
- Debug 的時候直接下中斷點就好，不用看一堆 log

三個人可以專注在業務邏輯，而不是花時間搞基礎設施。

### 2. 邊界依然清楚

雖然是單體，但模組之間的邊界很清楚。Review code 的時候如果看到 Coupon 模組直接 reference Member 的 entity，馬上就知道有問題。
## 身為主管的反思

十三年的開發經驗讓我學到一件事：**好的架構不是追求最新、最酷的技術，而是最適合當下情境的方案**。

年輕的時候，我也會追逐新技術。看到 microservices、event sourcing、CQRS 這些名詞就想試試看，覺得不用就落伍了。

但現在帶團隊了，考慮的層面更廣：
- **團隊能力**：三個人能不能 handle 複雜的分散式架構？
- **時程壓力**：九個月要交付，有多少時間可以踩坑和學習？
- **維護成本**：系統上線後是我們自己維護，複雜度直接影響後續成本
- **客戶需求**：客戶要的是穩定快速的系統，不是炫技的架構圖

微模組架構是在這些限制下，最務實的選擇。

它給了我們：
- 清晰的模組邊界（未來要拆不難）
- 簡單的部署流程（一個 container 搞定）
- 良好的開發體驗（不用處理分散式問題）
- 夠用的擴展性（單體也可以 scale）

更重要的是：**它讓團隊專注在業務邏輯，而不是基礎設施**。

Martin Fowler 說：「Microservices are not the goal, they are a means to an end.」

微服務不是目標，只是達成目標的手段。

身為主管，我的職責是選擇「最有效達成目標」的手段，而不是「最潮」的手段。

這個決定，我相信九個月會證明是對的。明白：**技術選擇要符合現實條件**。

- 團隊規模小，做微服務只是自找麻煩
- 預算有限，基礎設施成本要算進去
- 時間緊迫，先求有再求好

微模組對我們來說是最務實的選擇。它給了我們模組化的好處，卻不用承擔微服務的複雜度。

Martin Fowler 說：「Microservices are not the goal, they are a means to an end.」

微服務不是目標，只是達成目標的手段。
