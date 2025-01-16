---
layout: post
title: "從 AWS 跳槽到 Azure：一個不得不的決定"
date: 2025-01-17 14:30:00 +0800
categories: [雲端架構]
tags: [Azure, AWS, 雲端遷移, App Service]
---

## 說好的 AWS 呢？

如果你看過我之前的文章，應該知道我從 2020 年底開始用 AWS。EC2、RDS、S3、Lambda⋯⋯這些服務我都摸得挺熟的了。尤其是 2021 年那個物聯網專案，整套架構都跑在 AWS 上，雖然過程中踩了不少坑，但最後還是穩穩地跑起來了。

所以當我接到這個新專案的時候，理所當然地認為還是會用 AWS。

結果第一次開會，客戶就說：「我們公司統一用 Microsoft 的方案，雲端要用 Azure。」

我當場愣了兩秒。

不是說 AWS 不好，而是⋯⋯我根本沒碰過 Azure 啊。雖然都是雲端服務，但介面、術語、架構思維都不太一樣。這不就等於要重新學嗎？

但沒辦法，客戶說了算。只好硬著頭皮上。

## AWS 經驗能用多少？

剛開始接觸 Azure 的時候，我一直在找「這個對應 AWS 的哪個服務」。

這個習慣其實蠻有用的，至少讓我不用從零開始：

| 我熟悉的 AWS | 對應的 Azure | 差異點 |
|-------------|-------------|--------|
| EC2 | App Service / VM | App Service 是 PaaS，不用管 OS |
| RDS | SQL Database | 設定方式差蠻多，但概念類似 |
| S3 | Blob Storage | API 不同，但都是物件儲存 |
| ElastiCache | Azure Cache for Redis | 用法幾乎一樣 |
| CloudFront | Front Door | Front Door 功能更多，但也更複雜 |
| VPC | Virtual Network | 概念相同，但 Azure 的 subnet 規劃更細 |

看起來都有對應的服務，應該不難吧？

錯了。魔鬼藏在細節裡。

## 第一個大坑：App Service

在 AWS，我習慣用 EC2。需要什麼就自己裝，控制權完全在我手上。但 Azure 這邊，客戶希望我們用 App Service，理由是「比較好管理」。

App Service 是 PaaS（Platform as a Service），不用管 OS、不用處理 scaling、不用設定 load balancer⋯⋯聽起來很美好對吧？

但實際用起來，限制也很多：

### 限制 1：執行環境固定

App Service 只支援特定的 runtime。我們用 .NET Core 還好，但如果要裝一些客製化的工具或套件，就很麻煩。不像 EC2 想裝什麼就裝什麼。

### 限制 2：檔案系統不持久

一開始我天真地以為可以把暫存檔案寫到硬碟，結果發現 App Service 重啟以後檔案就不見了。後來才知道要用 Blob Storage 或者 mount 一個 persistent storage。

### 限制 3：冷啟動問題

App Service 有個「Always On」的設定，如果沒開的話，一段時間沒人訪問就會進入閒置狀態，下次請求會很慢（要等它重新啟動）。

這跟 AWS Lambda 的 cold start 問題有點像，但 Lambda 是設計成這樣的，App Service 卻讓人覺得是個 bug。

### 那為什麼還是選了 App Service？

抱怨歸抱怨，但 App Service 還是有它的優勢：

1. **部署超級簡單**：設定好 CI/CD，git push 就自動部署
2. **自動 scaling**：設定條件後，流量大的時候會自動多開幾個 instance
3. **整合度高**：跟其他 Azure 服務整合得很好（尤其是 Application Insights）
4. **省維護成本**：不用管 OS 更新、安全性 patch 這些雜事

而且最重要的是：**客戶買的是 App Service 的 reserved instance**，已經付錢了，不用白不用。

## 用 Docker 部署的好處

雖然 App Service 支援直接跑 .NET Core，但我們還是選擇用 Docker container 部署。

為什麼多此一舉？

### 1. 環境一致性

