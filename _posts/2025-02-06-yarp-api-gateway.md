---
layout: post
title: "為什麼我們用 YARP 自建 API Gateway 而不用 Azure APIM"
date: 2025-02-06 11:20:00 +0800
categories: [架構設計]
tags: [YARP, API Gateway, .NET Core, Azure APIM]
---

## Azure API Management 太貴了

當初在規劃架構的時候，理所當然地想到要用 API Gateway。

畢竟我們有多個模組（會員、點數、優惠券⋯⋯），未來可能會拆成獨立服務，需要一個統一的入口來做 routing、authentication、rate limiting 這些事。

Azure 官方的方案是 **API Management (APIM)**，功能超級完整：
- API routing 和 versioning
- 認證和授權
- Rate limiting 和 quota
- Request/Response 轉換
- 監控和分析
- Developer portal

看起來很棒，對吧？

然後我看了一下價格：

- **Developer tier**: $50/月（只能測試用，不能用在 production）
- **Basic tier**: $150/月（500萬次 API 呼叫）
- **Standard tier**: $730/月（1000萬次 API 呼叫）
- **Premium tier**: $2900/月起跳

我們預估的 API 呼叫量大概每個月 1000 萬次左右，所以至少要 Standard tier。

一個月 $730 美金，一年就要 $8760。

我去找主管討論：「這個 APIM 一年要 27 萬台幣耶⋯⋯」

主管皺眉：「這麼貴？有沒有其他選擇？」

## 自建 API Gateway 的考量

其實市面上有不少開源的 API Gateway：
- **Kong**：很成熟，但比較重，而且要跑 PostgreSQL
- **Traefik**：輕量，但設定語法要學
- **Nginx**：萬用工具，但做複雜邏輯比較麻煩

我想了想：我們全部都用 .NET Core，團隊對 C# 很熟，為什麼不用 .NET 來做？

然後我發現了 **YARP**（Yet Another Reverse Proxy）。

## YARP 是什麼？

YARP 是 Microsoft 開發的開源 reverse proxy library，專門給 .NET 應用使用。

它的特色是：
- **輕量**：基於 ASP.NET Core，效能很好
- **靈活**：用 C# 寫邏輯，想怎麼改就怎麼改
- **整合度高**：跟 .NET 的生態系無縫整合（DI、middleware、configuration）
- **免費**：MIT License，愛怎麼用就怎麼用

最重要的是：**我們不用學新的工具或語法，就用我們熟悉的 C# 就好**。

## 第一版：基礎 routing

先來看最簡單的用法。

安裝 YARP：

```bash
dotnet add package Yarp.ReverseProxy
```

設定檔（appsettings.json）：

```json
{
  "ReverseProxy": {
    "Routes": {
      "member-route": {
        "ClusterId": "member-cluster",
        "Match": {
          "Path": "/api/members/{**catch-all}"
        }
      },
      "points-route": {
        "ClusterId": "points-cluster",
        "Match": {
          "Path": "/api/points/{**catch-all}"
        }
      },
      "coupon-route": {
        "ClusterId": "coupon-cluster",
        "Match": {
          "Path": "/api/coupons/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "member-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://member-service.internal"
          }
        }
      },
      "points-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://points-service.internal"
          }
        }
      },
      "coupon-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://coupon-service.internal"
          }
        }
      }
    }
  }
}
```

Program.cs：

```csharp
var builder = WebApplication.CreateBuilder(args);

// 加入 YARP
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

// 使用 YARP
app.MapReverseProxy();

app.Run();
```

就這樣，一個基礎的 API Gateway 就做好了。

請求 `/api/members/123` 會被轉發到 member-service，`/api/points/` 會轉發到 points-service。

## 加上認證

當然，我們不能讓所有人都能呼叫 API。需要加上 JWT 認證。

好消息：YARP 可以無縫整合 ASP.NET Core 的 authentication middleware。

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://auth.mydomain.com";
        options.Audience = "myapi";
    });

builder.Services.AddAuthorization();

// ... 

app.UseAuthentication();
app.UseAuthorization();
app.MapReverseProxy();
```

如果想要針對特定 route 做授權，可以用 `AuthorizationPolicy`：

```json
{
  "Routes": {
    "admin-route": {
      "ClusterId": "admin-cluster",
      "Match": {
        "Path": "/api/admin/{**catch-all}"
      },
      "AuthorizationPolicy": "AdminOnly"
    }
  }
}
```

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
});
```

就這麼簡單。不用寫一堆 Nginx 或 Kong 的設定檔。

## Rate Limiting

APIM 有個很好用的功能是 rate limiting（限流），避免 API 被打爆。

.NET 7 之後內建了 Rate Limiting middleware，可以直接用：

```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100; // 每分鐘最多 100 次請求
        opt.QueueLimit = 0;
    });
});

// ...

app.UseRateLimiter();
app.MapReverseProxy();
```

如果要針對不同 route 設定不同的限流規則：

```csharp
builder.Services.AddRateLimiter(options =>
{
    // 一般 API：每分鐘 100 次
    options.AddFixedWindowLimiter("general", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
    });
    
    // 註冊 API：每分鐘 10 次（防濫用）
    options.AddFixedWindowLimiter("register", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 10;
    });
});
```

在 route 設定中指定：

```json
{
  "Routes": {
    "register-route": {
      "ClusterId": "member-cluster",
      "Match": {
        "Path": "/api/members/register"
      },
      "RateLimiterPolicy": "register"
    },
    "general-route": {
      "ClusterId": "member-cluster",
      "Match": {
        "Path": "/api/members/{**catch-all}"
      },
      "RateLimiterPolicy": "general"
    }
  }
}
```

