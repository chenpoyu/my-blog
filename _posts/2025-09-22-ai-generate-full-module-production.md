---
layout: post
title: "實戰:用 AI 生成完整微服務模組 - 效率提升 70%"
date: 2025-09-22 18:30:00 +0800
categories: [技術筆記, AI]
tags: [AI 輔助開發, .NET Core, 實戰案例]
---

## 終極測試:建立一個真實的微服務模組

前三週做了這些準備:
- 建立框架文件
- 轉成 RAG 知識庫
- 優化 Prompt 和微調 Local LLM

這週要來驗證成果:**用 AI 從零建立一個完整的微服務模組**。

## 實戰案例:充電站預約系統

PM 丟來一個新需求:

```
需求:充電站預約管理
功能:
1. 使用者可以預約充電站時段
2. 查詢可用時段 (依地點、日期篩選)
3. 取消預約 (只能取消未開始的預約)
4. 管理員查詢所有預約紀錄 (支援分頁、狀態篩選)

技術要求:
- RESTful API
- 支援 JWT 認證
- 資料持久化 (SQL Server)
- 需要單元測試
```

以前這種需求,我大概要花 **2 天** 才能完成。現在用 AI 試試看。

## Step 1: 分析需求,拆解任務 (10 分鐘)

先在筆記本寫下需要的元件:

**資料模型**:
- `Reservation` Entity (預約紀錄)
  - 預約ID、使用者ID、充電站ID
  - 開始時間、結束時間、狀態
  - 審計欄位 (CreateAt, UpdateUser 等)

**API 端點**:
- `POST /api/reservation` - 建立預約
- `GET /api/reservation` - 查詢預約列表 (分頁)
- `GET /api/reservation/{id}` - 取得單一預約
- `DELETE /api/reservation/{id}` - 取消預約
- `GET /api/reservation/available-slots` - 查詢可用時段

**商業邏輯**:
- 預約時段不可重疊
- 只能取消「pending」或「confirmed」狀態的預約
- 過去的時段不能預約

## Step 2: 用 AI 生成 Entity (5 分鐘)

在 VS Code 的 Continue.dev 輸入:

```
@framework-guide 建立一個 Reservation Entity

欄位需求:
- ReservationId (主鍵)
- UserId (使用者ID)
- StationId (充電站ID)
- StartTime (開始時間)
- EndTime (結束時間)
- Status (狀態: pending, confirmed, cancelled, completed)
- CancelReason (取消原因,可選)
- 審計欄位

請確保符合框架規範
```

**AI 生成** (2 秒):

```csharp
namespace Backend.Database.Entities;

public class Reservation
{
    public string ReservationId { get; set; } = null!;
    public string UserId { get; set; } = null!;
    public string StationId { get; set; } = null!;
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public string Status { get; set; } = "pending";
    public string? CancelReason { get; set; }
    
    // 審計欄位
    public DateTime CreateAt { get; set; }
    public string CreateUser { get; set; } = null!;
    public string CreateIp { get; set; } = null!;
    public DateTime UpdateAt { get; set; }
    public string UpdateUser { get; set; } = null!;
    public string UpdateIp { get; set; } = null!;
}
```

✅ 完美!直接複製貼上,一個字都不用改。

## Step 3: 生成 DTOs (10 分鐘)

```
@framework-guide 為 Reservation 建立所有需要的 DTOs:

1. GetReservationReq - 查詢預約列表 (支援分頁、狀態篩選、日期範圍)
2. GetReservationRes - 預約回應 (包含充電站名稱)
3. PostReservationReq - 建立預約請求
4. GetAvailableSlotsReq - 查詢可用時段請求
5. GetAvailableSlotsRes - 可用時段回應
```

**AI 生成的代碼** (10 秒):

