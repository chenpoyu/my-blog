---
layout: post
title: "Keycloak 正式環境部署筆記"
date: 2024-02-19 10:05:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, Production, Database]
---

前面幾週都在測試環境研究 Keycloak，這週來看看如何部署到正式環境。測試環境用 `start-dev` 模式跑，資料存在內建的 H2 資料庫，重啟就不見了。正式環境當然不能這樣搞。

## 正式環境的考量

正式環境跟測試環境差很多：

**資料庫**：要用正式的關聯式資料庫（PostgreSQL 或 MySQL），而且要做備份。

**高可用性**：單台掛了整個 SSO 就掛了，所有系統都不能登入。要做 HA（High Availability）。

**HTTPS**：正式環境絕對不能用 HTTP，Token 會被截取。一定要 HTTPS。

**效能**：測試環境只有幾個人用，正式環境可能幾百幾千個使用者同時登入。

**監控**：要能即時知道 Keycloak 的狀態，出問題要能馬上發現。

這週先把資料庫和基本的 production mode 搞定，HA 和監控之後再說。

## 安裝 PostgreSQL

假設企業環境已經有 PostgreSQL，要在上面建立 Keycloak 用的資料庫：

```sql
CREATE DATABASE keycloak;
CREATE USER keycloak WITH PASSWORD 'strong_password_here';
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
```

Keycloak 會自動建立需要的 table，不用手動執行 schema。

## 準備 Keycloak 設定檔

正式環境不用 Docker 了，直接在 Linux 伺服器上跑。下載 Keycloak 的 binary：

```bash
wget https://github.com/keycloak/keycloak/releases/download/23.0.4/keycloak-23.0.4.tar.gz
tar xzf keycloak-23.0.4.tar.gz
cd keycloak-23.0.4
```

設定資料庫連線，編輯 `conf/keycloak.conf`：

```properties
# Database
db=postgres
db-username=keycloak
db-password=strong_password_here
db-url=jdbc:postgresql://db.mycompany.com:5432/keycloak

# HTTP/HTTPS
hostname=sso.mycompany.com
https-certificate-file=/etc/keycloak/cert.pem
https-certificate-key-file=/etc/keycloak/key.pem

# Admin
# 正式環境不要在這設定 admin 密碼，應該在第一次啟動時設定
```

## 建置 Keycloak

正式模式需要先 build：

```bash
bin/kc.sh build
```

這個指令會根據設定檔建置 Keycloak，產生最佳化的執行檔。執行時間蠻久的，要等一下。

## 取得 SSL 憑證

正式環境一定要 HTTPS。我們用 Let's Encrypt 取得免費的 SSL 憑證：

```bash
sudo certbot certonly --standalone -d sso.mycompany.com
```

憑證會產生在 `/etc/letsencrypt/live/sso.mycompany.com/`，把它複製到 Keycloak 的設定目錄：

```bash
sudo mkdir /etc/keycloak
sudo cp /etc/letsencrypt/live/sso.mycompany.com/fullchain.pem /etc/keycloak/cert.pem
sudo cp /etc/letsencrypt/live/sso.mycompany.com/privkey.pem /etc/keycloak/key.pem
sudo chown keycloak:keycloak /etc/keycloak/*.pem
```

## 啟動 Keycloak

第一次啟動要設定 admin 帳號：

```bash
export KEYCLOAK_ADMIN=admin
export KEYCLOAK_ADMIN_PASSWORD=複雜的密碼
bin/kc.sh start
```

看到 `Keycloak 23.0.4 started` 就代表啟動成功了。

開瀏覽器連 `https://sso.mycompany.com`，可以看到 Keycloak 的登入頁面。用剛才設定的 admin 帳號登入，確認能進管理介面。

## 設定成系統服務

每次手動啟動太麻煩，而且伺服器重開機後 Keycloak 不會自動啟動。要設定成 systemd service。

建立 `/etc/systemd/system/keycloak.service`：

```ini
[Unit]
Description=Keycloak Application Server
After=network.target

[Service]
Type=idle
User=keycloak
Group=keycloak
ExecStart=/opt/keycloak/bin/kc.sh start
TimeoutStartSec=600
TimeoutStopSec=600

[Install]
WantedBy=multi-user.target
```

啟用並啟動服務：

```bash
sudo systemctl daemon-reload
sudo systemctl enable keycloak
sudo systemctl start keycloak
sudo systemctl status keycloak
```

這樣伺服器重開機後，Keycloak 會自動啟動。

## 匯入測試環境的設定

如果測試環境已經建好了 realm、users、clients 這些設定，可以搬到正式環境。Keycloak 提供匯出匯入功能。

在測試環境匯出 realm：

1. 進入 `company` realm
2. 點 Realm settings → Actions → Partial export
3. 勾選要匯出的項目（users、clients、roles 等）
4. 點 Export，下載 JSON 檔

然後在正式環境匯入：

1. 點左上角 Create realm
2. 點 Browse 選擇剛才下載的 JSON 檔
3. 點 Create

這樣測試環境的設定就搬到正式環境了。不過要注意，Client 的 redirect URIs 要改成正式環境的網址。

## 效能調校

正式環境要調一些效能相關的參數。編輯 `conf/keycloak.conf`：

```properties
# JVM heap size
# 根據伺服器的記憶體調整，建議至少 2GB
JAVA_OPTS_APPEND="-Xms2048m -Xmx2048m"

# Database connection pool
db-pool-initial-size=20
db-pool-max-size=100
db-pool-min-size=10

# Cache
cache=ispn
cache-stack=tcp
```

Keycloak 預設使用 Infinispan 做 cache，可以加快 Token 驗證和使用者查詢的速度。

## 遇到的問題

一開始啟動時一直報錯：`Unable to start HTTP server`。查了半天才發現是防火牆沒開 8080 和 8443 port：

```bash
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload
```

另外 SSL 憑證的權限也要注意，如果 Keycloak 讀不到憑證檔，會啟動失敗。要確認 keycloak user 有讀取權限。

還有資料庫連線，一開始我用 `localhost`，結果連不上。要用伺服器的實際 IP 或 hostname。

## 安全性檢查清單

部署好後，要做一些安全性檢查：

- ✅ 使用 HTTPS，不能有 HTTP
- ✅ Admin 密碼夠複雜，而且定期更換
- ✅ 資料庫密碼夠複雜
- ✅ 防火牆只開必要的 port（8443 對外，8080 只允許內網）
- ✅ 定期更新 Keycloak 版本，修補安全漏洞
- ✅ 啟用登入失敗鎖定（Brute Force Detection）
- ✅ 設定 Token 的過期時間不要太長
- ⏳ 啟用審計日誌（Audit Log）- 還沒做
- ⏳ 定期備份資料庫 - 還沒做

## 下一步

基本的正式環境部署完成了，但還有一些要做的：

**高可用性**：至少要兩台 Keycloak，前面放 Load Balancer。Keycloak 是 stateless 的（session 存在 cache），很容易做 HA。

**監控和告警**：要接 Prometheus + Grafana，監控 Keycloak 的 JVM、資料庫連線、回應時間等等。

**備份機制**：PostgreSQL 要做定期備份，而且要測試還原流程。

**災難復原計畫**：萬一機房出事，要多久能復原？需要做異地備援嗎？

這些都是接下來要處理的。不過目前單機版的 Keycloak 已經可以運作了。先把基礎建設做好，再來優化。

總算把 Keycloak 從研究階段推到正式環境了。接下來想研究一些進階主題，像是多因子認證、社交登入整合等等。
