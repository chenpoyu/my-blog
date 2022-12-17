---
layout: post
title: "SignalR 即時通訊開發"
date: 2020-09-08 15:20:00 +0800
categories: [框架, .NET]
tags: [ASP.NET Core, SignalR, WebSocket, Real-time]
---

這週研究 ASP.NET Core SignalR 即時通訊框架，了解如何建立雙向即時通訊應用程式。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## SignalR 簡介

SignalR 是 ASP.NET Core 的即時通訊函式庫，提供伺服器與客戶端之間的雙向通訊：

主要功能：
- 自動選擇最佳傳輸方式（WebSocket、Server-Sent Events、Long Polling）
- 自動重新連線
- 群組管理
- 支援擴展到多伺服器

適用場景：
- 聊天應用程式
- 即時儀表板
- 協作工具
- 遊戲
- 即時通知

## 建立 SignalR Hub

### 安裝套件

```bash
dotnet add package Microsoft.AspNetCore.SignalR
```

### 定義 Hub

```csharp
using Microsoft.AspNetCore.SignalR;
using System.Threading.Tasks;

namespace SignalRDemo.Hubs
{
    public class ChatHub : Hub
    {
        // 發送訊息給所有客戶端
        public async Task SendMessage(string user, string message)
        {
            await Clients.All.SendAsync("ReceiveMessage", user, message);
        }

        // 發送訊息給特定使用者
        public async Task SendPrivateMessage(string toUser, string message)
        {
            await Clients.User(toUser).SendAsync("ReceiveMessage", Context.User.Identity.Name, message);
        }

        // 加入群組
        public async Task JoinGroup(string groupName)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
            await Clients.Group(groupName).SendAsync("UserJoined", Context.User.Identity.Name);
        }

        // 離開群組
        public async Task LeaveGroup(string groupName)
        {
            await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
            await Clients.Group(groupName).SendAsync("UserLeft", Context.User.Identity.Name);
        }

        // 發送訊息給群組
        public async Task SendToGroup(string groupName, string message)
        {
            await Clients.Group(groupName).SendAsync("ReceiveMessage", Context.User.Identity.Name, message);
        }

        // 連線時觸發
        public override async Task OnConnectedAsync()
        {
            var connectionId = Context.ConnectionId;
            var userName = Context.User?.Identity?.Name ?? "Anonymous";
            
            await Clients.All.SendAsync("UserConnected", userName, connectionId);
            await base.OnConnectedAsync();
        }

        // 斷線時觸發
        public override async Task OnDisconnectedAsync(Exception exception)
        {
            var userName = Context.User?.Identity?.Name ?? "Anonymous";
            
            await Clients.All.SendAsync("UserDisconnected", userName);
            await base.OnDisconnectedAsync(exception);
        }
    }
}
```

### 註冊 SignalR

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddSignalR(options =>
    {
        // 設定選項
        options.EnableDetailedErrors = true;
        options.KeepAliveInterval = TimeSpan.FromSeconds(10);
        options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
        options.HandshakeTimeout = TimeSpan.FromSeconds(15);
    });

    services.AddControllers();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseRouting();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapHub<ChatHub>("/chathub");
        endpoints.MapControllers();
    });
}
```

## JavaScript 客戶端

### 安裝 SignalR 客戶端

```bash
npm install @microsoft/signalr
```

### 建立連線

```javascript
// chat.js
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chathub")
    .withAutomaticReconnect()
    .configureLogging(signalR.LogLevel.Information)
    .build();

// 接收訊息
connection.on("ReceiveMessage", (user, message) => {
    const msg = `${user}: ${message}`;
    console.log(msg);
    displayMessage(msg);
});

// 使用者連線
connection.on("UserConnected", (userName, connectionId) => {
    console.log(`${userName} connected (${connectionId})`);
    displayNotification(`${userName} 已加入聊天室`);
});

// 使用者斷線
connection.on("UserDisconnected", (userName) => {
    console.log(`${userName} disconnected`);
    displayNotification(`${userName} 已離開聊天室`);
});

// 啟動連線
connection.start()
    .then(() => {
        console.log("SignalR Connected");
        document.getElementById("sendButton").disabled = false;
    })
    .catch(err => console.error(err.toString()));

// 發送訊息
document.getElementById("sendButton").addEventListener("click", async () => {
    const user = document.getElementById("userInput").value;
    const message = document.getElementById("messageInput").value;
    
    try {
        await connection.invoke("SendMessage", user, message);
        document.getElementById("messageInput").value = "";
    } catch (err) {
        console.error(err);
    }
});

// 重新連線事件
connection.onreconnecting(error => {
    console.log("Reconnecting...", error);
    displayNotification("連線中斷，正在重新連線...");
});

connection.onreconnected(connectionId => {
    console.log("Reconnected", connectionId);
    displayNotification("已重新連線");
});

connection.onclose(error => {
    console.log("Connection closed", error);
    displayNotification("連線已關閉");
    document.getElementById("sendButton").disabled = true;
});