```csharp
// GetReservationReq.cs
public class GetReservationReq : BasePageReq
{
    public string? UserId { get; set; }
    public string? StationId { get; set; }
    public string? Status { get; set; }
    public DateTime? StartDate { get; set; }
    public DateTime? EndDate { get; set; }

    public Expression<Func<Reservation, bool>> GetExpression()
    {
        Expression<Func<Reservation, bool>> expression = x => true;

        if (!string.IsNullOrWhiteSpace(UserId))
            expression = expression.And(x => x.UserId == UserId);

        if (!string.IsNullOrWhiteSpace(StationId))
            expression = expression.And(x => x.StationId == StationId);

        if (!string.IsNullOrWhiteSpace(Status))
            expression = expression.And(x => x.Status == Status);

        if (StartDate.HasValue)
            expression = expression.And(x => x.StartTime >= StartDate.Value);

        if (EndDate.HasValue)
            expression = expression.And(x => x.EndTime <= EndDate.Value);

        return expression;
    }
}

// GetReservationRes.cs
public class GetReservationRes
{
    public string Id { get; set; }
    public string UserId { get; set; }
    public string StationId { get; set; }
    public string StationName { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public string Status { get; set; }
    public DateTime CreateAt { get; set; }
}

// PostReservationReq.cs
public class PostReservationReq
{
    public string StationId { get; set; } = null!;
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
}

// GetAvailableSlotsReq.cs
public class GetAvailableSlotsReq
{
    public string StationId { get; set; } = null!;
    public DateTime Date { get; set; }
}

// GetAvailableSlotsRes.cs
public class GetAvailableSlotsRes
{
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public bool IsAvailable { get; set; }
}
```

✅ `GetExpression()` 方法、BasePageReq 繼承、命名規範全都對。只需要微調 `GetAvailableSlotsRes` 的結構。

## Step 4: 生成 Repository (15 分鐘)

```
@framework-guide 建立 ReservationRepository

需要實作的方法:
1. 查詢某時段是否有重疊的預約 (HasConflictAsync)
2. 查詢某充電站在指定日期的所有預約 (GetReservationsByDateAsync)
3. 查詢可用時段 (GetAvailableSlotsAsync)

請繼承 BaseSystemRepository<Reservation>
```

**AI 生成** (8 秒):

```csharp
public interface IReservationRepository : IBaseSystemRepository<Reservation>
{
    Task<bool> HasConflictAsync(string stationId, DateTime startTime, DateTime endTime, string? excludeReservationId = null);
    Task<List<Reservation>> GetReservationsByDateAsync(string stationId, DateTime date);
}

public class ReservationRepository : BaseSystemRepository<Reservation>, IReservationRepository
{
    private readonly SystemContext _context;

    public ReservationRepository(
        SystemContext context,
        IMapper mapper,
        IHttpContextAccessor httpContextAccessor)
        : base(context, mapper, httpContextAccessor)
    {
        _context = context;
    }

    public async Task<bool> HasConflictAsync(
        string stationId, 
        DateTime startTime, 
        DateTime endTime, 
        string? excludeReservationId = null)
    {
        var query = _context.Reservations.AsNoTracking()
            .Where(x => x.StationId == stationId)
            .Where(x => x.Status == "confirmed" || x.Status == "pending")
            .Where(x => 
                (x.StartTime < endTime && x.EndTime > startTime) // 時段重疊判斷
            );

        if (!string.IsNullOrWhiteSpace(excludeReservationId))
            query = query.Where(x => x.ReservationId != excludeReservationId);

        return await query.AnyAsync();
    }

    public async Task<List<Reservation>> GetReservationsByDateAsync(string stationId, DateTime date)
    {
        var startOfDay = date.Date;
        var endOfDay = date.Date.AddDays(1);

        return await _context.Reservations.AsNoTracking()
            .Where(x => x.StationId == stationId)
            .Where(x => x.StartTime >= startOfDay && x.StartTime < endOfDay)
            .OrderBy(x => x.StartTime)
            .ToListAsync();
    }
}
```

