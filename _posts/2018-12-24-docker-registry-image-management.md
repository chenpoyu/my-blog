---
layout: post
title: "Docker Registry 私有映像檔倉庫"
date: 2018-12-24 14:15:00 +0800
categories: [DevOps, Container]
tags: [Docker, Registry, Harbor, Private Repository]
---

前幾週學會了 Docker 和 Docker Compose，但映像檔都存在本機。這樣有幾個問題：

1. CI server 建置的映像檔，測試環境怎麼拿到？
2. 公司內部的應用程式，不想放到公開的 Docker Hub
3. 每台機器都要重新 build 一次，浪費時間

這週來建立私有的 Docker Registry，解決映像檔分享和管理的問題。

> 使用版本：Docker Registry 2.7

## Docker Hub 的限制

Docker Hub 是官方的公開映像檔倉庫，很方便，但有些限制：

1. **免費帳號只能有一個私有 repository**（我們有 5 個專案）
2. **安全性考量**：公司內部應用不適合放公開平台
3. **網路速度**：從國外 pull 映像檔有時候很慢
4. **版本控制**：缺乏細緻的權限管理

所以需要自己架設私有 Registry。

## Docker Registry 架構

Docker Registry 是一個無狀態的伺服器應用，用來儲存和分發 Docker 映像檔。

基本概念：
- **Registry**：映像檔倉庫服務
- **Repository**：映像檔儲存位置（例如 `my-shop-api`）
- **Tag**：映像檔版本標籤（例如 `1.0.0`、`latest`）

完整映像檔名稱格式：
```
[registry-host]:[port]/[repository]:[tag]

# 範例
registry.mycompany.com:5000/my-shop-api:1.0.0
```

## 快速啟動 Registry

最簡單的方式，用官方映像檔啟動：

```bash
docker run -d \
  -p 5000:5000 \
  --name registry \
  --restart=always \
  registry:2.7
```

現在 Registry 已經在 `localhost:5000` 運行了！

### 測試推送映像檔

```bash
# 1. 建置映像檔
docker build -t my-shop-api:1.0.0 .

# 2. 標記映像檔（指向私有 registry）
docker tag my-shop-api:1.0.0 localhost:5000/my-shop-api:1.0.0

# 3. 推送到私有 registry
docker push localhost:5000/my-shop-api:1.0.0

# The push refers to repository [localhost:5000/my-shop-api]
# 1.0.0: digest: sha256:abc123... size: 2827
```

### 從 Registry 拉取映像檔

在另一台機器（或刪除本機映像檔後）：

```bash
# 拉取映像檔
docker pull localhost:5000/my-shop-api:1.0.0

# 執行
docker run -d -p 8080:8080 localhost:5000/my-shop-api:1.0.0
```

成功！

## 持久化儲存

預設情況下，映像檔存在容器內部，容器刪除後映像檔就不見了。

使用 volume 持久化：

```bash
docker run -d \
  -p 5000:5000 \
  --name registry \
  --restart=always \
  -v /opt/registry/data:/var/lib/registry \
  registry:2.7
```

現在映像檔會存在 host 的 `/opt/registry/data` 目錄。

## 使用 Docker Compose 管理

建立 `docker-compose.registry.yml`：

```yaml
version: '3.7'

services:
  registry:
    image: registry:2.7
    container_name: docker-registry
    ports:
      - "5000:5000"
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      - registry-data:/data
    restart: always

volumes:
  registry-data:
```

啟動：
```bash
docker-compose -f docker-compose.registry.yml up -d
```

## 加入基本認證

預設的 Registry 沒有認證，任何人都能 push/pull，不安全！

### 產生密碼檔

```bash
mkdir -p /opt/registry/auth

docker run --entrypoint htpasswd \
  registry:2.7 -Bbn admin secretpass > /opt/registry/auth/htpasswd
```

這會產生一個使用者 `admin`，密碼 `secretpass`。

### 啟用認證

```yaml
version: '3.7'

services:
  registry:
    image: registry:2.7
    container_name: docker-registry
    ports:
      - "5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      - registry-data:/data
      - /opt/registry/auth:/auth
    restart: always

volumes:
  registry-data:
```

