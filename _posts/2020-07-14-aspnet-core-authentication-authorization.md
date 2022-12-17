---
layout: post
title: "ASP.NET Core 認證與授權基礎"
date: 2020-07-14 15:20:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, Authentication, Authorization, Security]
---

這週研究 ASP.NET Core 的認證與授權機制，它提供完整的安全框架。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## Spring Security 回顧

在 Spring Boot 中，我們使用 Spring Security：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .formLogin()
            .and()
            .httpBasic();
    }
}
```

## 認證 vs 授權

- **認證 (Authentication)**：驗證使用者身分
- **授權 (Authorization)**：驗證使用者權限

## Cookie 認證

### 設定 Cookie 認證

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie(options =>
        {
            options.LoginPath = "/Account/Login";
            options.LogoutPath = "/Account/Logout";
            options.AccessDeniedPath = "/Account/AccessDenied";
            options.ExpireTimeSpan = TimeSpan.FromHours(2);
            options.SlidingExpiration = true;
            options.Cookie.HttpOnly = true;
            options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        });

    services.AddAuthorization();
    services.AddControllers();
}

public void Configure(IApplicationBuilder app)
{
    app.UseRouting();
    
    app.UseAuthentication(); // 必須在 UseAuthorization 之前
    app.UseAuthorization();
    
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### 登入邏輯

```csharp
[ApiController]
[Route("api/[controller]")]
public class AccountController : ControllerBase
{
    private readonly IUserService _userService;

    public AccountController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        var user = await _userService.ValidateUserAsync(
            request.Username, request.Password);
        
        if (user == null)
        {
            return Unauthorized(new { message = "帳號或密碼錯誤" });
        }

        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim(ClaimTypes.Role, user.Role)
        };

        var claimsIdentity = new ClaimsIdentity(
            claims, CookieAuthenticationDefaults.AuthenticationScheme);

        var authProperties = new AuthenticationProperties
        {
            IsPersistent = request.RememberMe,
            ExpiresUtc = DateTimeOffset.UtcNow.AddHours(2)
        };

        await HttpContext.SignInAsync(
            CookieAuthenticationDefaults.AuthenticationScheme,
            new ClaimsPrincipal(claimsIdentity),
            authProperties);

        return Ok(new { message = "登入成功", username = user.Username });
    }

    [HttpPost("logout")]
    public async Task<IActionResult> Logout()
    {
        await HttpContext.SignOutAsync(
            CookieAuthenticationDefaults.AuthenticationScheme);
        
        return Ok(new { message = "登出成功" });
    }
}
```

## JWT 認證

### 安裝套件

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 設定 JWT

```csharp
// appsettings.json
{
  "JwtSettings": {
    "Secret": "YourSuperSecretKeyThatIsAtLeast32CharactersLong!",
    "Issuer": "MyShopApi",
    "Audience": "MyShopClient",
    "ExpirationMinutes": 60
  }
}

// JwtSettings.cs
public class JwtSettings
{
    public string Secret { get; set; }
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public int ExpirationMinutes { get; set; }
}

// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    var jwtSettings = Configuration.GetSection("JwtSettings").Get<JwtSettings>();
    var key = Encoding.ASCII.GetBytes(jwtSettings.Secret);

    services.Configure<JwtSettings>(Configuration.GetSection("JwtSettings"));

    services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(options =>
    {
        options.RequireHttpsMetadata = false;
        options.SaveToken = true;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = true,
            ValidIssuer = jwtSettings.Issuer,
            ValidateAudience = true,
            ValidAudience = jwtSettings.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });

    services.AddAuthorization();
    services.AddControllers();
}
```

### JWT 服務

```csharp
public interface IJwtService
{
    string GenerateToken(User user);
    ClaimsPrincipal ValidateToken(string token);
}

public class JwtService : IJwtService
{
    private readonly JwtSettings _jwtSettings;

    public JwtService(IOptions<JwtSettings> jwtSettings)
    {
        _jwtSettings = jwtSettings.Value;
    }