✅ 繼承正確、DI 注入正確、時段重疊邏輯也對!

**我只需要補充**:
- `GetAvailableSlotsAsync` 方法 (這個商業邏輯比較複雜,AI 生成的不夠完整)

手動加了 20 行代碼:

```csharp
public async Task<List<TimeSlot>> GetAvailableSlotsAsync(string stationId, DateTime date)
{
    var reservations = await GetReservationsByDateAsync(stationId, date);
    
    // 假設營業時間 08:00 - 22:00,每次預約 1 小時
    var slots = new List<TimeSlot>();
    var currentTime = date.Date.AddHours(8);
    var endTime = date.Date.AddHours(22);
    
    while (currentTime < endTime)
    {
        var slotEnd = currentTime.AddHours(1);
        var isAvailable = !reservations.Any(r => 
            r.StartTime < slotEnd && r.EndTime > currentTime
        );
        
        slots.Add(new TimeSlot 
        { 
            StartTime = currentTime, 
            EndTime = slotEnd, 
            IsAvailable = isAvailable 
        });
        
        currentTime = slotEnd;
    }
    
    return slots;
}
```

## Step 5: 生成 Service (20 分鐘)

```
@framework-guide 建立 ReservationService

需要實作:
1. GetReservationsAsync - 查詢預約列表
2. GetReservationByIdAsync - 取得單一預約
3. PostReservationAsync - 建立預約 (檢查時段衝突、不可預約過去時段)
4. CancelReservationAsync - 取消預約 (只能取消 pending/confirmed 狀態)
5. GetAvailableSlotsAsync - 查詢可用時段

請繼承 BaseSystemService
```

**AI 生成** (12 秒):

```csharp
public interface IReservationService
{
    Task<PageResult<GetReservationRes>> GetReservationsAsync(GetReservationReq req);
    Task<GetReservationRes> GetReservationByIdAsync(string id);
    Task PostReservationAsync(PostReservationReq req);
    Task CancelReservationAsync(string id, string? reason);
    Task<List<GetAvailableSlotsRes>> GetAvailableSlotsAsync(GetAvailableSlotsReq req);
}

public class ReservationService : BaseSystemService, IReservationService
{
    private readonly IReservationRepository _reservationRepository;
    private readonly IMapper _mapper;

    public ReservationService(
        IHttpContextAccessor httpContextAccessor,
        IReservationRepository reservationRepository,
        IMapper mapper)
        : base(httpContextAccessor)
    {
        _reservationRepository = reservationRepository;
        _mapper = mapper;
    }

    public async Task<PageResult<GetReservationRes>> GetReservationsAsync(GetReservationReq req)
    {
        return await _reservationRepository.QueryAsync<GetReservationRes>(
            req.GetExpression(),
            req
        );
    }

    public async Task<GetReservationRes> GetReservationByIdAsync(string id)
    {
        var reservation = await _reservationRepository.FirstOrDefaultAsync(
            x => x.ReservationId == id
        );

        if (reservation == null)
            throw new BaseException("預約不存在", "40404");

        return _mapper.Map<GetReservationRes>(reservation);
    }

    public async Task PostReservationAsync(PostReservationReq req)
    {
        // 檢查預約時段是否在未來
        if (req.StartTime <= DateTime.UtcNow)
            throw new BaseException("不可預約過去的時段", "40001");

        // 檢查結束時間是否大於開始時間
        if (req.EndTime <= req.StartTime)
            throw new BaseException("結束時間必須大於開始時間", "40001");

        // 檢查時段衝突
        var hasConflict = await _reservationRepository.HasConflictAsync(
            req.StationId,
            req.StartTime,
            req.EndTime
        );

        if (hasConflict)
            throw new BaseException("此時段已被預約", "40001");

        var reservation = new Reservation
        {
            ReservationId = Guid.NewGuid().ToString("N"),
            UserId = UserId ?? throw new BaseException("使用者未登入", "40100"),
            StationId = req.StationId,
            StartTime = req.StartTime,
            EndTime = req.EndTime,
            Status = "pending",
            CreateAt = DateTime.UtcNow,
            CreateUser = UserId,
            CreateIp = GetClientIp(),
            UpdateAt = DateTime.UtcNow,
            UpdateUser = UserId,
            UpdateIp = GetClientIp()
        };

        await _reservationRepository.InsertAsync(reservation);
    }

    public async Task CancelReservationAsync(string id, string? reason)
    {
        var reservation = await _reservationRepository.FirstOrDefaultAsync(
            x => x.ReservationId == id
        );

        if (reservation == null)
            throw new BaseException("預約不存在", "40404");

        // 只能取消自己的預約 (除非是管理員)
        if (reservation.UserId != UserId && !Roles.Contains("Admin"))
            throw new BaseException("無權限取消此預約", "40300");

        // 只能取消 pending 或 confirmed 狀態
        if (reservation.Status != "pending" && reservation.Status != "confirmed")
            throw new BaseException("此預約無法取消", "40001");

        // 不能取消已開始的預約
        if (reservation.StartTime <= DateTime.UtcNow)
            throw new BaseException("無法取消已開始的預約", "40001");

        reservation.Status = "cancelled";
        reservation.CancelReason = reason;
        reservation.UpdateAt = DateTime.UtcNow;
        reservation.UpdateUser = UserId ?? "system";
        reservation.UpdateIp = GetClientIp();

        await _reservationRepository.UpdateAsync(reservation);
    }

    public async Task<List<GetAvailableSlotsRes>> GetAvailableSlotsAsync(GetAvailableSlotsReq req)
    {
        var slots = await _reservationRepository.GetAvailableSlotsAsync(
            req.StationId,
            req.Date
        );

        return _mapper.Map<List<GetAvailableSlotsRes>>(slots);
    }
}
```