重新啟動：
```bash
docker-compose -f docker-compose.registry.yml up -d
```

### 登入 Registry

```bash
docker login localhost:5000
# Username: admin
# Password: secretpass
# Login Succeeded
```

現在只有登入的使用者才能 push/pull 映像檔。

## HTTPS 支援

HTTP 的 Registry 不安全，而且某些 Docker 版本預設不允許使用非 HTTPS 的 registry。

### 產生自簽名憑證（測試用）

```bash
mkdir -p /opt/registry/certs

openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout /opt/registry/certs/domain.key \
  -x509 -days 365 \
  -out /opt/registry/certs/domain.crt \
  -subj "/C=TW/ST=Taipei/L=Taipei/O=MyCompany/CN=registry.mycompany.local"
```

### 設定 HTTPS

```yaml
version: '3.7'

services:
  registry:
    image: registry:2.7
    container_name: docker-registry
    ports:
      - "443:443"
    environment:
      REGISTRY_HTTP_ADDR: 0.0.0.0:443
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      - registry-data:/data
      - /opt/registry/auth:/auth
      - /opt/registry/certs:/certs
    restart: always

volumes:
  registry-data:
```

### 信任自簽名憑證

在使用 Registry 的機器上：

**macOS:**
```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  /opt/registry/certs/domain.crt
```

**Linux:**
```bash
sudo mkdir -p /etc/docker/certs.d/registry.mycompany.local
sudo cp /opt/registry/certs/domain.crt \
  /etc/docker/certs.d/registry.mycompany.local/ca.crt
sudo systemctl restart docker
```

## Harbor - 企業級 Registry

基本的 Docker Registry 功能很陽春，缺少：
- 網頁 UI
- 映像檔掃描（漏洞檢測）
- 複製功能
- 垃圾回收
- 細緻的權限控制

**Harbor** 是 VMware 開源的企業級 Docker Registry，補足了這些功能。

> Harbor 是基於 Docker Registry 的增強版，2018 年底已經相當成熟

### Harbor 特色

1. **網頁管理介面**：不用下指令，視覺化管理
2. **角色權限管理**：可以設定哪些使用者能 push/pull 哪些 repository
3. **映像檔掃描**：整合 Clair，自動掃描映像檔的安全漏洞
4. **映像檔複製**：可以在多個 Harbor instance 間同步
5. **審計日誌**：追蹤所有操作記錄

### 安裝 Harbor

下載 Harbor offline installer：

```bash
wget https://github.com/goharbor/harbor/releases/download/v1.7.0/harbor-offline-installer-v1.7.0.tgz
tar xvf harbor-offline-installer-v1.7.0.tgz
cd harbor
```

編輯 `harbor.cfg`：

```bash
# 主機名稱
hostname = registry.mycompany.com

# 管理員密碼
harbor_admin_password = Harbor12345

# 資料庫密碼
db_password = root123

# 資料儲存路徑
data_volume = /opt/harbor/data
```

執行安裝腳本：

```bash
sudo ./install.sh
```

Harbor 會自動：
1. 載入所需的 Docker 映像檔
2. 產生設定檔
3. 啟動所有服務（透過 docker-compose）

### 訪問 Harbor

打開瀏覽器：`http://registry.mycompany.com`

預設帳號：
- Username: `admin`
- Password: `Harbor12345`

### 使用 Harbor

**1. 建立專案**

登入後，點「New Project」：
- Project Name: `myshop`
- Access Level: 選「Private」（私有專案）

**2. 建立使用者**

「Administration」→「Users」→「New User」
- Username: `developer`
- Password: `dev123456`

**3. 加入專案成員**

進入「myshop」專案 →「Members」→「+ User」
- 選擇 `developer`
- Role: `Developer`（可以 push/pull）

**4. 推送映像檔**

```bash
# 登入 Harbor
docker login registry.mycompany.com
# Username: developer
# Password: dev123456

# 標記映像檔
docker tag my-shop-api:1.0.0 registry.mycompany.com/myshop/my-shop-api:1.0.0

# 推送
docker push registry.mycompany.com/myshop/my-shop-api:1.0.0
```

在 Harbor UI 上就能看到映像檔了！

**5. 掃描映像檔**