    public string GenerateToken(User user)
    {
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Role, user.Role)
        };

        var key = new SymmetricSecurityKey(
            Encoding.ASCII.GetBytes(_jwtSettings.Secret));
        
        var credentials = new SigningCredentials(
            key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public ClaimsPrincipal ValidateToken(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.ASCII.GetBytes(_jwtSettings.Secret);

        var parameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = true,
            ValidIssuer = _jwtSettings.Issuer,
            ValidateAudience = true,
            ValidAudience = _jwtSettings.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };

        return tokenHandler.ValidateToken(token, parameters, out _);
    }
}
```

### JWT 登入

```csharp
[HttpPost("login")]
public async Task<IActionResult> Login([FromBody] LoginRequest request)
{
    var user = await _userService.ValidateUserAsync(
        request.Username, request.Password);
    
    if (user == null)
    {
        return Unauthorized(new { message = "帳號或密碼錯誤" });
    }

    var token = _jwtService.GenerateToken(user);

    return Ok(new
    {
        token,
        tokenType = "Bearer",
        expiresIn = _jwtSettings.ExpirationMinutes * 60,
        user = new
        {
            user.Id,
            user.Username,
            user.Email,
            user.Role
        }
    });
}
```

使用 Token：

```bash
curl -H "Authorization: Bearer <token>" \
  https://localhost:5001/api/products
```

## 授權

### 使用 [Authorize] 屬性

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 所有使用者都可存取
    [HttpGet]
    [AllowAnonymous]
    public IActionResult GetAll()
    {
        return Ok("公開資料");
    }

    // 需要認證
    [HttpGet("{id}")]
    [Authorize]
    public IActionResult GetById(int id)
    {
        return Ok($"產品 {id}");
    }

    // 需要特定角色
    [HttpPost]
    [Authorize(Roles = "Admin")]
    public IActionResult Create([FromBody] Product product)
    {
        return Ok("建立成功");
    }

    // 多個角色（OR）
    [HttpPut("{id}")]
    [Authorize(Roles = "Admin,Manager")]
    public IActionResult Update(int id, [FromBody] Product product)
    {
        return Ok("更新成功");
    }

    // 需要所有角色（AND）
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    [Authorize(Roles = "SuperUser")]
    public IActionResult Delete(int id)
    {
        return Ok("刪除成功");
    }
}
```

Spring Security 對應：

```java
@RestController
@RequestMapping("/api/products")
public class ProductsController {
    
    @GetMapping
    public ResponseEntity<?> getAll() {
        return ResponseEntity.ok("公開資料");
    }
    
    @GetMapping("/{id}")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<?> getById(@PathVariable Long id) {
        return ResponseEntity.ok("產品 " + id);
    }
    
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<?> create(@RequestBody Product product) {
        return ResponseEntity.ok("建立成功");
    }
}
```

### 授權策略 (Policy)

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthorization(options =>
    {
        // 年齡限制
        options.AddPolicy("Adult", policy =>
            policy.RequireClaim("DateOfBirth")
                  .RequireAssertion(context =>
                  {
                      var dob = DateTime.Parse(
                          context.User.FindFirst("DateOfBirth").Value);
                      return DateTime.Today.Year - dob.Year >= 18;
                  }));

        // 需要特定權限
        options.AddPolicy("CanEditProduct", policy =>
            policy.RequireClaim("Permission", "Product.Edit"));

        // 多重要求
        options.AddPolicy("SeniorAdmin", policy =>
            policy.RequireRole("Admin")
                  .RequireClaim("Seniority", "Senior")
                  .RequireAuthenticatedUser());

        // 自訂要求
        options.AddPolicy("CanAccessReport", policy =>
            policy.Requirements.Add(new ReportAccessRequirement()));
    });
}

// Controller 使用
[HttpGet("sensitive")]
[Authorize(Policy = "Adult")]
public IActionResult GetSensitiveContent()
{
    return Ok("限制級內容");
}

[HttpPut("products/{id}")]
[Authorize(Policy = "CanEditProduct")]
public IActionResult UpdateProduct(int id, Product product)
{
    return Ok("更新成功");
}
```

### 自訂授權要求

```csharp
// 定義要求
public class ReportAccessRequirement : IAuthorizationRequirement
{
    public string Department { get; set; }
}

