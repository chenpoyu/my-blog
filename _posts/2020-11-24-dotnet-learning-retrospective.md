---
layout: post
title: ".NET Core 學習回顧與展望"
date: 2020-11-24 16:00:00 +0800
categories: [框架, .NET]
tags: [.NET Core, 心得, 學習筆記]
---

這週回顧過去半年學習 .NET Core 的歷程，總結從 Java 工程師角度轉換到 .NET 生態系的心得和體悟。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## 學習歷程回顧

### 五月至七月：框架基礎

從五月開始接觸 .NET Core，首先學習的是框架的核心概念。相較於 Spring Boot，ASP.NET Core 的依賴注入系統設計更簡潔，Middleware 概念也比 Servlet Filter 更直觀。

Entity Framework Core 的 LINQ 語法一開始不太習慣，但熟悉後發現比 JPA 的 Criteria API 更具表達力。特別是 Code First 的開發模式，讓資料庫遷移變得非常順暢。

```csharp
// LINQ 的表達力
var query = await _context.Orders
    .Where(o => o.Status == OrderStatus.Confirmed)
    .Include(o => o.Items)
    .ThenInclude(i => i.Product)
    .Select(o => new OrderSummary
    {
        Id = o.Id,
        Total = o.Items.Sum(i => i.Quantity * i.UnitPrice),
        ItemCount = o.Items.Count
    })
    .ToListAsync();

// 相比 JPA 更簡潔
// CriteriaBuilder cb = em.getCriteriaBuilder();
// CriteriaQuery<OrderSummary> cq = cb.createQuery(OrderSummary.class);
// Root<Order> root = cq.from(Order.class);
// ...太長了
```

### 八月至十月：進階技術

八月開始深入研究 gRPC 和 SignalR，這兩項技術在 Java 生態中都有對應的方案，但 .NET 的整合度更高。gRPC 的 Code First 開發體驗比 Spring 的實作更流暢，SignalR 則是開箱即用，不需要像 Java 一樣配置複雜的 WebSocket。

測試方面，xUnit 的設計比 JUnit 5 更現代化，特別是 Theory 和 InlineData 的組合非常實用。WebApplicationFactory 讓整合測試變得異常簡單，不需要像 Spring Test 一樣處理複雜的 Context 載入。

```csharp
// xUnit Theory 測試
[Theory]
[InlineData(100, 0.1, 110)]
[InlineData(200, 0.2, 240)]
[InlineData(50, 0, 50)]
public void CalculateTotal_WithDiscount_ReturnsCorrectAmount(
    decimal price, decimal discount, decimal expected)
{
    var result = _calculator.CalculateTotal(price, discount);
    result.Should().Be(expected);
}

// 相比 JUnit 5 更簡潔
// @ParameterizedTest
// @CsvSource({"100,0.1,110", "200,0.2,240"})
// void testCalculateTotal(double price, double discount, double expected) { }
```

Docker 整合也讓我印象深刻。multi-stage builds 在 .NET 中的體驗非常好，編譯階段和執行階段的分離讓映像檔大小控制得很理想。Kubernetes 部署時，.NET Core 的啟動速度比 Spring Boot 快很多。

### 十一月：架構與優化

十一月專注在效能優化和架構設計。記憶體管理方面，雖然 .NET 有 GC，但 Span<T> 和 Memory<T> 提供的零配置能力讓效能優化有更多空間。這是 Java 生態比較缺乏的。

```csharp
// Span<T> 零配置處理
public void ProcessData(string input)
{
    ReadOnlySpan<char> span = input.AsSpan();
    
    // 不產生新字串
    var firstPart = span.Slice(0, 10);
    var secondPart = span.Slice(10);
    
    // 直接在 stack 上操作
    Span<int> buffer = stackalloc int[256];
    for (int i = 0; i < buffer.Length; i++)
    {
        buffer[i] = i;
    }
}
```

CI/CD 方面，GitHub Actions 和 Azure DevOps 的整合都很完善。相較於 Jenkins，YAML 定義的 Pipeline 更容易維護和版本控制。

架構設計上，Clean Architecture 和 DDD 的實踐在 .NET 社群相當成熟。MediatR 讓 CQRS 模式實作變得非常簡單，這比 Java 的 Axon Framework 輕量很多。

