---
layout: post
title: ".NET Core CI/CD 實踐指南"
date: 2020-11-10 14:30:00 +0800
categories: [框架, .NET]
tags: [.NET Core, CI/CD, DevOps, GitHub Actions, Azure DevOps]
---

這週研究 .NET Core 應用程式的持續整合和持續部署，涵蓋 GitHub Actions 和 Azure DevOps 兩種主流方案。

> 本文環境：**.NET Core 3.1 LTS** + **C# 9.0**

## GitHub Actions

### 基本 CI 流程

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - name: Setup .NET
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.1.x

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore --configuration Release

    - name: Test
      run: dotnet test --no-build --configuration Release --verbosity normal

    - name: Publish
      run: dotnet publish --no-build --configuration Release --output ./publish

    - name: Upload artifact
      uses: actions/upload-artifact@v2
      with:
        name: webapp
        path: ./publish
```

### 多專案建置

```yaml
# .github/workflows/multi-project.yml
name: Multi-Project CI

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        project:
          - MyApp.API
          - MyApp.Worker
          - MyApp.Functions

    steps:
    - uses: actions/checkout@v2

    - name: Setup .NET
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.1.x

    - name: Build ${{ matrix.project }}
      run: |
        cd src/${{ matrix.project }}
        dotnet restore
        dotnet build --configuration Release
        dotnet test --configuration Release

    - name: Publish ${{ matrix.project }}
      run: |
        cd src/${{ matrix.project }}
        dotnet publish --configuration Release --output ../../publish/${{ matrix.project }}

    - name: Upload ${{ matrix.project }} artifact
      uses: actions/upload-artifact@v2
      with:
        name: ${{ matrix.project }}
        path: ./publish/${{ matrix.project }}
```

### Docker 建置與推送

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
    - uses: actions/checkout@v2

    - name: Log in to Container Registry
      uses: docker/login-action@v1
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v3
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}

    - name: Build and push Docker image
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        build-args: |
          BUILD_CONFIGURATION=Release
```

### 部署到 Azure

```yaml
# .github/workflows/azure-deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [ main ]

env:
  AZURE_WEBAPP_NAME: my-app-name
  DOTNET_VERSION: '3.1.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - name: Setup .NET
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Build and publish
      run: |
        dotnet restore
        dotnet build --configuration Release
        dotnet publish -c Release -o ./publish

    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.AZURE_WEBAPP_NAME }}
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: ./publish
```

## Azure DevOps

### 基本 Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main
    - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetSdkVersion: '3.1.x'

stages:
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: Build
    displayName: 'Build Job'
    steps:
    - task: UseDotNet@2
      displayName: 'Use .NET SDK $(dotnetSdkVersion)'
      inputs:
        packageType: 'sdk'
        version: '$(dotnetSdkVersion)'

    - task: DotNetCoreCLI@2
      displayName: 'Restore dependencies'
      inputs:
        command: 'restore'
        projects: '**/*.csproj'

    - task: DotNetCoreCLI@2
      displayName: 'Build'
      inputs:
        command: 'build'
        projects: '**/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-restore'

    - task: DotNetCoreCLI@2
      displayName: 'Run tests'
      inputs:
        command: 'test'
        projects: '**/*Tests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'

    - task: PublishCodeCoverageResults@1
      displayName: 'Publish code coverage'
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'

    - task: DotNetCoreCLI@2
      displayName: 'Publish'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory) --no-build'
        zipAfterPublish: true

    - task: PublishBuildArtifacts@1
      displayName: 'Publish artifact'
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'
        artifactName: 'drop'
```

### 多環境部署

```yaml
# azure-pipelines-multi-env.yml
stages:
- stage: Build
  displayName: 'Build'
  jobs:
  - job: Build
    steps:
    # ... build steps

- stage: DeployDev
  displayName: 'Deploy to Development'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: DeployDev
    displayName: 'Deploy to Dev'
    environment: 'Development'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App (Dev)'
            inputs:
              azureSubscription: 'Azure-Subscription'
              appType: 'webAppLinux'
              appName: 'myapp-dev'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'

- stage: DeployStaging
  displayName: 'Deploy to Staging'
  dependsOn: DeployDev
  condition: succeeded()
  jobs:
  - deployment: DeployStaging
    displayName: 'Deploy to Staging'
    environment: 'Staging'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App (Staging)'
            inputs:
              azureSubscription: 'Azure-Subscription'
              appType: 'webAppLinux'
              appName: 'myapp-staging'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'