在 Harbor UI 中，點擊映像檔 → 點「Scan」按鈕

Harbor 會使用 Clair 掃描映像檔，檢查是否有已知的 CVE 漏洞。

掃描結果會顯示：
- High: 3（高風險漏洞）
- Medium: 12（中風險）
- Low: 25（低風險）

可以點進去看詳細的 CVE 編號和修復建議。

## 整合到 CI/CD

### Jenkins Pipeline 推送映像檔

修改 Jenkinsfile：

```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY = 'registry.mycompany.com'
        IMAGE_NAME = 'myshop/my-shop-api'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'harbor-credentials') {
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }
    }
}
```

在 Jenkins 中設定 Credentials：
1. 「Manage Jenkins」→「Manage Credentials」
2. 新增「Username with password」
3. ID: `harbor-credentials`
4. Username: `developer`
5. Password: `dev123456`

## 映像檔管理最佳實踐

### 1. 使用有意義的 Tag

❌ 不好：
```bash
my-shop-api:latest
my-shop-api:v1
```

✅ 好：
```bash
my-shop-api:1.2.3
my-shop-api:1.2.3-20181224
my-shop-api:1.2.3-abc123f  # 加上 git commit hash
```

### 2. 多階段標記

同一個映像檔打多個標籤：

```bash
docker tag my-shop-api:build-123 registry.mycompany.com/myshop/my-shop-api:1.0.0
docker tag my-shop-api:build-123 registry.mycompany.com/myshop/my-shop-api:1.0
docker tag my-shop-api:build-123 registry.mycompany.com/myshop/my-shop-api:1
docker tag my-shop-api:build-123 registry.mycompany.com/myshop/my-shop-api:latest
```

這樣使用者可以：
- `1.0.0`：鎖定特定版本
- `1.0`：使用 1.0.x 的最新版
- `latest`：永遠使用最新版

### 3. 定期清理舊映像檔

映像檔會越來越多，佔用大量空間。

Harbor 有「Tag Retention」功能，可以設定保留規則：
- 保留最新 10 個版本
- 保留最近 30 天的版本
- 刪除沒有 tag 的映像檔

### 4. 使用映像檔掃描

每次 push 後自動掃描：

Harbor 可以設定「Automatically scan images on push」，每次有新映像檔就自動掃描。

如果發現高風險漏洞，可以設定「Prevent vulnerable images from running」，禁止部署有漏洞的映像檔。

## 遇到的問題

### 問題一：Cannot push to insecure registry

Docker 預設不允許使用 HTTP 的 registry。

錯誤訊息：
```
Error response from daemon: Get https://registry.mycompany.com/v2/: 
http: server gave HTTP response to HTTPS client
```

解決方法：設定 insecure registry（僅測試環境）

編輯 `/etc/docker/daemon.json`：
```json
{
  "insecure-registries": ["registry.mycompany.com:5000"]
}
```

重啟 Docker：
```bash
sudo systemctl restart docker
```

**生產環境務必使用 HTTPS！**

### 問題二：磁碟空間不足

Registry 的映像檔會佔用大量空間，一不小心就塞爆硬碟。

解決方法：啟用垃圾回收

編輯 Registry 設定，加入：
```yaml
storage:
  delete:
    enabled: true
```

手動執行垃圾回收：
```bash
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

Harbor 的話，在 UI 上有「Garbage Collection」功能。

### 問題三：網路速度慢

從 Registry pull 映像檔很慢。

解決方法：
1. 在每個機房部署 Registry mirror
2. 使用 Harbor 的 replication 功能同步
3. 優化映像檔大小（使用 alpine、multi-stage build）

## 心得

有了私有 Registry 後，CI/CD 流程變得更順暢了：

1. Jenkins 建置並 push 映像檔到 Harbor
2. 測試環境直接 pull 映像檔部署
3. 測試通過後，同一個映像檔部署到生產環境

而且 Harbor 的映像檔掃描功能很實用，能及早發現安全漏洞。上週掃描時發現我們用的 base image 有高風險漏洞，趕緊更新了。

下週要研究 Kubernetes，真正實現容器編排和自動擴展。Docker Compose 適合單機環境，但多台機器的叢集管理就需要 Kubernetes 了。