## 客製化邏輯：Transform

有時候我們需要修改 request 或 response，YARP 提供了 Transform 機制。

舉例：在所有 request header 加上 `X-Gateway-Version`：

```csharp
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(context =>
    {
        context.AddRequestTransform(async transformContext =>
        {
            transformContext.ProxyRequest.Headers.Add(
                "X-Gateway-Version", "1.0");
        });
    });
```

或者更複雜的例子：根據使用者的會員等級，加上不同的 header：

```csharp
.AddTransforms(context =>
{
    context.AddRequestTransform(async transformContext =>
    {
        var user = transformContext.HttpContext.User;
        var memberLevel = user.FindFirst("MemberLevel")?.Value ?? "Basic";
        
        transformContext.ProxyRequest.Headers.Add(
            "X-Member-Level", memberLevel);
    });
});
```

後端服務就可以根據這個 header 做不同的處理，而不用重複去查詢會員資料。

## 監控和日誌

YARP 整合了 ASP.NET Core 的 logging 和 telemetry。

如果你已經在用 Application Insights，YARP 的 metrics 會自動送過去：
- Request count
- Response time
- Error rate

也可以自己寫 middleware 記錄更多資訊：

```csharp
app.Use(async (context, next) =>
{
    var stopwatch = Stopwatch.StartNew();
    var path = context.Request.Path;
    
    await next();
    
    stopwatch.Stop();
    
    _logger.LogInformation(
        "Request {Path} took {ElapsedMs}ms, Status: {StatusCode}",
        path, stopwatch.ElapsedMilliseconds, context.Response.StatusCode);
});

app.MapReverseProxy();
```

這樣就可以分析哪些 API 比較慢、哪些 route 的錯誤率比較高。

## Health Check

YARP 支援 health check，可以自動偵測 backend service 是否正常。

```json
{
  "Clusters": {
    "member-cluster": {
      "HealthCheck": {
        "Active": {
          "Enabled": true,
          "Interval": "00:00:30",
          "Timeout": "00:00:10",
          "Policy": "ConsecutiveFailures",
          "Path": "/health"
        }
      },
      "Destinations": {
        "destination1": {
          "Address": "https://member-service-1.internal",
          "Health": "https://member-service-1.internal/health"
        },
        "destination2": {
          "Address": "https://member-service-2.internal",
          "Health": "https://member-service-2.internal/health"
        }
      }
    }
  }
}
```

YARP 會每 30 秒打一次 `/health` endpoint。如果某個 destination 掛了，就會自動把流量導到其他正常的 destination。

## 效能

這是我很在意的一點。自建 API Gateway 會不會變成瓶頸？

我們做了簡單的壓測（用 k6）：

- **直接呼叫 backend service**：1200 req/s，平均延遲 80ms
- **透過 YARP**：1150 req/s，平均延遲 85ms

幾乎沒有 overhead。

這是因為 YARP 是用 `HttpClient` 直接 proxy 請求，沒有多餘的序列化/反序列化過程。

而且 YARP 本身就是 ASP.NET Core 應用，可以跑在 App Service 上，scaling 也很容易。

## 跟 APIM 比起來

當然，YARP 不是完美的。跟 APIM 比起來，少了：

- **Developer Portal**：沒有內建的 API 文件和測試介面（不過我們用 Swagger 也夠了）
- **Analytics Dashboard**：沒有漂亮的圖表（但可以用 Application Insights）
- **Policy Expression Language**：APIM 有自己的 policy language，寫起來比較簡潔（但我們用 C# 更靈活）
- **Managed Service**：APIM 是完全託管的，YARP 要自己維護

但這些對我們來說都不是問題。我們要的只是：
- Routing
- Authentication/Authorization
- Rate limiting
- 一些基本的 transform

這些 YARP 都做得到，而且更靈活。

## 成本比較

**APIM Standard tier**：$730/月

**YARP**：
- App Service (P1V3): $175/月
- 開發和維護成本：難以量化，但我們都在用 .NET，沒有額外學習成本

省了 **$550/月**，一年省下 **$6600**。

更重要的是，我們有完全的控制權。想加什麼功能就加，不用受限於 APIM 的框架。

## 心得

YARP 讓我重新思考了「API Gateway」的定義。

大家習慣把 API Gateway 想成一個「產品」：Kong、APIM、AWS API Gateway⋯⋯都是現成的方案，功能很多，但也很貴，而且不見得所有功能都用得到。

其實 API Gateway 的核心就是 reverse proxy + 一些額外的邏輯（認證、限流、轉換）。

YARP 把 reverse proxy 做成 library，讓你可以用熟悉的程式語言（C#）來寫這些邏輯。不用學新的設定語法，不用花大錢買商業方案。

當然，這不代表 YARP 適合所有情境。如果你的團隊不熟 .NET，或者真的需要 APIM 的進階功能（像 policy expression、developer portal），那還是買 APIM 吧。

但如果你跟我們一樣，只是需要一個簡單、便宜、可控的 API Gateway，YARP 是個很棒的選擇。

## 期待與挑戰

YARP Gateway 的開發已經完成，接下來就是上線測試了。

這個決定能不能成功，九個月後就知道了：
- 成本能省多少（預估從 $730 降到 $175）
- 效能是否穩定
- 維護成本會不會太高

風險當然有，但我相信這是目前最務實的選擇。

如果你也在考慮 API Gateway 的方案，預算有限又想要彈性，YARP 絕對值得評估看看。

下週會分享我們如何整合 .NET Core 後端和 React 前端，以及部署流程的設計。
