---
layout: post
title: "React 專案部署與 CI/CD：從開發到上線的自動化流程"
date: 2022-07-29 11:20:00 +0800
categories: [前端開發]
tags: [React, CI/CD, Deployment, GitHub Actions, Nginx]
---

## 手動部署的痛苦

專案初期，我們的部署流程是這樣的：

1. 本機執行 `npm run build`
2. 把 `build/` 資料夾壓縮
3. 用 FileZilla 上傳到伺服器
4. SSH 登入伺服器，解壓縮
5. 重啟 Nginx

每次部署要 15-20 分鐘，而且容易出錯（忘記編譯、上傳錯檔案）。

必須建立自動化流程。

## 部署架構

我們的部署架構：

```
開發者 Push -> GitHub -> GitHub Actions -> 建置 -> 部署到伺服器 -> Nginx
```

**優點**：
- 自動化，減少人為錯誤
- 每次部署都有紀錄
- 可以快速回溯版本

## 步驟 1：設定環境變數

React 專案通常需要不同環境的設定（開發、測試、正式）。

建立環境變數檔案：

```bash
# .env.development
REACT_APP_API_URL=http://localhost:3000/api
REACT_APP_ENV=development

# .env.production
REACT_APP_API_URL=https://api.example.gov.tw/api
REACT_APP_ENV=production
```

在程式碼中使用：

```javascript
const API_URL = process.env.REACT_APP_API_URL;

axios.get(`${API_URL}/users`);
```

**重點**：環境變數必須以 `REACT_APP_` 開頭，Create React App 才會打包進去。

## 步驟 2：建立 GitHub Actions Workflow

在專案根目錄建立 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to Production

on:
  push:
    branches:
      - main  # 推送到 main 分支時觸發

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. 取得程式碼
      - name: Checkout code
        uses: actions/checkout@v3

      # 2. 設定 Node.js 環境
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'

      # 3. 安裝依賴
      - name: Install dependencies
        run: npm ci

      # 4. 執行測試
      - name: Run tests
        run: npm test -- --watchAll=false

      # 5. 建置專案
      - name: Build project
        run: npm run build
        env:
          REACT_APP_API_URL: ${{ secrets.PROD_API_URL }}
          REACT_APP_ENV: production

      # 6. 部署到伺服器
      - name: Deploy to server
        uses: easingthemes/ssh-deploy@v2.1.5
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
          REMOTE_USER: ${{ secrets.REMOTE_USER }}
          SOURCE: "build/"
          TARGET: "/var/www/html/app"
```

## 步驟 3：設定 GitHub Secrets

在 GitHub 專案設定中加入機敏資訊：

1. 進入 **Settings > Secrets and variables > Actions**
2. 新增以下 Secrets：
   - `PROD_API_URL`：正式環境 API 網址
   - `SSH_PRIVATE_KEY`：SSH 私鑰
   - `REMOTE_HOST`：伺服器 IP
   - `REMOTE_USER`：SSH 使用者名稱

**產生 SSH 金鑰**：

```bash
ssh-keygen -t rsa -b 4096 -C "deploy@example.com"
```

將公鑰加到伺服器的 `~/.ssh/authorized_keys`。

## 步驟 4：設定 Nginx

伺服器上的 Nginx 設定：

```nginx
# /etc/nginx/sites-available/app
server {
    listen 80;
    server_name app.example.gov.tw;

    root /var/www/html/app;
    index index.html;

    # React Router 的 SPA 路由支援
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 反向代理
    location /api/ {
        proxy_pass http://backend-server:8080/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 快取靜態資源
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

啟用設定：

```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 步驟 5：測試部署流程

推送程式碼到 `main` 分支：

```bash
git add .
git commit -m "feat: add new feature"
git push origin main
```

GitHub Actions 會自動：
1. 執行測試
2. 建置專案
3. 部署到伺服器

可以在 GitHub 的 **Actions** 頁籤查看執行狀態。

## 進階：分支部署策略

我們採用的分支策略：

- `develop`：開發環境，自動部署到測試伺服器
- `staging`：預發布環境，部署到 UAT 伺服器
- `main`：正式環境，部署到正式伺服器

不同分支的 Workflow：

```yaml
on:
  push:
    branches:
      - develop
      - staging
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # ...
      - name: Deploy
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "Deploying to production"
            # 部署到正式環境
          elif [ "${{ github.ref }}" == "refs/heads/staging" ]; then
            echo "Deploying to staging"
            # 部署到 UAT
          else
            echo "Deploying to development"
            # 部署到測試環境
          fi
```

## 效能優化：快取與壓縮

### 1. Gzip 壓縮

在 Nginx 中啟用 Gzip：

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_min_length 1000;
```

### 2. Build 時的程式碼分割

React 預設會做 code splitting，但可以進一步優化：

```javascript
// 使用 React.lazy 動態載入元件
const UserPage = React.lazy(() => import('./pages/UserPage'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Route path="/users" element={<UserPage />} />
    </Suspense>
  );
}
```

### 3. 分析打包大小

```bash
npm install --save-dev webpack-bundle-analyzer
```

在 `package.json` 加入：

```json
"scripts": {
  "analyze": "source-map-explorer 'build/static/js/*.js'"
}
```

執行 `npm run analyze` 可以看到各模組的大小。

## 監控與回溯

### 部署通知

在 Workflow 加入 Slack 通知：

```yaml
- name: Notify Slack
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: 'Deployment to production ${{ job.status }}'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### 版本回溯

每次部署都加上 tag：

```bash
git tag -a v1.2.3 -m "Release version 1.2.3"
git push origin v1.2.3
```

需要回溯時：

```bash
git checkout v1.2.2
# 觸發部署流程
```

## 實際效益

自動化部署後：
- **部署時間**：從 15 分鐘降到 3 分鐘
- **錯誤率**：幾乎為零（自動化流程）
- **可追蹤性**：每次部署都有紀錄

最重要的是，開發者可以專注在寫程式，不用擔心部署問題。

## 小結

CI/CD 不只是自動化工具，而是一種開發文化：
- 頻繁整合，減少合併衝突
- 自動測試，確保品質
- 快速部署，縮短反饋週期

下週會開始分享其他技術領域的內容，React 系列暫時告一段落。

這三個月從 Vue 轉到 React 的旅程，收穫很多，也建立了一套完整的開發流程。
