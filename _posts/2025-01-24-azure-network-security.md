---
layout: post
title: "Azure 網路架構：從公開到私有的安全升級之路"
date: 2025-01-24 16:45:00 +0800
categories: [雲端架構]
tags: [Azure, Network Security, Private Link, Front Door]
---

## 從一開始就要做對

去年看過太多系統因為「圖方便」而把資料庫開 public endpoint，結果被攻破的案例。

所以這次專案一開始，我就跟團隊明確要求：**所有內部服務都不能有公開端點**。

資料庫公開這種事，說實話是很低級的錯誤。我做了十三年的工程師，如果連這個都沒意識到，那也太丟臉了。

在系統架構會議上，我直接畫出我們的目標架構：

```
Internet → Front Door → Private Link → App Service (VNet) → Private Endpoint → SQL Database (private)
```

團隊裡有人問：「這樣會不會太複雜？要設定一堆 VNet、Subnet、Private Endpoint⋯⋯」

我的回答是：「複雜是相對的。一開始做對，總比之後出事再來改容易。而且這次處理的是會員個資，資安不能妥協。」

這不是技術追求，是基本的專業要求。

## 目標：完全私有化

理想的架構應該是這樣：

```
Internet → Front Door → Private Link → App Service (VNet) → Private Endpoint → SQL Database (private)
```

所有內部服務都在 VNet 裡面，不暴露公開 endpoint。只有 Front Door 是對外的入口，而且可以加上 WAF（Web Application Firewall）防護。

聽起來很美好，實作起來⋯⋯踩了不少坑。

## 第一步：建立 Virtual Network

這個部分還算簡單，就是在 Azure 上建一個 VNet，然後劃分幾個 subnet：

```
VNet: 10.0.0.0/16
├── AppServiceSubnet: 10.0.1.0/24      # 給 App Service 用
├── PrivateEndpointSubnet: 10.0.2.0/24 # 給 Private Endpoint 用
└── NATGatewaySubnet: 10.0.3.0/24      # 給 NAT Gateway 用
```

為什麼要分這麼多 subnet？

- **AppServiceSubnet**：App Service 整合到 VNet 時需要一個專屬的 subnet
- **PrivateEndpointSubnet**：Private Endpoint 要放的地方
- **NATGatewaySubnet**：讓 App Service 對外連線時有固定 IP（等等會講）

## 第二步：App Service 整合 VNet

App Service 預設是跑在 Azure 管理的多租戶環境裡，沒有 VNet 的概念。要把它「拉進」VNet 需要開啟 VNet Integration 功能。

```bash
az webapp vnet-integration add \
  --resource-group myResourceGroup \
  --name myAppService \
  --vnet myVNet \
  --subnet AppServiceSubnet
```

整合之後，App Service 就可以透過 VNet 存取內部資源了。

但有個問題：**App Service 對外連線的 IP 還是浮動的**。

這在某些情況下很麻煩。比如我們要連第三方的 payment gateway，對方要求我們提供固定 IP 加到白名單。但 App Service 的 outbound IP 有一堆，而且可能會變動。

解法：NAT Gateway。

## NAT Gateway：給 App Service 固定的出口 IP

NAT Gateway 可以讓 subnet 裡的資源共用一個固定的 public IP 對外連線。

設定步驟：

1. 建立一個 Public IP（記得選 Static）
2. 建立 NAT Gateway，掛上這個 Public IP
3. 把 NAT Gateway 指派給 AppServiceSubnet

```bash
# 建立 Public IP
az network public-ip create \
  --resource-group myResourceGroup \
  --name myNatGatewayIP \
  --sku Standard \
  --allocation-method Static

# 建立 NAT Gateway
az network nat gateway create \
  --resource-group myResourceGroup \
  --name myNatGateway \
  --public-ip-addresses myNatGatewayIP

# 指派給 subnet
az network vnet subnet update \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name AppServiceSubnet \
  --nat-gateway myNatGateway
```

這樣一來，App Service 對外連線就會走這個固定 IP。把這個 IP 給第三方廠商加白名單就搞定了。

不過 NAT Gateway 也不便宜，一個月大概 $40 美金（還沒算流量費）。但為了穩定性還是值得。

## Private Link：讓資料庫完全私有

接下來要處理資料庫。

預設的 SQL Database 有個 public endpoint（xxx.database.windows.net），雖然可以設防火牆規則，但本質上還是公開的。

用 Private Link 可以在 VNet 裡建立一個 Private Endpoint，這樣就只能從 VNet 內部存取資料庫了。

### 建立 Private Endpoint

在 Azure Portal 操作比較簡單：

1. 進入 SQL Database 的設定頁
2. 選擇「Private endpoint connections」
3. 新增一個 Private Endpoint，選擇 PrivateEndpointSubnet
4. Azure 會自動在那個 subnet 建立一個 network interface，配一個內部 IP（比如 10.0.2.4）

### 設定 Private DNS Zone

這是我覺得最容易搞混的部分。

Private Endpoint 建好之後，你的資料庫會有兩個連線方式：
- Public endpoint: `mydb.database.windows.net`（指向公開 IP）
- Private endpoint: `mydb.privatelink.database.windows.net`（指向內部 IP）