function displayMessage(message) {
    const li = document.createElement("li");
    li.textContent = message;
    document.getElementById("messagesList").appendChild(li);
}

function displayNotification(message) {
    const div = document.createElement("div");
    div.className = "notification";
    div.textContent = message;
    document.getElementById("notifications").appendChild(div);
    
    setTimeout(() => div.remove(), 3000);
}
```

## .NET 客戶端

### 建立連線

```csharp
using Microsoft.AspNetCore.SignalR.Client;
using System;
using System.Threading.Tasks;

namespace SignalRClient
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var connection = new HubConnectionBuilder()
                .WithUrl("https://localhost:5001/chathub")
                .WithAutomaticReconnect()
                .Build();

            // 接收訊息
            connection.On<string, string>("ReceiveMessage", (user, message) =>
            {
                Console.WriteLine($"{user}: {message}");
            });

            // 使用者連線
            connection.On<string, string>("UserConnected", (userName, connectionId) =>
            {
                Console.WriteLine($"[系統] {userName} 已加入聊天室");
            });

            // 使用者斷線
            connection.On<string>("UserDisconnected", userName =>
            {
                Console.WriteLine($"[系統] {userName} 已離開聊天室");
            });

            // 重新連線
            connection.Reconnecting += error =>
            {
                Console.WriteLine("[系統] 連線中斷，正在重新連線...");
                return Task.CompletedTask;
            };

            connection.Reconnected += connectionId =>
            {
                Console.WriteLine("[系統] 已重新連線");
                return Task.CompletedTask;
            };

            connection.Closed += error =>
            {
                Console.WriteLine("[系統] 連線已關閉");
                return Task.CompletedTask;
            };

            try
            {
                await connection.StartAsync();
                Console.WriteLine("SignalR Connected");
                Console.WriteLine("請輸入使用者名稱：");
                var userName = Console.ReadLine();

                while (true)
                {
                    Console.Write("訊息: ");
                    var message = Console.ReadLine();

                    if (string.IsNullOrEmpty(message))
                        break;

                    await connection.InvokeAsync("SendMessage", userName, message);
                }

                await connection.StopAsync();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"錯誤: {ex.Message}");
            }

            Console.WriteLine("Press any key to exit...");
            Console.ReadKey();
        }
    }
}
```

## 強型別 Hub

### 定義介面

```csharp
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string userName);
    Task UserLeft(string userName);
    Task TypingNotification(string userName);
}

public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.ReceiveMessage(user, message);
    }

    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName).UserJoined(Context.User.Identity.Name);
    }

    public async Task NotifyTyping()
    {
        await Clients.Others.TypingNotification(Context.User.Identity.Name);
    }
}
```

### 客戶端使用

```csharp
connection.On<string, string>("ReceiveMessage", (user, message) =>
{
    Console.WriteLine($"{user}: {message}");
});

connection.On<string>("UserJoined", userName =>
{
    Console.WriteLine($"{userName} 已加入");
});

connection.On<string>("TypingNotification", userName =>
{
    Console.WriteLine($"{userName} 正在輸入...");
});
```

## 進階功能

### 串流傳輸

```csharp
public class DataHub : Hub
{
    // Server-to-Client 串流
    public async IAsyncEnumerable<int> Counter(
        int count,
        int delay,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        for (var i = 0; i < count; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            yield return i;
            await Task.Delay(delay, cancellationToken);
        }
    }

    // Client-to-Server 串流
    public async Task UploadStream(IAsyncEnumerable<string> stream)
    {
        await foreach (var item in stream)
        {
            Console.WriteLine($"Received: {item}");
        }
    }
}

// 客戶端接收串流
connection.stream("Counter", 10, 500)
    .subscribe({
        next: (item) => {
            console.log(item);
        },
        complete: () => {
            console.log("Stream completed");
        },
        error: (err) => {
            console.error(err);
        }
    });
```

### 使用者識別

```csharp
public class NameUserIdProvider : IUserIdProvider
{
    public string GetUserId(HubConnectionContext connection)
    {
        return connection.User?.FindFirst(ClaimTypes.Name)?.Value;
    }
}

// 註冊
services.AddSingleton<IUserIdProvider, NameUserIdProvider>();

// Hub 中使用
public async Task SendPrivateMessage(string toUserId, string message)
{
    await Clients.User(toUserId).SendAsync("ReceiveMessage", 
        Context.User.Identity.Name, message);
}
```

### Hub Filter

```csharp
public class LoggingHubFilter : IHubFilter
{
    private readonly ILogger<LoggingHubFilter> _logger;

    public LoggingHubFilter(ILogger<LoggingHubFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object>> next)
    {
        _logger.LogInformation(
            "Calling hub method '{Method}' by user '{User}'",
            invocationContext.HubMethodName,
            invocationContext.Context.User?.Identity?.Name);

        try
        {
            return await next(invocationContext);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Exception calling '{Method}'",
                invocationContext.HubMethodName);
            throw;
        }
    }
}