## 技術對比總結

### 開發體驗

| 面向 | .NET Core | Java/Spring | 評價 |
|------|-----------|-------------|------|
| 專案建立 | `dotnet new` | Spring Initializr | .NET 更快 |
| 熱重載 | 支援 | 需要 DevTools | .NET 更好 |
| 套件管理 | NuGet | Maven/Gradle | 相當 |
| 型別系統 | 更嚴格 | 較寬鬆 | .NET 更安全 |
| 語言特性 | C# 9.0 | Java 11 | C# 更現代 |

### 框架特性

| 功能 | ASP.NET Core | Spring Boot | 心得 |
|------|--------------|-------------|------|
| 依賴注入 | 內建 | 內建 | .NET 更簡潔 |
| 設定管理 | appsettings.json | application.yml | Spring 更彈性 |
| 中間件 | Middleware | Filter/Interceptor | .NET 更直觀 |
| ORM | EF Core | JPA/Hibernate | EF Core 更現代 |
| 查詢語法 | LINQ | Criteria API | LINQ 更優雅 |

### 效能表現

```csharp
// TechEmpower Benchmark Round 21 (部分結果)
// JSON 序列化 (每秒請求數)
// ASP.NET Core: 522,000
// Spring Boot: 189,000

// 資料庫查詢
// ASP.NET Core: 434,000
// Spring Boot: 156,000

// 啟動時間
// .NET Core: ~1-2 秒
// Spring Boot: ~3-5 秒
```

### 生態系比較

**優勢**：
- .NET Core 跨平台支援完整
- Visual Studio 和 VS Code 整合優秀
- Azure 雲端整合無縫
- 效能優異
- async/await 語法優雅

**劣勢**：
- 第三方函式庫不如 Java 豐富
- 社群資源相對較少（但正在成長）
- 大數據生態不如 Java
- 企業採用度低於 Java（逐漸改善）

## 實戰經驗

### 專案架構演進

半年來嘗試了不同的架構模式：

```
第一階段：傳統三層架構
├── Controllers/
├── Services/
├── Repositories/
└── Models/

第二階段：Clean Architecture
├── Domain/
├── Application/
├── Infrastructure/
└── Presentation/

第三階段：Vertical Slice
├── Features/
│   ├── Products/
│   ├── Orders/
│   └── Customers/
└── Shared/
```

每種架構都有其適用場景。小型專案用 Vertical Slice 最快，大型企業專案用 Clean Architecture 最穩，複雜業務邏輯用 DDD 最合適。

### 效能優化心得

1. **資料庫查詢優化**

```csharp
// 避免 N+1 問題
// Bad
foreach (var order in orders)
{
    var items = await _context.OrderItems
        .Where(i => i.OrderId == order.Id)
        .ToListAsync();
}

// Good
var orders = await _context.Orders
    .Include(o => o.Items)
    .ToListAsync();
```

2. **使用 Span<T> 減少配置**

```csharp
// 字串處理優化
public bool IsValidEmail(string email)
{
    var span = email.AsSpan();
    var atIndex = span.IndexOf('@');
    
    if (atIndex <= 0 || atIndex >= span.Length - 1)
    {
        return false;
    }

    var domain = span.Slice(atIndex + 1);
    return domain.IndexOf('.') > 0;
}
```

3. **快取策略**

```csharp
// 分散式快取
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product:{id}";
    var cached = await _cache.GetStringAsync(cacheKey);

    if (cached != null)
    {
        return JsonSerializer.Deserialize<Product>(cached);
    }

    var product = await _repository.GetByIdAsync(id);

    await _cache.SetStringAsync(
        cacheKey,
        JsonSerializer.Serialize(product),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        });

    return product;
}
```

### 部署經驗

Docker 部署策略：

```dockerfile
# 最佳化的 Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:3.1 AS build
WORKDIR /src
COPY ["MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:3.1 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]

# 映像檔大小: ~200MB
# 相比 Spring Boot: ~300-400MB
```

Kubernetes 部署配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 80
          initialDelaySeconds: 5