問題是，App Service 的 connection string 通常還是寫 `mydb.database.windows.net`。要怎麼讓它自動走 Private Endpoint？

答案：Private DNS Zone。

建立一個 Private DNS Zone（`privatelink.database.windows.net`），並且 link 到你的 VNet。Azure 會自動設定 DNS 解析，讓 `mydb.database.windows.net` 在 VNet 內部解析成內部 IP。

設定完之後，在 App Service 裡面測試一下：

```bash
# 在 App Service 的 SSH console
nslookup mydb.database.windows.net
```

應該會解析成 10.0.2.4（你的 Private Endpoint IP），而不是公開 IP。

### 關閉公開存取

確認 App Service 可以透過 Private Endpoint 連到資料庫之後，最後一步：**關閉資料庫的公開存取**。

在 SQL Database 的「Firewalls and virtual networks」設定中，選擇「Deny public network access」。

這樣一來，資料庫就完全私有化了。外部完全連不進來，只有 VNet 內部可以存取。

## Front Door：統一的入口與 CDN

現在內部都安全了，但 App Service 還是有個公開的網址（xxx.azurewebsites.net）。

雖然我們可以限制 App Service 只接受來自特定來源的流量，但更好的做法是在前面加一層 Front Door。

Front Door 是 Azure 的全球 CDN + load balancer + WAF，功能很強大。

### 設定流程

1. 建立 Front Door profile（選 Standard 或 Premium，我們用 Standard）
2. 新增 endpoint（xxx.azurefd.net）
3. 新增 origin group，把 App Service 的網址加進去
4. 設定 routing rules，把流量導到 origin group

完成後，使用者訪問 xxx.azurefd.net 就會被導到 App Service。

### 好處

- **CDN 加速**：靜態檔案會被 cache，減少 App Service 的負擔
- **WAF 防護**：可以擋 SQL injection、XSS 等常見攻擊
- **Global load balancing**：如果有多個 region 可以自動分流
- **SSL offloading**：HTTPS 在 Front Door 這層處理，App Service 可以用 HTTP

### 限制 App Service 只接受 Front Door

最後一步是設定 App Service 的 access restrictions，只允許來自 Front Door 的流量。

```bash
az webapp config access-restriction add \
  --resource-group myResourceGroup \
  --name myAppService \
  --priority 100 \
  --service-tag AzureFrontDoor.Backend
```

這樣就算有人知道 App Service 的網址（xxx.azurewebsites.net），直接訪問也會被擋下來。必須透過 Front Door 才能存取。

## Blob Storage 也要私有化

對了，我們還有用 Blob Storage 存一些檔案（使用者上傳的照片、文件等）。

Blob 預設也是公開的，一樣用 Private Endpoint 來處理。流程跟資料庫差不多：

1. 建立 Private Endpoint，選擇 PrivateEndpointSubnet
2. 設定 Private DNS Zone（`privatelink.blob.core.windows.net`）
3. 關閉公開存取

這樣 App Service 就可以透過內部網路存取 Blob，外部完全連不到。

## 最終架構

經過一番折騰，我們的架構變成這樣：

## 最終架構

按照規劃，我們一開始就把架構建成這樣：
Front Door (CDN + WAF)
  ↓
App Service (VNet Integration)
  ├→ Private Endpoint → SQL Database (private)
  ├→ Private Endpoint → Blob Storage (private)
  ├→ Private Endpoint → Redis Cache (private)
  └→ NAT Gateway → Third-party APIs (固定 IP)
```

所有內部服務都在 VNet 裡，外部完全連不到。只有 Front Door 是對外的，而且有 WAF 保護。

這個架構圖已經給客戶和資安顧問看過了，他們都很滿意。接下來就是實際建置和測試。

## 成本

這些東西當然不是免費的。我們每個月在網路這塊的花費：

- **NAT Gateway**: $40
- **Private Endpoint**: $7 x 3 = $21（資料庫、Blob、Redis 各一個）
- **Front Door**: $35（Standard tier）
- **Data transfer**: $50 左右（視流量而定）

總共大約 **$150/月**。

比起一開始什麼都沒有的架構，多了不少成本。但考慮到安全性，這錢花得值得。

## 接下來的計畫

Azure 的網路架構比 AWS 複雜一些（或者說「靈活」一些？）。

AWS 的 VPC、Security Group、NAT Gateway 概念都蠻直觀的。Azure 這邊 Private Link、Private Endpoint、Service Endpoint⋯⋯一堆名詞讓人搞混。

而且 Private DNS Zone 的設定也沒有很直覺，團隊花了好幾個小時才搞懂 DNS 解析的邏輯。

但搞懂之後，就會覺得這套架構確實蠻強大的。

身為團隊 lead，我的原則一直是：**資安問題不能妥協，該做的一開始就要做到位**。

是的，這會增加初期的設定成本。但總比未來系統上線後被駭，然後要緊急停機修補要好太多。

而且客戶看到你從一開始就注重資安，信任度也會提升。這個投資應該值得。

架構規劃好了，接下來就是實際建置。九個月的時間不算多，要好好把握。

下週會分享我們怎麼用 YARP 自建 API Gateway，取代 Azure API Management（因為太貴了）。