// 註冊
services.AddSignalR(options =>
{
    options.AddFilter<LoggingHubFilter>();
});
```

## 擴展到多伺服器

### 使用 Redis Backplane

```bash
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

```csharp
services.AddSignalR()
    .AddStackExchangeRedis("localhost:6379", options =>
    {
        options.Configuration.ChannelPrefix = "MyApp";
    });
```

### 使用 Azure SignalR Service

```bash
dotnet add package Microsoft.Azure.SignalR
```

```csharp
services.AddSignalR()
    .AddAzureSignalR(options =>
    {
        options.ConnectionString = Configuration["Azure:SignalR:ConnectionString"];
    });
```

## 安全性

### JWT 認證

```csharp
// Startup.cs
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = Configuration["Jwt:Issuer"],
            ValidAudience = Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
        };

        // SignalR 使用
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];

                var path = context.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(accessToken) &&
                    path.StartsWithSegments("/chathub"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });

// Hub 使用授權
[Authorize]
public class ChatHub : Hub
{
    public async Task SendMessage(string message)
    {
        var userName = Context.User.Identity.Name;
        await Clients.All.SendAsync("ReceiveMessage", userName, message);
    }
}

// 客戶端傳送 Token
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chathub", {
        accessTokenFactory: () => {
            return getAccessToken(); // 取得 JWT Token
        }
    })
    .build();
```

### 授權政策

```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
    {
        policy.RequireClaim("Role", "Admin");
    });
});

[Authorize(Policy = "AdminOnly")]
public class AdminHub : Hub
{
    // 只有 Admin 可以呼叫
}
```

## 實際應用範例

### 即時通知系統

```csharp
public interface INotificationClient
{
    Task ReceiveNotification(NotificationDto notification);
}

public class NotificationHub : Hub<INotificationClient>
{
    private readonly INotificationService _notificationService;

    public NotificationHub(INotificationService notificationService)
    {
        _notificationService = notificationService;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (!string.IsNullOrEmpty(userId))
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, userId);
            
            // 載入未讀通知
            var unreadNotifications = await _notificationService.GetUnreadAsync(userId);
            
            foreach (var notification in unreadNotifications)
            {
                await Clients.Caller.ReceiveNotification(notification);
            }
        }

        await base.OnConnectedAsync();
    }

    public async Task MarkAsRead(int notificationId)
    {
        var userId = Context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        await _notificationService.MarkAsReadAsync(notificationId, userId);
    }
}

// 從 Service 發送通知
public class NotificationService
{
    private readonly IHubContext<NotificationHub, INotificationClient> _hubContext;

    public NotificationService(IHubContext<NotificationHub, INotificationClient> hubContext)
    {
        _hubContext = hubContext;
    }

    public async Task SendToUserAsync(string userId, NotificationDto notification)
    {
        await _hubContext.Clients.Group(userId).ReceiveNotification(notification);
    }

    public async Task BroadcastAsync(NotificationDto notification)
    {
        await _hubContext.Clients.All.ReceiveNotification(notification);
    }
}
```

### 即時儀表板

```csharp
public class DashboardHub : Hub
{
    public async Task SubscribeToMetrics(string metricType)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"metric_{metricType}");
    }

    public async Task UnsubscribeFromMetrics(string metricType)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"metric_{metricType}");
    }
}

// 背景服務定期推送數據
public class MetricsPublisher : BackgroundService
{
    private readonly IHubContext<DashboardHub> _hubContext;

    public MetricsPublisher(IHubContext<DashboardHub> hubContext)
    {
        _hubContext = hubContext;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var cpuMetric = GetCpuUsage();
            await _hubContext.Clients.Group("metric_cpu")
                .SendAsync("ReceiveMetric", "CPU", cpuMetric, stoppingToken);

            var memoryMetric = GetMemoryUsage();
            await _hubContext.Clients.Group("metric_memory")
                .SendAsync("ReceiveMetric", "Memory", memoryMetric, stoppingToken);

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private double GetCpuUsage() => Random.Shared.NextDouble() * 100;
    private double GetMemoryUsage() => Random.Shared.NextDouble() * 100;
}
```

## 效能最佳化

### 訊息壓縮

```csharp
services.AddSignalR()
    .AddMessagePackProtocol(options =>
    {
        options.SerializerOptions = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);
    });
```

### 連線限制

```csharp
services.AddSignalR(options =>
{
    options.MaximumReceiveMessageSize = 32 * 1024; // 32 KB
    options.StreamBufferCapacity = 10;
});
```

## 小結

ASP.NET Core SignalR：
- 簡化即時雙向通訊開發
- 自動選擇最佳傳輸協定
- 支援群組和使用者管理
- 內建重新連線機制

相較於原生 WebSocket：
- 更高階的抽象
- 自動降級到其他傳輸方式
- 內建擴展方案
- 完整的認證授權整合

下週將探討單元測試與整合測試策略。