- stage: DeployProduction
  displayName: 'Deploy to Production'
  dependsOn: DeployStaging
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployProduction
    displayName: 'Deploy to Production'
    environment: 'Production'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App (Production)'
            inputs:
              azureSubscription: 'Azure-Subscription'
              appType: 'webAppLinux'
              appName: 'myapp-prod'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              deploymentMethod: 'zipDeploy'
```

### Docker 建置

```yaml
# azure-pipelines-docker.yml
trigger:
  branches:
    include:
    - main
  tags:
    include:
    - v*

resources:
- repo: self

variables:
  dockerRegistryServiceConnection: 'MyDockerRegistry'
  imageRepository: 'myapp'
  containerRegistry: 'myregistry.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
- stage: Build
  displayName: 'Build and Push Docker Image'
  jobs:
  - job: Build
    displayName: 'Build Job'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Docker@2
      displayName: 'Build Docker image'
      inputs:
        command: 'build'
        repository: '$(imageRepository)'
        dockerfile: '$(dockerfilePath)'
        containerRegistry: '$(dockerRegistryServiceConnection)'
        tags: |
          $(tag)
          latest

    - task: Docker@2
      displayName: 'Push Docker image'
      inputs:
        command: 'push'
        repository: '$(imageRepository)'
        containerRegistry: '$(dockerRegistryServiceConnection)'
        tags: |
          $(tag)
          latest

    - task: Docker@2
      displayName: 'Scan image for vulnerabilities'
      inputs:
        command: 'scan'
        arguments: '$(containerRegistry)/$(imageRepository):$(tag)'

- stage: Deploy
  displayName: 'Deploy to AKS'
  dependsOn: Build
  jobs:
  - deployment: Deploy
    displayName: 'Deploy to Kubernetes'
    environment: 'production'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to Kubernetes'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'MyAKSCluster'
              namespace: 'production'
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yml
                $(Pipeline.Workspace)/manifests/service.yml
              containers: '$(containerRegistry)/$(imageRepository):$(tag)'
```

## 自動化測試整合

### 單元測試和覆蓋率

```yaml
# GitHub Actions
- name: Test with coverage
  run: |
    dotnet test \
      --configuration Release \
      --no-build \
      --collect:"XPlat Code Coverage" \
      --results-directory ./coverage

- name: Generate coverage report
  uses: codecov/codecov-action@v2
  with:
    files: ./coverage/**/coverage.cobertura.xml
    fail_ci_if_error: true
```

```yaml
# Azure DevOps
- task: DotNetCoreCLI@2
  displayName: 'Run tests with coverage'
  inputs:
    command: 'test'
    projects: '**/*Tests/*.csproj'
    arguments: >
      --configuration $(buildConfiguration)
      --no-build
      --collect:"XPlat Code Coverage"
      --settings coverlet.runsettings
      --logger trx
      --results-directory $(Common.TestResultsDirectory)

- task: PublishTestResults@2
  displayName: 'Publish test results'
  condition: succeededOrFailed()
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '**/*.trx'
    searchFolder: '$(Common.TestResultsDirectory)'

- task: PublishCodeCoverageResults@1
  displayName: 'Publish code coverage'
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: '$(Common.TestResultsDirectory)/**/coverage.cobertura.xml'
    failIfCoverageEmpty: true
```

### 整合測試

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Testing
      - ConnectionStrings__DefaultConnection=Server=db;Database=TestDb;User=sa;Password=Test123!
    depends_on:
      - db
    networks:
      - test-network

  db:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Test123!
    networks:
      - test-network

  integration-tests:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - ASPNETCORE_ENVIRONMENT=Testing
      - ConnectionStrings__DefaultConnection=Server=db;Database=TestDb;User=sa;Password=Test123!
    depends_on:
      - db
      - app
    networks:
      - test-network
    command: dotnet test --configuration Release --logger "trx;LogFileName=test_results.trx"

networks:
  test-network:
    driver: bridge
```

```yaml
# GitHub Actions with integration tests
- name: Run integration tests
  run: |
    docker-compose -f docker-compose.test.yml up \
      --abort-on-container-exit \
      --exit-code-from integration-tests

- name: Cleanup
  if: always()
  run: docker-compose -f docker-compose.test.yml down -v
```