**驚艷!!!** AI 自動加了:
- ✅ 所有必要的驗證 (時段檢查、權限檢查)
- ✅ 從 `BaseSystemService` 取得 `UserId` 和 `Roles`
- ✅ 使用 `GetClientIp()` 記錄 IP
- ✅ 正確的錯誤碼
- ✅ 商業邏輯完整 (不能取消已開始的預約、只能取消自己的)

**我只需要調整**:
- 錯誤碼對應公司的標準 (例如 "40100" 改成 "9AU01")
- 補充「管理員可看所有預約,一般使用者只能看自己的」邏輯

改了 5 行代碼就完成了。

## Step 6: 生成 Controller (10 分鐘)

```
@framework-guide 建立 ReservationController

路由: /api/reservation
需要實作所有 CRUD 端點,請加上 Swagger 註解
```

**AI 生成** (5 秒):

```csharp
[Route("api/reservation")]
public class ReservationController : SystemBaseController
{
    private readonly IReservationService _reservationService;

    public ReservationController(IReservationService reservationService)
    {
        _reservationService = reservationService;
    }

    /// <summary>
    /// 查詢預約列表
    /// </summary>
    [HttpGet]
    [ProducesResponseType(StatusCodes.Status200OK, Type = typeof(BaseSystemResponse<PageResult<GetReservationRes>>))]
    public async Task<IActionResult> GetReservationsAsync([FromQuery] GetReservationReq req)
    {
        var res = await _reservationService.GetReservationsAsync(req);
        return HandleResult(res);
    }

    /// <summary>
    /// 取得單一預約
    /// </summary>
    [HttpGet("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetReservationByIdAsync(string id)
    {
        var res = await _reservationService.GetReservationByIdAsync(id);
        return HandleResult(res);
    }

    /// <summary>
    /// 建立預約
    /// </summary>
    [HttpPost]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> PostReservationAsync([FromBody] PostReservationReq req)
    {
        await _reservationService.PostReservationAsync(req);
        return HandleResult(new EmptyResult());
    }

    /// <summary>
    /// 取消預約
    /// </summary>
    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> CancelReservationAsync(string id, [FromQuery] string? reason)
    {
        await _reservationService.CancelReservationAsync(id, reason);
        return HandleResult(new EmptyResult());
    }

    /// <summary>
    /// 查詢可用時段
    /// </summary>
    [HttpGet("available-slots")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAvailableSlotsAsync([FromQuery] GetAvailableSlotsReq req)
    {
        var res = await _reservationService.GetAvailableSlotsAsync(req);
        return HandleResult(res);
    }
}
```