// 實作處理器
public class ReportAccessHandler : AuthorizationHandler<ReportAccessRequirement>
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public ReportAccessHandler(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ReportAccessRequirement requirement)
    {
        var user = context.User;

        // 檢查是否為 Admin
        if (user.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // 檢查部門
        var department = user.FindFirst("Department")?.Value;
        if (department == "Finance" || department == "Management")
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // 檢查工作時間
        var currentHour = DateTime.Now.Hour;
        if (currentHour >= 9 && currentHour < 18)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// 註冊
services.AddSingleton<IAuthorizationHandler, ReportAccessHandler>();
```

### 資源授權

```csharp
// 定義要求
public class DocumentOwnerRequirement : IAuthorizationRequirement { }

// 實作處理器
public class DocumentOwnerHandler 
    : AuthorizationHandler<DocumentOwnerRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentOwnerRequirement requirement,
        Document document)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (document.OwnerId == int.Parse(userId))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Controller 使用
[HttpPut("documents/{id}")]
public async Task<IActionResult> UpdateDocument(
    int id, 
    [FromBody] DocumentDto dto,
    [FromServices] IAuthorizationService authService)
{
    var document = await _documentService.GetByIdAsync(id);
    
    if (document == null)
        return NotFound();

    var authResult = await authService.AuthorizeAsync(
        User, document, "DocumentOwner");

    if (!authResult.Succeeded)
        return Forbid();

    await _documentService.UpdateAsync(document);
    return Ok();
}
```

## 取得目前使用者資訊

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ProfileController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProfile()
    {
        // 方式一：使用 User 屬性
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var username = User.Identity.Name;
        var email = User.FindFirst(ClaimTypes.Email)?.Value;
        var roles = User.Claims
            .Where(c => c.Type == ClaimTypes.Role)
            .Select(c => c.Value)
            .ToList();

        // 方式二：檢查角色
        var isAdmin = User.IsInRole("Admin");

        // 方式三：檢查 Claim
        var hasPermission = User.HasClaim("Permission", "Product.Edit");

        return Ok(new
        {
            userId,
            username,
            email,
            roles,
            isAdmin
        });
    }
}
```

### 使用服務取得使用者

```csharp
public interface ICurrentUserService
{
    int GetUserId();
    string GetUsername();
    string GetEmail();
    bool IsInRole(string role);
}

public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public int GetUserId()
    {
        var userId = _httpContextAccessor.HttpContext?.User
            .FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        return int.Parse(userId ?? "0");
    }

    public string GetUsername()
    {
        return _httpContextAccessor.HttpContext?.User.Identity?.Name;
    }

    public string GetEmail()
    {
        return _httpContextAccessor.HttpContext?.User
            .FindFirst(ClaimTypes.Email)?.Value;
    }

    public bool IsInRole(string role)
    {
        return _httpContextAccessor.HttpContext?.User.IsInRole(role) ?? false;
    }
}

// 註冊
services.AddHttpContextAccessor();
services.AddScoped<ICurrentUserService, CurrentUserService>();
```

## 密碼雜湊

```csharp
public interface IPasswordHasher
{
    string HashPassword(string password);
    bool VerifyPassword(string password, string hash);
}

public class PasswordHasher : IPasswordHasher
{
    public string HashPassword(string password)
    {
        return BCrypt.Net.BCrypt.HashPassword(password);
    }

    public bool VerifyPassword(string password, string hash)
    {
        return BCrypt.Net.BCrypt.Verify(password, hash);
    }
}

// 使用
var hashedPassword = _passwordHasher.HashPassword("mypassword");
var isValid = _passwordHasher.VerifyPassword("mypassword", hashedPassword);
```

需要安裝套件：

```bash
dotnet add package BCrypt.Net-Next
```

## Refresh Token

```csharp
public class RefreshToken
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public string Token { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedByIp { get; set; }
    public DateTime? RevokedAt { get; set; }
    public string RevokedByIp { get; set; }
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    public bool IsActive => RevokedAt == null && !IsExpired;
}

public class TokenResponse
{
    public string AccessToken { get; set; }
    public string RefreshToken { get; set; }
    public int ExpiresIn { get; set; }
}

[HttpPost("refresh")]
public async Task<IActionResult> RefreshToken([FromBody] RefreshTokenRequest request)
{
    var refreshToken = await _tokenService.GetRefreshTokenAsync(request.RefreshToken);
    
    if (refreshToken == null || !refreshToken.IsActive)
    {
        return Unauthorized(new { message = "Invalid refresh token" });
    }

    var user = await _userService.GetByIdAsync(refreshToken.UserId);
    
    // 產生新的 token
    var newAccessToken = _jwtService.GenerateToken(user);
    var newRefreshToken = await _tokenService.GenerateRefreshTokenAsync(
        user.Id, GetIpAddress());
    
    // 撤銷舊的 refresh token
    await _tokenService.RevokeRefreshTokenAsync(
        refreshToken.Token, GetIpAddress());

    return Ok(new TokenResponse
    {
        AccessToken = newAccessToken,
        RefreshToken = newRefreshToken.Token,
        ExpiresIn = _jwtSettings.ExpirationMinutes * 60
    });
}
```

## 小結

ASP.NET Core 的認證與授權：
- 支援多種認證方式（Cookie、JWT、OAuth）
- 授權策略靈活強大
- Claims-based 授權
- 與依賴注入完美整合

相較於 Spring Security：
- 設定更簡潔
- Policy-based 授權更直覺
- JWT 實作更輕量
- 不需要額外的 Filter Chain 設定

下週將探討 EF Core 的效能優化技巧。
