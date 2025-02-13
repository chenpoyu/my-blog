---
layout: post
title: ".NET Core + React：前後端分離的部署實戰"
date: 2025-02-13 09:30:00 +0800
categories: [全端開發]
tags: [.NET Core, React, DevOps, CI/CD]
---

## 全部押寶在 Microsoft 技術棧

這次專案跟過去最大的不同，是我們把技術棧統一了。

還記得 2021 年那個物聯網專案嗎？後端 Java + Spring Boot，部署在 AWS。那時候團隊還要維護 Java 和 .NET 兩套環境，開發機器要裝一堆工具，Jenkins 的 pipeline 也要寫兩份不同的腳本。

2022 年轉到 React 之後，前端倒是統一了。但後端還是 Java 和 .NET 並存，看專案需求決定用哪個。

這次終於下定決心：**全部都用 .NET Core**。

不是因為 .NET Core 比 Java 好（兩個都很成熟），而是：
- 團隊大部分人對 C# 比較熟
- 客戶環境以 Windows/Azure 為主
- .NET Core 的效能已經很不錯了
- 省掉切換程式語言的成本

前端繼續用 React。這個沒什麼好說的，2022 年轉換之後就一直用到現在。雖然 Vue 3 看起來也不錯，但團隊已經建立了一套 React 的開發流程，沒必要再改。

## 專案結構

我們的 repository 結構長這樣：

```
my-app/
├── src/
│   ├── MyApp.Api/                    # API Gateway (YARP)
│   ├── MyApp.Modules.Member/         # 會員模組
│   ├── MyApp.Modules.Points/         # 點數模組
│   ├── MyApp.Modules.Coupon/         # 優惠券模組
│   ├── MyApp.Modules.Activity/       # 活動模組
│   ├── MyApp.Modules.Article/        # 文章模組
│   ├── MyApp.Modules.Survey/         # 問卷模組
│   ├── MyApp.Modules.Notification/   # 通知模組
│   └── MyApp.Shared/                 # 共用程式碼
├── web/
│   ├── src/
│   │   ├── components/               # React 元件
│   │   ├── pages/                    # 頁面
│   │   ├── services/                 # API 呼叫
│   │   └── utils/                    # 工具函式
│   ├── public/
│   └── package.json
├── tests/
│   ├── MyApp.Tests.Unit/
│   └── MyApp.Tests.Integration/
└── docker-compose.yml
```

一個 repo 包含前後端所有程式碼。有人說這樣會讓 repo 太大，但我覺得好處更多：
- 前後端版本同步，不會出現「前端已經更新但後端還沒部署」的問題
- 改 API 的時候可以同時改前端，一起 commit
- CI/CD pipeline 比較好寫

## 本機開發環境

### 後端啟動

後端是 .NET 8，開發工具統一用 VSCode。

當初選 VSCode 的原因很簡單：前後端都可以用同一個 IDE，減少工具切換的成本。而且 VSCode 的 C# Dev Kit 已經很成熟了，除錯、IntelliSense 都沒問題。

```bash
cd src/MyApp.Api
dotnet restore
dotnet run
```

API 會跑在 `https://localhost:5001`。

因為我們用模組化架構，所有模組都在同一個 process 裡，不用開一堆 terminal 分別啟動不同服務。省事很多。

### 前端啟動

前端用 **Vite + React**，不是 Create React App。

2022 年的時候我們還在用 CRA，但這兩年 Vite 已經成為主流了。啟動速度快、HMR（Hot Module Replacement）也更順，沒理由不換。

```bash
cd web
npm install
npm run dev
### CORS 問題

前端和後端跑在不同 port，會遇到 CORS 問題。解決方法是在 .NET 這邊開啟 CORS：

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("DevPolicy", policy =>
    {
        policy.WithOrigins("http://localhost:5173")  // Vite 預設 port
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

// ...

if (app.Environment.IsDevelopment())
{
    app.UseCors("DevPolicy");
}
```

這樣本機開發就沒問題了。ronment.IsDevelopment())
{
    app.UseCors("DevPolicy");
}
```

這樣本機開發就沒問題了。

## API 呼叫的統一管理

以前寫 React 的時候，API 呼叫散落在各個元件裡，很難管理。這次我們把所有 API 呼叫都集中在 `services/` 資料夾。

```javascript
// services/memberService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'https://localhost:5001';