✅ 完美!Swagger 註解、HTTP 方法、路由、回應格式全對。

## Step 7: AutoMapper Profile (5 分鐘)

```
@framework-guide 建立 ReservationProfile

需要映射:
- Reservation -> GetReservationRes (包含 StationName)
```

**AI 生成**:

```csharp
public class ReservationProfile : Profile
{
    public ReservationProfile()
    {
        CreateMap<Reservation, GetReservationRes>()
            .ForMember(dest => dest.Id, opt => opt.MapFrom(src => src.ReservationId))
            .ForMember(dest => dest.StationName, opt => opt.MapFrom(src => src.Station.Name));
    }
}
```

✅ 正確!它知道 `StationName` 要從關聯的 `Station` entity 取得。

## Step 8: 註冊到 DI (2 分鐘)

```
@framework-guide 在 ServiceRegistration 註冊 ReservationService 和 ReservationRepository
```

**AI 生成**:

```csharp
services.AddScoped<IReservationService, ReservationService>();
services.AddScoped<IReservationRepository, ReservationRepository>();
```

直接複製貼上。

## Step 9: 單元測試 (30 分鐘)

這是我第一次讓 AI 生成測試:

```
@framework-guide 為 ReservationService 生成 XUnit 單元測試

需要測試:
1. PostReservationAsync - 成功建立預約
2. PostReservationAsync - 時段衝突應拋出例外
3. PostReservationAsync - 預約過去時段應拋出例外
4. CancelReservationAsync - 成功取消預約
5. CancelReservationAsync - 取消別人的預約應拋出例外
6. CancelReservationAsync - 取消已開始的預約應拋出例外

使用 Moq 模擬 Repository
```

**AI 生成** (15 秒):

```csharp
public class ReservationServiceTests
{
    private readonly Mock<IReservationRepository> _mockRepo;
    private readonly Mock<IHttpContextAccessor> _mockHttpContext;
    private readonly ReservationService _service;

    public ReservationServiceTests()
    {
        _mockRepo = new Mock<IReservationRepository>();
        _mockHttpContext = new Mock<IHttpContextAccessor>();
        
        // 模擬 JWT Token 中的使用者資訊
        var claims = new List<Claim>
        {
            new Claim("UserId", "test-user-123")
        };
        var identity = new ClaimsIdentity(claims, "TestAuth");
        var claimsPrincipal = new ClaimsPrincipal(identity);
        
        _mockHttpContext.Setup(x => x.HttpContext.User).Returns(claimsPrincipal);
        
        _service = new ReservationService(
            _mockHttpContext.Object,
            _mockRepo.Object,
            null // Mapper 暫時不需要
        );
    }

    [Fact]
    public async Task PostReservationAsync_Success()
    {
        // Arrange
        var req = new PostReservationReq
        {
            StationId = "station-1",
            StartTime = DateTime.UtcNow.AddHours(1),
            EndTime = DateTime.UtcNow.AddHours(2)
        };
        
        _mockRepo.Setup(x => x.HasConflictAsync(
            It.IsAny<string>(),
            It.IsAny<DateTime>(),
            It.IsAny<DateTime>(),
            null
        )).ReturnsAsync(false);

        // Act
        await _service.PostReservationAsync(req);

        // Assert
        _mockRepo.Verify(x => x.InsertAsync(It.IsAny<Reservation>()), Times.Once);
    }

    [Fact]
    public async Task PostReservationAsync_PastTime_ThrowsException()
    {
        // Arrange
        var req = new PostReservationReq
        {
            StationId = "station-1",
            StartTime = DateTime.UtcNow.AddHours(-1),
            EndTime = DateTime.UtcNow
        };

        // Act & Assert
        var exception = await Assert.ThrowsAsync<BaseException>(
            () => _service.PostReservationAsync(req)
        );
        Assert.Equal("不可預約過去的時段", exception.Message);
    }

    [Fact]
    public async Task PostReservationAsync_Conflict_ThrowsException()
    {
        // Arrange
        var req = new PostReservationReq
        {
            StationId = "station-1",
            StartTime = DateTime.UtcNow.AddHours(1),
            EndTime = DateTime.UtcNow.AddHours(2)
        };
        
        _mockRepo.Setup(x => x.HasConflictAsync(
            It.IsAny<string>(),
            It.IsAny<DateTime>(),
            It.IsAny<DateTime>(),
            null
        )).ReturnsAsync(true);

        // Act & Assert
        var exception = await Assert.ThrowsAsync<BaseException>(
            () => _service.PostReservationAsync(req)
        );
        Assert.Equal("此時段已被預約", exception.Message);
    }

    // ... 其他測試案例
}
```

