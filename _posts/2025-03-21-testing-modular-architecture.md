---
layout: post
title: "測試微模組架構：不要只測單元，要測整合"
date: 2025-03-21 16:45:00 +0800
categories: [測試]
tags: [單元測試, 整合測試, xUnit, .NET Core]
---

## 測試債開始累積

兩個月前開始寫測試的時候，我們很認真地寫單元測試。每個 service、每個 query handler 都有對應的測試。

測試覆蓋率看起來很漂亮：**85%**。

但上週部署到測試環境時，還是出了好幾個 bug：

1. Coupon 模組呼叫 Member 模組時，傳錯參數格式
2. Points 模組的資料庫 schema 跟 Member 模組衝突
3. gRPC service 的 timeout 設定太短，高流量時會失敗

這些問題在單元測試裡完全測不出來，因為單元測試都是用 mock。

「看來要加整合測試了。」我在每日站會上說。

## 單元測試的限制

單元測試的原則是：**隔離測試，用 mock 取代相依性**。

比如測試 `CouponService` 的時候，會 mock `IMemberQuery`：

```csharp
public class CouponServiceTests
{
    [Fact]
    public async Task GetAvailableCoupons_WhenMemberIsActive_ReturnsCoupons()
    {
        // Arrange
        var mockMemberQuery = new Mock<IMemberQuery>();
        mockMemberQuery
            .Setup(x => x.GetMemberAsync("M001"))
            .ReturnsAsync(new MemberDto 
            { 
                Id = "M001", 
                IsActive = true, 
                Level = "Gold" 
            });
        
        var mockCouponRepo = new Mock<ICouponRepository>();
        mockCouponRepo
            .Setup(x => x.GetByMemberLevelAsync("Gold"))
            .ReturnsAsync(new List<Coupon> 
            { 
                new Coupon { Id = "C001", Name = "折價券" } 
            });
        
        var service = new CouponService(
            mockMemberQuery.Object, 
            mockCouponRepo.Object);
        
        // Act
        var result = await service.GetAvailableCouponsAsync("M001");
        
        // Assert
        Assert.Single(result);
        Assert.Equal("C001", result[0].Id);
    }
}
```

這個測試可以驗證 `CouponService` 的商業邏輯是否正確。

但它測不出：
- `IMemberQuery` 的實際實作會不會出錯
- gRPC 呼叫會不會 timeout
- 資料庫 schema 是否正確

這就是單元測試的限制：**它只測「這個 class」，不測「整個系統」**。

## 整合測試的重要性

整合測試是：**測試多個元件一起工作的情況**。

在我們的微模組架構裡，最重要的是測試「模組之間的整合」。

```csharp
public class CouponModuleIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public CouponModuleIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task GetAvailableCoupons_WithRealMemberModule_ReturnsCoupons()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // 先透過 Member API 建立會員
        var memberResponse = await client.PostAsJsonAsync("/api/members", new
        {
            Name = "測試會員",
            Phone = "0912345678",
            Email = "test@example.com"
        });
        
        var member = await memberResponse.Content.ReadFromJsonAsync<MemberDto>();
        
        // Act - 呼叫 Coupon API，內部會透過 gRPC 呼叫 Member 模組
        var couponResponse = await client.GetAsync($"/api/coupons/available?memberId={member.Id}");
        
        // Assert
        couponResponse.EnsureSuccessStatusCode();
        var coupons = await couponResponse.Content.ReadFromJsonAsync<List<CouponDto>>();
        Assert.NotEmpty(coupons);
    }
}
```

這個測試：
- 真的啟動整個應用程式（包括所有模組）
- 真的透過 gRPC 呼叫 Member 模組
- 測試 Member 和 Coupon 兩個模組的整合

如果 gRPC 設定錯誤、參數格式不對、或是模組之間的介面不一致，這個測試就會失敗。

## 測試金字塔

業界有個經典的「測試金字塔」理論：

```
       /\
      /E2E\          少量的端對端測試（最慢、最脆弱）
     /------\
    /Integration\    適量的整合測試（中等速度）
   /------------\
  /Unit  Tests  \   大量的單元測試（最快、最穩定）
 /--------------\
```

我們的比例大概是：
- **單元測試**：70%（測試商業邏輯、邊界條件）
- **整合測試**：25%（測試模組整合、gRPC 通訊）
- **E2E 測試**：5%（測試關鍵流程，如註冊、結帳）