export const memberService = {
  async getMembers(filters) {
    const response = await axios.get(`${API_URL}/api/members`, {
      params: filters
    });
    return response.data;
  },

  async getMember(id) {
    const response = await axios.get(`${API_URL}/api/members/${id}`);
    return response.data;
  },

  async createMember(data) {
    const response = await axios.post(`${API_URL}/api/members`, data);
    return response.data;
  },

  async updateMember(id, data) {
    const response = await axios.put(`${API_URL}/api/members/${id}`, data);
    return response.data;
  }
};
```

在元件裡這樣用：

```javascript
import { memberService } from '@/services/memberService';

function MemberList() {
  const [members, setMembers] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    const fetchMembers = async () => {
      setLoading(true);
      try {
        const data = await memberService.getMembers();
        setMembers(data);
      } catch (error) {
        console.error('Failed to fetch members', error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchMembers();
  }, []);

  // ...
}
```

這樣做的好處：
- API 路徑統一管理，要改的時候只改一個地方
- 容易 mock（寫測試的時候很方便）
- 可以加上統一的錯誤處理、loading state 等

## 環境變數管理

不同環境（開發、測試、正式）會有不同的設定。

### 後端

本機開發時我們還是用 `appsettings.Development.json`，但部署到 Azure 後，**完全不用 appsettings.json 檔案**。

為什麼？因為 Azure App Service 有**組態（Configuration）**功能。

所有的設定都在 Azure Portal 上管理：
- 資料庫連線字串
- Redis 位址
- 第三方 API 金鑰
- Feature flags

這樣做的好處：
- **安全**：機敏資訊不會出現在 source code 裡
- **靈活**：要改設定不用重新部署，在 Portal 上改完重啟就好
- **環境隔離**：測試和正式環境的設定完全獨立

在程式裡讀取組態的方式跟讀 appsettings 一模一樣：

```csharp
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var redisConnection = builder.Configuration["Redis:ConnectionString"];
```

Azure 會自動把 Configuration 的值注入到 `IConfiguration` 裡，透明無感。

### 前端

React 用 `.env` 檔案：

```
.env                       # 預設值
.env.development          # 本機開發
.env.staging              # 測試環境
.env.production           # 正式環境
```

```bash
# .env.development
REACT_APP_API_URL=https://localhost:5001

# .env.production
REACT_APP_API_URL=https://api.myapp.com
```

Build 的時候 React 會自動帶入對應的環境變數。

## Docker 部署

雖然是部署到 Azure App Service，但我們還是用 Docker container。

### 後端 Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/MyApp.Api/MyApp.Api.csproj", "MyApp.Api/"]
COPY ["src/MyApp.Modules.Member/MyApp.Modules.Member.csproj", "MyApp.Modules.Member/"]
# ... 其他模組的 csproj

RUN dotnet restore "MyApp.Api/MyApp.Api.csproj"
COPY src/ .
WORKDIR "/src/MyApp.Api"
RUN dotnet build "MyApp.Api.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.Api.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### 前端 Dockerfile

前端比較特別，我們用多階段建置：第一階段用 Node 編譯，第二階段用 Nginx 提供靜態檔案。

```dockerfile
# Stage 1: Build
FROM node:18-alpine AS build
WORKDIR /app
COPY web/package*.json ./
RUN npm ci
COPY web/ .
RUN npm run build

# Stage 2: Serve
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY web/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Nginx 設定檔（支援 React Router 的 SPA routing）：

```nginx
server {
    listen 80;
    server_name _;
    
    root /usr/share/nginx/html;
    index index.html;
    
    # React Router 支援
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # API proxy（可選，如果前端和後端在同一個 domain）
    location /api/ {
        proxy_pass https://api.myapp.com/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # 靜態檔案 cache
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

## CI/CD Pipeline

我們用 GitHub Actions。每次 push 到 main branch 就會自動部署到 Azure。

### 後端 Pipeline

```yaml
name: Deploy Backend to Azure

on:
  push:
    branches: [main]
    paths:
      - 'src/**'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Log in to Azure Container Registry
        uses: docker/login-action@v2
        with:
          registry: myregistry.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      
      - name: Build and push Docker image
        run: |
          docker build -t myregistry.azurecr.io/myapp-api:${{ github.sha }} -f Dockerfile.backend .
          docker push myregistry.azurecr.io/myapp-api:${{ github.sha }}
      
      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: myapp-api
          images: myregistry.azurecr.io/myapp-api:${{ github.sha }}
```

### 前端 Pipeline

```yaml
name: Deploy Frontend to Azure

on:
  push:
    branches: [main]
    paths:
      - 'web/**'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Log in to Azure Container Registry
        uses: docker/login-action@v2
        with:
          registry: myregistry.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      
      - name: Build and push Docker image
        run: |
          docker build -t myregistry.azurecr.io/myapp-web:${{ github.sha }} -f Dockerfile.frontend .
          docker push myregistry.azurecr.io/myapp-web:${{ github.sha }}
      
      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: myapp-web
          images: myregistry.azurecr.io/myapp-web:${{ github.sha }}
```

這樣設定的好處是：改後端不會觸發前端部署，改前端不會觸發後端部署。省時間也省成本。

## 前後端分開部署 vs 放在一起？

有人可能會問：為什麼要分成兩個 App Service？能不能把 React 的靜態檔案直接放在 .NET API 裡面一起部署？

當然可以。.NET 支援 serve static files：

```csharp
app.UseStaticFiles();
app.MapFallbackToFile("index.html");
```

把 React build 出來的檔案放到 `wwwroot/`，一個 container 就搞定了。

## API 文件：Swagger 的優缺點

這次我們用 **Swagger（OpenAPI）** 來管理 API 文件。

設定很簡單：

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My App API",
        Version = "v1",
        Description = "會員系統 API 文件"
    });
    
    // 加入 XML 註解
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

app.UseSwagger();
app.UseSwaggerUI();
```

在 Controller 加上 XML 註解：

```csharp
/// <summary>
/// 取得會員列表
/// </summary>
/// <param name="pageIndex">頁碼（從 1 開始）</param>
/// <param name="pageSize">每頁筆數</param>
/// <returns>會員列表</returns>
[HttpGet]
public async Task<ActionResult<PagedResult<MemberDto>>> GetMembers(
    int pageIndex = 1, int pageSize = 20)
{
    // ...
}
```

Swagger UI 就會自動產生漂亮的文件，還可以直接測試 API。

**優點**：
- 自動化，不用手寫文件
- 可以直接在網頁上測試 API
- 前端可以看到最新的 API 規格

**缺點**：
- Swagger UI 不夠直覺，前端還是常常要來問細節
- 如果沒寫 XML 註解，文件就很陽春

下次如果有機會，我想試試用 NSwag 或 OpenAPI Generator 自動產生 TypeScript 的 API client。這樣前端就不用手刻 `memberService.js`，而且有型別檢查，更不容易出錯。

## 目前的狀況

開發環境和部署流程都建立好了，團隊已經開始實際開發功能。

目前看起來這些選擇還不錯：

- **.NET + React 的組合**：後端用強型別，前端用 TypeScript + React，開發體驗還算順
- **Vite 啟動快**：開發時的 HMR 幾乎是秒級，比之前的 CRA 快很多
- **Docker**：本機和雲端用同一個 image，應該能減少環境不一致的問題
- **Azure 組態**：不用把機敏資訊寫進 code，看起來蠻方便的
- **VSCode 統一工具**：前後端用同一個 IDE，團隊成員切換成本低

但真正的考驗還沒開始。接下來九個月要陸續開發十幾個模組，系統複雜度會越來越高。

這些技術選擇能不能撐住，就看實戰了。

下週會分享模組之間的 gRPC 通訊機制，以及為什麼我們選擇 gRPC 而不是 event-driven。
## 實際效益

系統上線兩個月了，回頭看這些決定：

- **.NET + React 的組合很順**：後端用強型別，前端用靈活的 JSX，各取所長
- **Docker 部署很穩**：本機、測試、正式環境都跑同一個 image，幾乎沒有「環境不一致」的問題
- **CI/CD 省很多時間**：git push 之後就不用管了，自動 build、test、deploy

唯一的遺憾是當初沒有把 API 文件自動化做好。現在前後端要對 API 規格都是靠 Swagger，但 Swagger UI 不夠直覺，前端同事常常要來問「這個欄位是什麼意思」。

下次應該考慮用 OpenAPI Generator 或者 NSwag，自動產生 TypeScript 的 API client，這樣前端就不用手刻 service 了。

下週會分享模組之間的通訊機制，以及如何用事件驅動架構減少模組耦合。