```

## 學習資源推薦

### 官方文件
- Microsoft Docs - 非常完整詳細
- .NET Blog - 新功能和最佳實踐
- ASP.NET Core GitHub - 原始碼學習

### 社群資源
- Stack Overflow - 問題解答
- Reddit r/dotnet - 討論交流
- .NET YouTube - 影片教學

### 書籍推薦
- Pro ASP.NET Core 3
- Clean Architecture (Robert C. Martin)
- Domain-Driven Design (Eric Evans)

### 實用工具
- Visual Studio 2019 / VS Code
- Rider (JetBrains)
- LINQPad - LINQ 測試工具
- dotnet-trace / dotnet-dump - 效能診斷

## 給 Java 工程師的建議

### 學習路徑

1. **第一個月**：熟悉 C# 語法和 .NET Core 基礎
   - 變數和型別系統
   - async/await 非同步
   - LINQ 查詢語法
   - 依賴注入

2. **第二個月**：掌握 ASP.NET Core
   - Middleware 機制
   - MVC 和 Web API
   - 認證和授權
   - 設定管理

3. **第三個月**：深入 EF Core
   - Code First 開發
   - 關係映射
   - 查詢優化
   - 資料庫遷移

4. **第四到六個月**：進階主題
   - gRPC 和 SignalR
   - 測試策略
   - 效能優化
   - 架構設計

### 心態調整

從 Java 轉到 .NET，需要調整的心態：

1. **不要抗拒 async/await**：雖然一開始不習慣，但比 CompletableFuture 更優雅

2. **擁抱 LINQ**：比 Stream API 更強大，值得投資時間學習

3. **相信編譯器**：C# 的型別推斷很聰明，不需要到處寫型別

4. **善用擴充方法**：這是 C# 獨有的特性，讓程式碼更乾淨

5. **接受 null 處理**：nullable reference types 比 Optional 更實用

### 常見誤區

1. **不要把 Java 的習慣帶過來**
   ```csharp
   // Bad (Java 風格)
   public List<Product> GetProducts()
   {
       List<Product> products = new List<Product>();
       // ...
       return products;
   }

   // Good (C# 風格)
   public List<Product> GetProducts()
   {
       var products = new List<Product>();
       // ...
       return products;
   }
   ```

2. **不要過度設計**
   ```csharp
   // Bad (過度抽象)
   public interface IProductFactory
   {
       IProduct CreateProduct();
   }

   // Good (簡單直接)
   public class Product
   {
       public static Product Create(string name, decimal price) =>
           new Product { Name = name, Price = price };
   }
   ```

3. **不要忽略 async/await**
   ```csharp
   // Bad
   var result = GetDataAsync().Result;

   // Good
   var result = await GetDataAsync();
   ```

## 未來展望

### 持續學習

繼續深入以下領域：
- **進階效能優化**：深入理解 .NET Core 運行時機制
- **微服務架構**：Service Mesh、分散式追蹤
- **雲端整合**：Azure、AWS 等雲端服務
- **DevOps 實踐**：CI/CD、容器化、監控

### 技術趨勢

1. **雲端原生**：.NET 在 Azure 的整合將更緊密
2. **容器化**：Docker 和 Kubernetes 支援持續改善
3. **微服務**：Dapr 等工具讓微服務開發更簡單
4. **效能**：持續優化，朝向更快的目標

### 學習計畫

接下來會繼續深入：
- Blazor WebAssembly
- Orleans (Actor Model)
- .NET MAUI (跨平台 UI)
- F# 函數式程式設計

## 總結

半年的 .NET Core 學習旅程，從陌生到熟悉，從模仿到創新。作為 Java 工程師，轉換到 .NET 生態並沒有想像中困難。兩個生態有很多相似之處，但 .NET Core 在某些方面確實有其獨特優勢。

關鍵在於保持開放的心態，不要固守舊有的思維模式。每個框架都有其設計哲學，理解並擁抱這些哲學，才能真正發揮框架的威力。

.NET Core 是一個優秀的框架，值得每位開發者學習和使用。無論是性能、開發體驗還是雲端整合，都展現出了微軟在現代化開發方向的用心。

希望這半年的學習筆記能幫助更多想要學習 .NET Core 的 Java 工程師。技術無國界，框架無高下，持續學習才是王道。

---

這是 .NET Core 學習系列的最後一篇文章。感謝一路以來的陪伴，我們下個系列再見。