本機開發、CI/CD、Azure App Service 都跑同一個 Docker image，不用擔心「在我電腦上可以跑啊」的問題。

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.Api/MyApp.Api.csproj", "MyApp.Api/"]
RUN dotnet restore "MyApp.Api/MyApp.Api.csproj"
COPY . .
WORKDIR "/src/MyApp.Api"
RUN dotnet build "MyApp.Api.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.Api.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

這個 Dockerfile 在哪裡都能跑，要換到 AKS 或自己的機器也沒問題。

### 2. 未來彈性

雖然現在用 App Service，但如果哪天需要更多控制權（比如要用到 App Service 不支援的功能），可以直接把 container 搬到 AKS 或 VM，不用重新改 code。

### 3. 多階段建置

Docker 的 multi-stage build 可以讓最終 image 變很小。Build 的時候用 SDK image，最後只留下 runtime 和編譯好的檔案。

我們的 image 大小從一開始的 800MB 優化到現在 200MB 左右。

## SQL Database：貴但省事

資料庫方面，Azure 的 SQL Database 就是 hosted SQL Server。

跟 AWS RDS 比起來：

**優點**：
- 效能不錯（用的是 SSD）
- 自動備份和 point-in-time restore
- 內建高可用性（Business Critical 層級有 read replica）
- 跟 .NET 整合得很順（畢竟都是 Microsoft 家的）

**缺點**：
- **貴**。真的很貴。同樣規格比 RDS 貴大概 20-30%
- 有些 SQL Server 的功能被閹割了（比如不能用 SQL Agent）

我們選了 General Purpose 層級，配置是：
- vCore: 4
- Storage: 250GB
- Zone Redundant: 開啟（跨 AZ 容錯）

一個月大概 $600 美金。看到帳單的時候心都在淌血。

但客戶說沒關係，他們有 EA（Enterprise Agreement），折扣下來還可以接受。

## Redis：唯一不用改的東西

Azure Cache for Redis 跟 AWS ElastiCache 幾乎一模一樣。

連線字串格式有點不同，其他的 code 完全不用改：

```csharp
// AWS ElastiCache
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "my-cluster.abc123.use1.cache.amazonaws.com:6379";
});

// Azure Cache for Redis
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "my-cache.redis.cache.windows.net:6380,password=xxx,ssl=True";
});
```

唯一要注意的是 Azure 預設強制用 SSL，port 是 6380 而不是 6379。

## 下一步：網路架構

選好了運算、資料庫、快取，接下來要處理網路這塊。

Front Door、NAT Gateway、Private Link⋯⋯這些東西聽起來很複雜，實際用起來也確實不簡單。

但為了安全性和效能，該做的還是要做。

下週會分享我們怎麼設計 Azure 的網路架構，以及踩過的一些坑。

## 心得

從 AWS 轉到 Azure 沒有想像中那麼痛苦，畢竟核心概念是相通的。

但細節上的差異還是讓人很頭痛。我花了大概兩個禮拜才比較習慣 Azure 的操作邏輯和術語。

最大的感想是：**雲端供應商 lock-in 是真實存在的**。

雖然大家都說要「cloud agnostic」，實際上一旦用了某家的服務，要換到另一家的成本非常高。不只是技術上的改動，還有團隊的學習曲線、文件的更新、營運流程的調整⋯⋯

所以選雲端供應商真的要慎重。

不過話說回來，如果客戶指定，那也沒得選了（攤手）。

## 未來九個月的挑戰

專案總共要執行九個月,現在才剛開始。

Azure 的學習曲線比我預期的陡，但我相信團隊可以適應。

接下來要處理的重點：
- **網路架構**：VNet、Private Link、Front Door 要好好規劃
- **成本控制**：Azure 的計費方式跟 AWS 不太一樣，要仔細監控
- **CI/CD**：Azure DevOps 或 GitHub Actions？還在評估
- **監控告警**：Application Insights 要設定好

雖然 Azure 不是我的首選，但既然客戶選了，那就好好把它用好。

這九個月會是個考驗，但也是學習的機會。

下週會寫 Azure 的網路架構規劃，特別是 VNet、Private Link 這些東西怎麼設定。這部分跟 AWS 差蠻多的。
