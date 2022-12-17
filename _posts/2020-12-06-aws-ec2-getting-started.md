---
layout: post
title: "AWS EC2 入門：從零開始部署應用服務"
date: 2020-12-06 10:35:00 +0800
categories: [AWS, Cloud]
tags: [AWS, EC2, Cloud Computing, DevOps]
---

## 前言

作為一名多年開發 Java 和 .NET Core 應用的工程師，過去大多在地端機房或虛擬機器上部署服務。最近因為專案需求，開始接觸 AWS 雲端服務，第一站就是 EC2 (Elastic Compute Cloud)。這篇文章記錄我從零開始學習 EC2 的過程與心得。

## 為什麼選擇 EC2？

傳統的應用部署方式需要購買實體伺服器、安裝作業系統、設定網路等繁瑣步驟。EC2 提供了：

- **彈性擴展**：根據流量動態調整運算資源
- **按需付費**：只為實際使用的資源付費
- **快速部署**：幾分鐘內就能啟動一台伺服器
- **多種規格**：從小型測試環境到高效能運算都有對應方案

## 建立第一個 EC2 實例

### 1. 選擇 AMI (Amazon Machine Image)

我選擇了 Amazon Linux 2，因為它：
- 針對 AWS 環境優化
- 內建 AWS CLI
- 長期支援 (LTS)
- 預裝常用套件

### 2. 選擇實例類型

對於初期測試環境，我選擇了 `t2.micro`：
- 1 vCPU
- 1 GB 記憶體
- 符合 AWS 免費方案資格

未來正式環境可能會考慮 `t3.medium` 或 `c5.large`，取決於應用的 CPU/記憶體需求。

### 3. 設定安全群組 (Security Group)

這是一個容易忽略但非常重要的設定。我建立了以下規則：

```
# SSH 存取（僅限公司 IP）
Type: SSH
Protocol: TCP
Port: 22
Source: 203.x.x.x/32

# HTTP 存取
Type: HTTP
Protocol: TCP
Port: 80
Source: 0.0.0.0/0

# HTTPS 存取
Type: HTTPS
Protocol: TCP
Port: 443
Source: 0.0.0.0/0

# 自訂應用程式埠（例如 Spring Boot）
Type: Custom TCP
Protocol: TCP
Port: 8080
Source: 0.0.0.0/0
```

**安全提醒**：絕對不要將 SSH 埠開放給 `0.0.0.0/0`，這會成為安全漏洞。

### 4. 金鑰對 (Key Pair)

建立並下載 `.pem` 金鑰檔案後，記得設定正確權限：

```bash
chmod 400 my-ec2-key.pem
```

連線到 EC2：

```bash
ssh -i "my-ec2-key.pem" ec2-user@ec2-xx-xxx-xxx-xx.compute-1.amazonaws.com
```

## 部署 Java 應用程式

連上 EC2 後，我開始部署一個 Spring Boot 應用：

### 安裝 Java 11

```bash
sudo yum update -y
sudo amazon-linux-extras install java-openjdk11 -y
java -version
```

### 上傳應用程式

使用 SCP 上傳 JAR 檔：

```bash
scp -i "my-ec2-key.pem" my-app.jar ec2-user@ec2-xx-xxx-xxx-xx.compute-1.amazonaws.com:~/
```

### 建立 Systemd Service

為了讓應用程式開機自動啟動，建立 service 檔案：

```bash
sudo vi /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Spring Boot Application
After=syslog.target

[Service]
User=ec2-user
ExecStart=/usr/bin/java -jar /home/ec2-user/my-app.jar
SuccessExitStatus=143
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

啟動服務：

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
sudo systemctl status myapp.service
```

## 部署 .NET Core 應用程式

EC2 也能很好地支援 .NET Core：

### 安裝 .NET Core Runtime

```bash
sudo rpm -Uvh https://packages.microsoft.com/config/centos/7/packages-microsoft-prod.rpm
sudo yum install dotnet-runtime-3.1 -y
dotnet --version
```

### 執行應用程式

```bash
dotnet MyApp.dll --urls "http://0.0.0.0:5000"
```

同樣可以建立 systemd service 來管理 .NET Core 應用。

## Elastic IP 固定 IP 位址

預設情況下，EC2 重新啟動後 IP 會改變。為了避免這個問題，我配置了 Elastic IP：

1. 在 EC2 控制台選擇「彈性 IP」
2. 分配新地址
3. 關聯到 EC2 實例

這樣即使重啟實例，IP 位址也不會變動。

## 成本優化建議

初期使用 EC2 要注意的幾個成本陷阱：

1. **停止 vs 終止**
   - 停止實例：不收取運算費用，但 EBS 儲存仍計費
   - 終止實例：完全刪除，資料會遺失

2. **Elastic IP 閒置成本**
   - 只要 Elastic IP 沒有關聯到執行中的實例，就會計費

3. **資料傳輸費用**
   - 流出 AWS 的流量會計費
   - 同區域內的傳輸通常免費

## 監控與維護

基本的監控可以在 CloudWatch 看到：
- CPU 使用率
- 網路流量
- 磁碟讀寫

不過預設的 CloudWatch 監控間隔是 5 分鐘，如果需要更細緻的監控（1 分鐘），需要啟用詳細監控（額外收費）。

## 遇到的問題與解決

### 問題 1：無法 SSH 連線

**原因**：Security Group 沒有開放 SSH 埠或來源 IP 設定錯誤

**解決**：檢查並更新 Security Group 規則

### 問題 2：應用程式無法從外部存取

**原因**：
1. Security Group 沒有開放對應埠
2. 應用程式綁定在 localhost 而非 0.0.0.0
3. 防火牆（firewalld）阻擋

**解決**：
```bash
# 檢查防火牆狀態
sudo systemctl status firewalld

# 如果啟用，開放對應埠
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### 問題 3：磁碟空間不足

**原因**：預設 EBS 卷只有 8GB

**解決**：可以動態擴展 EBS 卷大小，不需要停機

## 下一步計劃

目前已經成功在 EC2 上部署了應用程式，但還有很多 AWS 服務可以探索：

- **Load Balancer**：處理流量分散
- **Auto Scaling**：自動擴展實例數量
- **RDS**：託管資料庫服務
- **Lambda**：無伺服器運算

最吸引我的是 Lambda，因為它可以按執行次數計費，對於某些不需要常駐的功能來說非常合適。

## 總結

EC2 是進入 AWS 生態系統的絕佳起點。雖然初期會覺得設定項目很多，但熟悉後發現它的彈性遠超傳統主機。對於習慣 Java 和 .NET Core 開發的工程師來說，EC2 提供了一個熟悉的 VM 環境，可以用傳統的方式部署應用程式。

不過我也開始思考：是否所有服務都需要用 EC2？也許某些輕量級的 API 可以用 Lambda 來實作，這樣能節省成本並減少維護負擔。

下一篇文章計畫分享 AWS Lambda 與 API Gateway 的使用經驗。

## 參考資料

- [AWS EC2 使用者指南](https://docs.aws.amazon.com/ec2/)
- [Amazon Linux 2 文件](https://aws.amazon.com/amazon-linux-2/)
- [AWS 定價計算機](https://calculator.aws/)