✅ 測試結構正確、Moq 用法對、連 JWT Token 模擬都寫了!

我只需要補完剩下的測試案例 (AI 只生成了 3 個,我需要 6 個)。

## 總結:時間對比

| 任務 | 傳統方式 | 用 AI 輔助 | 節省時間 |
|------|---------|-----------|---------|
| Entity | 15 分鐘 | 5 分鐘 | 67% |
| DTOs | 30 分鐘 | 10 分鐘 | 67% |
| Repository | 45 分鐘 | 15 分鐘 | 67% |
| Service | 90 分鐘 | 25 分鐘 | 72% |
| Controller | 30 分鐘 | 10 分鐘 | 67% |
| AutoMapper | 10 分鐘 | 5 分鐘 | 50% |
| DI 註冊 | 5 分鐘 | 2 分鐘 | 60% |
| 單元測試 | 120 分鐘 | 40 分鐘 | 67% |
| **總計** | **345 分鐘 (5.75 小時)** | **112 分鐘 (1.87 小時)** | **67.5%** |

**實際效果**: 原本要花 **2 天 (16 小時)**,現在 **半天不到 (4 小時)** 就完成。

扣掉寫文件和測試的時間,**核心代碼生成只用了 1.5 小時**!

## 遇到的問題與解決

### 問題 1: AI 有時會「過度創新」

生成 Service 時,它加了一個「自動確認預約」的功能 (超過 15 分鐘未確認就自動取消)。

雖然很酷,但**需求裡沒寫**。

**解法**: 在 Prompt 加上「請嚴格按照需求實作,不要加額外功能」。

### 問題 2: 商業邏輯還是要人工驗證

AI 生成的「時段衝突檢查」邏輯是對的,但我發現一個 edge case:
- 假設有預約 10:00-11:00
- 新預約 11:00-12:00 會被擋下 (但其實不應該衝突)

**原因**: AI 用的是 `<` 和 `>`,應該用 `<=` 和 `>=`。

這種細節還是要人工 Review。

### 問題 3: 測試覆蓋率不夠

AI 只生成了「正向測試」和「基本錯誤測試」,但沒有:
- 邊界條件測試 (例如預約時段剛好 24 小時)
- 併發測試 (兩人同時預約同一時段)
- 效能測試

這些還是要手動補。

## 開發流程的改變

