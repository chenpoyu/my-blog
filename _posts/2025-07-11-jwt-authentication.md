---
layout: post
title: "API 認證：JWT 的實作細節"
date: 2025-07-11 14:15:00 +0800
categories: [安全性]
tags: [JWT, API Security, 認證授權]
---

## 為什麼選 JWT 不選 Session

在規劃 API 認證的時候，團隊討論了兩個方案：

### 方案 1: Session-based

```
1. 使用者登入
2. Server 產生 session ID，存到 Redis
3. 回傳 session ID 給 client（放在 cookie）
4. 之後每次請求都帶 session ID
5. Server 從 Redis 查詢 session 資料
```

優點：
- Server 可以隨時撤銷 session
- 安全性較高（secret 不會暴露在 client）

缺點：
- **每次請求都要查 Redis**（增加延遲）
- **無法跨 domain**（cookie 有 same-site 限制）
- **難以水平擴展**（雖然可以用 Redis，但還是增加複雜度）

### 方案 2: JWT (JSON Web Token)

```
1. 使用者登入
2. Server 產生 JWT，包含使用者資訊
3. 回傳 JWT 給 client
4. 之後每次請求都帶 JWT
5. Server 驗證 JWT 的簽章（不用查資料庫）
```

優點：
- **不用查資料庫**（驗證只需要檢查簽章）
- **可以跨 domain**（JWT 放在 Authorization header）
- **容易水平擴展**（stateless）

缺點：
- **無法主動撤銷**（除非有 blacklist）
- **Token 可能包含敏感資料**（要小心）
- **Token 太大會增加每次請求的大小**

我們最後選了 **JWT**，因為：
1. 我們已經有 Redis cache，不想每次都查
2. 前端是 SPA（React），需要跨 domain 呼叫 API
3. 系統設計是 stateless，符合微模組架構

## JWT 的結構

JWT 由三部分組成，用 `.` 分隔：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJNMDAxIiwibmFtZSI6IkpvaG4iLCJleHAiOjE3MTkxMjM0NTZ9.xyz123abc456
[          Header          ].[              Payload              ].[  Signature  ]
```

### Header

```json
{
  "alg": "HS256",  // 簽章演算法
  "typ": "JWT"     // Token 類型
}
```

### Payload

```json
{
  "sub": "M001",              // Subject（使用者 ID）
  "name": "John Doe",         // 使用者名稱
  "email": "john@example.com",
  "level": "Gold",            // 會員等級
  "iat": 1719120000,          // Issued At（發行時間）
  "exp": 1719123600           // Expiration（過期時間）
}
```

### Signature

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

## 實作：產生 JWT

安裝套件：

```bash
dotnet add package System.IdentityModel.Tokens.Jwt
```

建立 JWT service：

```csharp
public class JwtService
{
    private readonly string _secretKey;
    private readonly string _issuer;
    private readonly int _expirationMinutes;
    
    public JwtService(IConfiguration configuration)
    {
        _secretKey = configuration["Jwt:SecretKey"];
        _issuer = configuration["Jwt:Issuer"];
        _expirationMinutes = int.Parse(configuration["Jwt:ExpirationMinutes"]);
    }
    
    public string GenerateToken(MemberDto member)
    {
        var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_secretKey));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);
        
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, member.Id),
            new Claim(JwtRegisteredClaimNames.Name, member.Name),
            new Claim(JwtRegisteredClaimNames.Email, member.Email),
            new Claim("level", member.Level),  // Custom claim
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };
        
        var token = new JwtSecurityToken(
            issuer: _issuer,
            audience: _issuer,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_expirationMinutes),
            signingCredentials: credentials
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

登入 API：

```csharp
[HttpPost("login")]
public async Task<IActionResult> Login([FromBody] LoginRequest request)
{
    var member = await _memberService.ValidateCredentialsAsync(
        request.Email, 
        request.Password
    );
    
    if (member == null)
    {
        return Unauthorized(new { message = "Invalid credentials" });
    }
    
    var token = _jwtService.GenerateToken(member);
    
    return Ok(new 
    { 
        token,
        expiresIn = 3600  // 1 小時
    });
}
```

## 實作：驗證 JWT

在 `Program.cs` 中設定 JWT authentication：

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Issuer"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"])
            ),
            ClockSkew = TimeSpan.Zero  // 不允許時間偏移
        };
    });