## 版本管理

### 自動版本號

```yaml
# GitHub Actions with GitVersion
- name: Install GitVersion
  uses: gittools/actions/gitversion/setup@v0.9.13
  with:
    versionSpec: '5.x'

- name: Determine Version
  id: gitversion
  uses: gittools/actions/gitversion/execute@v0.9.13

- name: Display version
  run: |
    echo "Version: ${{ steps.gitversion.outputs.semVer }}"
    echo "InformationalVersion: ${{ steps.gitversion.outputs.informationalVersion }}"

- name: Build with version
  run: |
    dotnet build \
      --configuration Release \
      /p:Version=${{ steps.gitversion.outputs.assemblySemVer }} \
      /p:FileVersion=${{ steps.gitversion.outputs.assemblySemFileVer }} \
      /p:InformationalVersion=${{ steps.gitversion.outputs.informationalVersion }}
```

### Semantic Release

```yaml
# .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [
      "@semantic-release/changelog",
      {
        "changelogFile": "CHANGELOG.md"
      }
    ],
    [
      "@semantic-release/npm",
      {
        "npmPublish": false
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md", "package.json"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

```yaml
# GitHub Actions with semantic-release
- name: Semantic Release
  uses: cycjimmy/semantic-release-action@v3
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    semantic_version: 19
    extra_plugins: |
      @semantic-release/changelog
      @semantic-release/git
```

## 部署策略

### Blue-Green 部署

```yaml
# azure-pipelines-bluegreen.yml
- stage: DeployBlue
  displayName: 'Deploy to Blue Slot'
  jobs:
  - deployment: DeployBlue
    environment: 'Production-Blue'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure-Subscription'
              appName: 'myapp-prod'
              slotName: 'blue'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'

          - task: AzureAppServiceManage@0
            displayName: 'Warm up Blue slot'
            inputs:
              azureSubscription: 'Azure-Subscription'
              action: 'Start Azure App Service'
              webAppName: 'myapp-prod'
              specifySlotOrASE: true
              resourceGroupName: 'myapp-rg'
              slotName: 'blue'

- stage: SwapSlots
  displayName: 'Swap Blue to Production'
  dependsOn: DeployBlue
  jobs:
  - deployment: SwapSlots
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureAppServiceManage@0
            displayName: 'Swap slots'
            inputs:
              azureSubscription: 'Azure-Subscription'
              action: 'Swap Slots'
              webAppName: 'myapp-prod'
              resourceGroupName: 'myapp-rg'
              sourceSlot: 'blue'
              targetSlot: 'production'
```

### Canary 部署

```yaml
# kubernetes/canary-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
  labels:
    app: myapp
    version: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:latest
        ports:
        - containerPort: 80

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
  labels:
    app: myapp
    version: stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:v1.0.0
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

## 環境變數管理

### GitHub Secrets

```yaml
# 使用 Secrets
- name: Deploy with secrets
  env:
    DATABASE_PASSWORD: ${{ secrets.DB_PASSWORD }}
    API_KEY: ${{ secrets.API_KEY }}
  run: |
    dotnet publish --configuration Release
    # 部署腳本
```

### Azure Key Vault 整合

```yaml
# Azure DevOps with Key Vault
- task: AzureKeyVault@2
  inputs:
    azureSubscription: 'Azure-Subscription'
    KeyVaultName: 'my-keyvault'
    SecretsFilter: '*'
    RunAsPreJob: true

- task: AzureWebApp@1
  inputs:
    appSettings: |
      -ConnectionStrings__DefaultConnection "$(DB-CONNECTION-STRING)"
      -ApiSettings__ApiKey "$(API-KEY)"
```

## 小結

CI/CD 最佳實踐：
- 自動化建置、測試和部署流程
- 使用多環境策略（Dev/Staging/Prod）
- 整合程式碼覆蓋率和品質檢查
- 實作藍綠或金絲雀部署
- 安全管理機密和環境變數

GitHub Actions vs Azure DevOps：
- GitHub Actions 更適合開源專案
- Azure DevOps 更適合企業級應用
- 兩者都支援 Docker 和 Kubernetes
- Azure DevOps 有更完整的測試報告

下週將探討 .NET Core 專案的架構設計模式。