### 以前 (純手工)
1. 看需求 (30 分鐘)
2. 設計資料模型 (1 小時)
3. 寫 Entity → DTO → Repository → Service → Controller (8 小時)
4. 寫測試 (2 小時)
5. 手動測試 API (1 小時)
6. 修 Bug (2 小時)
7. Code Review (1 小時)
8. 部署 (30 分鐘)

**總計: 16 小時**

### 現在 (AI 輔助)
1. 看需求 (30 分鐘)
2. 用 AI 生成所有代碼 (1.5 小時)
3. **人工 Review 和調整商業邏輯** (1.5 小時) ← 最重要!
4. 補充邊界測試 (30 分鐘)
5. 手動測試 API (30 分鐘)
6. 修 Bug (30 分鐘)
7. Code Review (30 分鐘)
8. 部署 (30 分鐘)

**總計: 6 小時**

**節省時間: 62.5%**

## 關鍵心得

### 1. AI 不是「取代」開發者,是「加速」開發

AI 很擅長:
- ✅ 生成樣板代碼 (Entity, DTO, Controller)
- ✅ 實作標準 CRUD 邏輯
- ✅ 遵守框架規範
- ✅ 寫基本單元測試

AI 不擅長:
- ❌ 理解複雜的商業邏輯
- ❌ 處理邊界情況和 edge cases
- ❌ 效能優化 (需要實際測試)
- ❌ 安全性考量 (例如 SQL Injection, XSS)

**最佳實踐**: 用 AI 生成 80% 的代碼,人工處理剩下 20% 的關鍵邏輯。

### 2. 框架文件是關鍵

如果沒有前三週的準備 (框架文件 + RAG + Prompt 優化),AI 生成的代碼品質會差很多。

**投資報酬率**: 花 3 週建立框架知識庫,換來長期 60%+ 的效率提升。

### 3. 人工 Review 不可省略

雖然 AI 生成的代碼 95% 正確,但那 5% 的錯誤可能是致命的:
- 邏輯判斷錯誤
- 安全漏洞
- 效能問題

**務必做 Code Review**,不能盲目相信 AI。

## 下一步計畫

這個實驗成功後,我計畫:

1. **擴展框架文件**
   - 加入更多「常見錯誤範例」
   - 補充「效能優化模式」
   - 新增「安全性檢查清單」

2. **建立 AI Code Reviewer**
   - 讓 AI 自動檢查 PR 是否符合框架規範
   - 自動標記潛在問題
   - 給出改進建議

3. **團隊推廣**
   - 建立 Docker Image,讓團隊成員快速安裝
   - 寫一份「AI 輔助開發指南」
   - 辦內部 Workshop 分享經驗

4. **持續優化**
   - 收集更多訓練資料 (目標 1000+ 組範例)
   - 每季重新 Fine-tune 模型
   - 追蹤團隊使用 AI 後的效率提升數據

## 四週總結

這一個月的實驗,從「建立框架文件」到「實戰生成完整模組」,證明了:**AI 可以真正融入日常開發流程**。

關鍵成功因素:
1. ✅ 完整的框架文件 (6000+ 行)
2. ✅ RAG 知識庫 (快速檢索相關規範)
3. ✅ 優化的 Prompt (System Message + Few-Shot)
4. ✅ Fine-tuning (讓 AI 深度理解公司規範)

成果:
- **開發時間縮短 60-70%**
- **代碼品質更一致** (都符合框架規範)
- **新人 Onboarding 更快** (有完整文件 + AI 輔助)

但也要記住:**AI 是工具,不是魔法**。它能幫你寫樣板代碼,但**關鍵的商業邏輯、架構設計、效能優化還是要靠人**。

未來的開發者,需要的不是「寫最快的代碼」,而是:
- **清楚表達需求** (Prompt Engineering)
- **快速驗證和審查代碼**
- **理解複雜的商業邏輯**
- **做出正確的架構決策**

這次實驗只是開始,AI 輔助開發的路還很長。但我相信,掌握這套方法的團隊,會在競爭中佔據優勢。