單元測試還是最多,因為它最快、最穩定。但整合測試也很重要，不能少。

## 測試資料庫的處理

整合測試會用到資料庫，但不能用正式環境的資料庫。

我們用 **TestContainers** 在測試時跑一個獨立的 SQL Server container：

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _dbContainer;
    
    public DatabaseFixture()
    {
        _dbContainer = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .Build();
    }
    
    public string ConnectionString => _dbContainer.GetConnectionString();
    
    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();
        
        // 執行 migration，建立 schema
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(ConnectionString)
            .Options;
        
        await using var context = new ApplicationDbContext(options);
        await context.Database.MigrateAsync();
    }
    
    public async Task DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

這樣每次跑測試時：
1. 自動啟動一個 SQL Server container
2. 執行 migration，建立所有 table
3. 跑測試
4. 測試結束後，刪除 container

完全隔離，不會影響其他環境。

## 測試 gRPC 通訊

測試 gRPC 比較麻煩，因為要真的啟動 gRPC server。

我們用 `WebApplicationFactory` 來做：

```csharp
public class GrpcIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    [Fact]
    public async Task MemberGrpcService_GetMember_ReturnsCorrectData()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // 先建立測試資料
        await client.PostAsJsonAsync("/api/members", new
        {
            Name = "John Doe",
            Phone = "0912345678"
        });
        
        // 建立 gRPC client
        var channel = GrpcChannel.ForAddress(client.BaseAddress, new GrpcChannelOptions
        {
            HttpClient = client
        });
        
        var grpcClient = new MemberService.MemberServiceClient(channel);
        
        // Act - 透過 gRPC 呼叫
        var response = await grpcClient.GetMemberAsync(new GetMemberRequest
        {
            MemberId = "M001"
        });
        
        // Assert
        Assert.Equal("John Doe", response.Name);
        Assert.True(response.IsActive);
    }
}
```

這個測試會：
1. 啟動整個應用程式（包括 gRPC server）
2. 建立 gRPC client
3. 真的透過 gRPC 呼叫 Member 服務
4. 驗證回應是否正確

## CI/CD 裡的測試

我們用 GitHub Actions 跑測試：

```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore
    
    - name: Run Unit Tests
      run: dotnet test --no-build --filter "Category=Unit" --verbosity normal
    
    - name: Run Integration Tests
      run: dotnet test --no-build --filter "Category=Integration" --verbosity normal
```

測試分兩個階段跑：
1. 先跑單元測試（快，幾秒鐘）
2. 再跑整合測試（慢，可能要幾分鐘）

如果單元測試失敗,就不跑整合測試，節省時間。

## 測試覆蓋率的迷思

團隊裡有個工程師很在意測試覆蓋率，一直想把覆蓋率拉到 90% 以上。

我跟他說：「覆蓋率不是重點，重點是測試品質。」

**85% 的高品質測試**，比 **95% 的低品質測試** 有用。

什麼是低品質測試？

```csharp
[Fact]
public void Member_SetName_UpdatesName()
{
    var member = new Member();
    member.SetName("John");
    Assert.Equal("John", member.Name);
}
```

這種測試只是在測 getter/setter，沒有意義。刪掉也不會怎樣。

好的測試應該測：
- **商業邏輯**：複雜的計算、判斷
- **邊界條件**：null、空字串、超大數字
- **整合點**：模組之間的介面

不要為了衝覆蓋率而寫沒意義的測試。

## 目前的狀況

測試策略調整後，雖然測試覆蓋率從 85% 降到 78%（刪了一些沒意義的測試），但 bug 數量明顯減少。

過去兩週部署到測試環境，只發現 2 個小 bug，都是 UI 的問題，後端邏輯完全正常。

這證明整合測試是有效的。

## 接下來的挑戰

測試做得再好，也不可能抓到所有 bug。

接下來要做的是：
- **Contract Testing**：測試模組之間的介面契約（用 Pact 或類似工具）
- **Performance Testing**：壓測，確保效能符合需求
- **Chaos Engineering**：故意讓服務掛掉，測試系統的恢復能力

這些都是進階的測試技巧，會在接下來幾個月慢慢導入。

專案還剩四個多月，測試這塊要繼續加強。

下週會寫監控和 logging 的設計，特別是怎麼用 Application Insights 追蹤問題。