// 記得加上 authentication middleware
app.UseAuthentication();
app.UseAuthorization();
```

在需要認證的 API 加上 `[Authorize]`：

```csharp
[Authorize]
[HttpGet("profile")]
public async Task<IActionResult> GetProfile()
{
    // 從 JWT 中取得使用者 ID
    var memberId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    
    var member = await _memberService.GetMemberAsync(memberId);
    return Ok(member);
}
```

## Token 過期時間該設多久？

這是個很常見的問題。

| 時間 | 優點 | 缺點 |
|-----|------|------|
| 15 分鐘 | 安全性高 | 使用者常常要重新登入 |
| 1 小時 | 平衡 | 中等風險 |
| 1 天 | 使用體驗好 | Token 被盜風險高 |
| 永久 | 最方便 | 非常不安全 |

我們最後選了 **1 小時 + Refresh Token**。

## Refresh Token

流程：

```
1. 登入時，產生兩個 token：
   - Access Token（1 小時過期）
   - Refresh Token（7 天過期）

2. Access Token 過期後，用 Refresh Token 換新的 Access Token

3. Refresh Token 過期後，要重新登入
```

實作：

```csharp
public class JwtService
{
    public (string accessToken, string refreshToken) GenerateTokens(MemberDto member)
    {
        var accessToken = GenerateToken(member, expiresInMinutes: 60);
        var refreshToken = GenerateToken(member, expiresInMinutes: 7 * 24 * 60);  // 7 天
        
        // 把 refresh token 存到 Redis，方便撤銷
        await _cache.SetStringAsync(
            $"refresh_token:{member.Id}",
            refreshToken,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromDays(7)
            }
        );
        
        return (accessToken, refreshToken);
    }
}
```

Refresh API：

```csharp
[HttpPost("refresh")]
public async Task<IActionResult> Refresh([FromBody] RefreshRequest request)
{
    // 驗證 refresh token
    var principal = ValidateToken(request.RefreshToken);
    if (principal == null)
    {
        return Unauthorized(new { message = "Invalid refresh token" });
    }
    
    var memberId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    
    // 檢查 Redis 中的 refresh token 是否存在（沒被撤銷）
    var cachedToken = await _cache.GetStringAsync($"refresh_token:{memberId}");
    if (cachedToken != request.RefreshToken)
    {
        return Unauthorized(new { message = "Refresh token revoked" });
    }
    
    // 產生新的 access token
    var member = await _memberService.GetMemberAsync(memberId);
    var newAccessToken = _jwtService.GenerateToken(member);
    
    return Ok(new { token = newAccessToken });
}
```

## 如何撤銷 JWT？

JWT 的缺點是：**無法主動撤銷**。

但我們可以用 **Token Blacklist** 實作撤銷：

```csharp
[Authorize]
[HttpPost("logout")]
public async Task<IActionResult> Logout()
{
    var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
    var jti = User.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
    
    // 把 token 的 JTI 加到 blacklist（Redis）
    await _cache.SetStringAsync(
        $"blacklist:{jti}",
        "1",
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)  // Token 過期時間
        }
    );
    
    return Ok(new { message = "Logged out" });
}
```

然後在 JWT middleware 中檢查 blacklist：

```csharp
options.Events = new JwtBearerEvents
{
    OnTokenValidated = async context =>
    {
        var cache = context.HttpContext.RequestServices
            .GetRequiredService<IDistributedCache>();
        
        var jti = context.Principal.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
        var blacklisted = await cache.GetStringAsync($"blacklist:{jti}");
        
        if (blacklisted != null)
        {
            context.Fail("Token has been revoked");
        }
    }
};
```

## 安全性注意事項

### 1. Secret Key 要夠長

```csharp
// ❌ 不要這樣
"Jwt:SecretKey": "mysecret"

// ✅ 至少 32 bytes
"Jwt:SecretKey": "ThisIsAVeryLongSecretKeyWith32BytesOrMore123456"
```

而且這個 key 要放在 **Key Vault**，不要寫在 appsettings.json。

### 2. 不要把敏感資料放在 JWT

JWT 的 payload 可以被 decode（雖然有簽章，但內容是 base64，不是加密）。

```csharp
// ❌ 不要這樣
new Claim("password", member.Password)  // 不要放密碼！
new Claim("creditCard", member.CreditCard)  // 不要放信用卡號！

// ✅ 只放必要的資訊
new Claim(JwtRegisteredClaimNames.Sub, member.Id)
new Claim("level", member.Level)
```

### 3. 使用 HTTPS

JWT 透過 HTTP 傳輸，**一定要用 HTTPS**。

不然 token 會被攔截。

### 4. 設定 CORS

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("https://www.example.com")  // 只允許特定 domain
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

app.UseCors("AllowFrontend");
```

## 下週：DR Plan

API 認證搞定了，下週要規劃 **Disaster Recovery（災難恢復）** 計畫。

如果 Azure 掛了，我們要怎麼快速恢復服務？

這是上線前最重要的準備之一。
