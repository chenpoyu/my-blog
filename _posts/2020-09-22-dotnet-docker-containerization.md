---
layout: post
title: ".NET Core 應用程式 Docker 容器化"
date: 2020-09-22 15:10:00 +0800
categories: [框架, .NET]
tags: [.NET Core, Docker, Container, DevOps]
---

這週研究如何將 .NET Core 應用程式容器化，使用 Docker 簡化部署流程。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0** + **Docker**

## Dockerfile 撰寫

### 基本 Dockerfile

```dockerfile
# 使用官方 .NET SDK 映像進行建置
FROM mcr.microsoft.com/dotnet/sdk:3.1 AS build
WORKDIR /src

# 複製專案檔並還原套件
COPY ["MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# 複製所有檔案並建置
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# 發行應用程式
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

# 建立最終映像
FROM mcr.microsoft.com/dotnet/aspnet:3.1 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 多階段建置優化

```dockerfile
# 第一階段：還原套件
FROM mcr.microsoft.com/dotnet/sdk:3.1 AS restore
WORKDIR /src
COPY ["MyApp/MyApp.csproj", "MyApp/"]
COPY ["MyApp.Core/MyApp.Core.csproj", "MyApp.Core/"]
COPY ["MyApp.Data/MyApp.Data.csproj", "MyApp.Data/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# 第二階段：建置
FROM restore AS build
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build --no-restore

# 第三階段：測試（可選）
FROM build AS test
WORKDIR /src/MyApp.Tests
COPY ["MyApp.Tests/MyApp.Tests.csproj", "MyApp.Tests/"]
RUN dotnet restore "MyApp.Tests.csproj"
COPY MyApp.Tests/ .
RUN dotnet test --no-restore --verbosity normal

# 第四階段：發行
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish \
    --no-restore \
    --no-build \
    /p:UseAppHost=false

# 第五階段：最終映像
FROM mcr.microsoft.com/dotnet/aspnet:3.1 AS final

# 建立非 root 使用者
RUN addgroup --system --gid 1001 appuser && \
    adduser --system --uid 1001 --ingroup appuser appuser

WORKDIR /app
COPY --from=publish --chown=appuser:appuser /app/publish .

# 切換至非 root 使用者
USER appuser

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1

EXPOSE 80
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### .dockerignore 設定

```
# .dockerignore
**/.dockerignore
**/.env
**/.git
**/.gitignore
**/.vs
**/.vscode
**/*.*proj.user
**/*.dbmdl
**/*.jfm
**/bin
**/charts
**/docker-compose*
**/compose*
**/Dockerfile*
**/node_modules
**/npm-debug.log
**/obj
**/secrets.dev.yaml
**/values.dev.yaml
README.md
```

## 建置與執行

### 建置映像

```bash
# 基本建置
docker build -t myapp:latest .

# 指定 Dockerfile
docker build -f Dockerfile.production -t myapp:1.0.0 .

# 使用建置參數
docker build --build-arg ASPNETCORE_ENVIRONMENT=Production -t myapp:latest .

# 不使用快取
docker build --no-cache -t myapp:latest .
```

### 執行容器

```bash
# 基本執行
docker run -d -p 8080:80 --name myapp myapp:latest

# 設定環境變數
docker run -d -p 8080:80 \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__DefaultConnection="Server=db;Database=myapp;User=sa;Password=YourPassword;" \
  --name myapp \
  myapp:latest

# 掛載 Volume
docker run -d -p 8080:80 \
  -v $(pwd)/logs:/app/logs \
  -v $(pwd)/appsettings.json:/app/appsettings.json:ro \
  --name myapp \
  myapp:latest

# 限制資源
docker run -d -p 8080:80 \
  --memory="512m" \
  --cpus="1.0" \
  --name myapp \
  myapp:latest
```

## Docker Compose

### 基本設定

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=myapp;User=sa;Password=YourPassword123!
    depends_on:
      - db
    networks:
      - app-network
    volumes:
      - ./logs:/app/logs

  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  sqldata:
```

### 多環境設定

```yaml
# docker-compose.override.yml (開發環境)
version: '3.8'

services:
  web:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    volumes:
      - ./src:/src
    build:
      args:
        - BUILD_CONFIGURATION=Debug
    command: dotnet watch run

  db:
    ports:
      - "1433:1433"
```

```yaml
# docker-compose.prod.yml (生產環境)
version: '3.8'

services:
  web:
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M

  db:
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
```

### 執行 Docker Compose

```bash
# 啟動服務
docker-compose up -d

# 使用特定 compose 檔案
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 查看日誌
docker-compose logs -f web

# 停止服務
docker-compose down

# 停止並移除 volumes
docker-compose down -v

# 重新建置
docker-compose up -d --build

# 擴展服務
docker-compose up -d --scale web=3
```

## 整合資料庫

### 使用 PostgreSQL

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=myapp;Username=postgres;Password=postgres
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
```

### 資料庫初始化

```dockerfile
# 在 Dockerfile 中加入 Migration
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

# 安裝 EF Core Tools
RUN dotnet tool install --global dotnet-ef
ENV PATH="${PATH}:/root/.dotnet/tools"

# 產生 Migration Script
RUN dotnet ef migrations script -o /app/publish/migration.sql
```

```bash
# docker-entrypoint.sh
#!/bin/bash
set -e

echo "Waiting for database..."
until dotnet ef database update --no-build; do
  echo "Database is unavailable - sleeping"
  sleep 1
done

echo "Database is up - executing command"
exec dotnet MyApp.dll
```

```dockerfile
# 在 Dockerfile 中使用
COPY docker-entrypoint.sh /app/
RUN chmod +x /app/docker-entrypoint.sh
ENTRYPOINT ["/app/docker-entrypoint.sh"]
```

## Redis 快取整合

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - Redis__Configuration=redis:6379
    depends_on:
      - redis

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  redis-data:
```

## 監控與日誌

### 加入健康檢查

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        .AddDbContextCheck<ApplicationDbContext>()
        .AddRedis(Configuration["Redis:Configuration"]);
}

public void Configure(IApplicationBuilder app)
{
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapHealthChecks("/health");
        endpoints.MapControllers();
    });
}
```

```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1
```

### 日誌收集

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # 或使用集中式日誌
  web:
    build: .
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: myapp.web
```

## 最佳實踐

### 映像大小優化

```dockerfile
# 使用 Alpine 基礎映像
FROM mcr.microsoft.com/dotnet/aspnet:3.1-alpine AS final

# 移除不需要的檔案
RUN rm -rf /app/*.pdb
```

### 快取層級優化

```dockerfile
# 先複製專案檔，再複製程式碼
# 這樣修改程式碼時不會重新還原套件
FROM mcr.microsoft.com/dotnet/sdk:3.1 AS build
WORKDIR /src

# 先還原套件（較少變動）
COPY ["MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# 再複製程式碼（經常變動）
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build -c Release -o /app/build --no-restore
```

### 安全性設定

```dockerfile
# 使用非 root 使用者
RUN addgroup --system --gid 1001 appuser && \
    adduser --system --uid 1001 --ingroup appuser appuser

# 設定檔案權限
COPY --from=publish --chown=appuser:appuser /app/publish .

USER appuser

# 不暴露敏感資訊
ENV ASPNETCORE_ENVIRONMENT=Production
```

## CI/CD 整合

### GitHub Actions

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Login to DockerHub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and push
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: myusername/myapp:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Run tests
      run: |
        docker-compose -f docker-compose.test.yml up --abort-on-container-exit
        docker-compose -f docker-compose.test.yml down
```

### Azure DevOps

```yaml
# azure-pipelines.yml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  dockerRegistryServiceConnection: 'myDockerRegistry'
  imageRepository: 'myapp'
  containerRegistry: 'myregistry.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
- stage: Build
  displayName: Build and push stage
  jobs:
  - job: Build
    displayName: Build
    steps:
    - task: Docker@2
      displayName: Build and push image
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
          latest
```

## 容器編排

### Kubernetes 基本部署

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:latest
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-connection
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: myapp
```

## 除錯容器

```bash
# 進入執行中的容器
docker exec -it myapp /bin/bash

# 查看日誌
docker logs -f myapp

# 查看容器資源使用
docker stats myapp

# 檢查容器詳細資訊
docker inspect myapp

# 複製檔案
docker cp myapp:/app/logs ./logs
```

## 小結

.NET Core Docker 容器化：
- 多階段建置減少映像大小
- Docker Compose 簡化多容器管理
- 健康檢查確保服務可用性
- 整合 CI/CD 自動化部署

相較於傳統部署：
- 環境一致性更高
- 部署速度更快
- 資源隔離更好
- 擴展更容易

下週將探討微服務架構設計模式。
