# Poyu 的部落格

個人技術筆記部落格，使用 Jekyll 靜態網站生成器建立。

## 專案結構

```
my-blog/
├── _config.yml          # Jekyll 配置檔
├── _layouts/            # 頁面模板
│   ├── default.html     # 預設模板
│   └── post.html        # 文章模板
├── _posts/              # 文章目錄（Markdown 格式）
│   ├── 2016-*.md        # 2016 年技術筆記
│   ├── 2017-*.md        # 2017 年技術筆記
│   ├── 2018-*.md        # 2018 年技術筆記
│   ├── 2019-*.md        # 2019 年技術筆記
│   ├── 2020-*.md        # 2020 年技術筆記
│   ├── 2021-*.md        # 2021 年技術筆記
│   ├── 2022-*.md        # 2022 年技術筆記
│   ├── 2023-*.md        # 2023 年技術筆記
│   ├── 2024-*.md        # 2024 年技術筆記
│   ├── 2025-*.md        # 2025 年技術筆記
│   └── 2026-*.md        # 2026 年技術筆記
├── assets/              # 靜態資源
│   └── css/
│       └── style.css    # 自訂樣式
├── .github/             # GitHub Actions 配置
│   └── workflows/
│       └── jekyll.yml   # 自動部署設定
├── index.html           # 首頁
├── archive.html         # 歸檔頁面（時間軸）
├── taxonomy.html        # 分類與標籤統一頁面
├── Gemfile              # Ruby 套件依賴
└── _site/               # 生成的靜態網站（不需版控）
```

## 技術棧

- **Jekyll** 4.3.0 - 靜態網站生成器
- **Kramdown** - Markdown 解析器
- **Ruby** 3.1+ - 執行環境
- **GitHub Pages** - 部署平台
- **GitHub Actions** - 自動化部署

## 本地開發

### 環境需求

- Ruby 3.1 或更高版本
- Bundler

### 安裝步驟

```bash
# 安裝依賴套件
bundle install

# 本地啟動開發伺服器
bundle exec jekyll serve

# 或使用即時重載（檔案變更時自動重新編譯）
bundle exec jekyll serve --livereload

# 指定端口
bundle exec jekyll serve --port 4001
```

啟動後，在瀏覽器開啟 `http://localhost:4000/my-blog/`

### 新增文章

在 `_posts/` 目錄下建立新的 Markdown 檔案，檔名格式：

```
YYYY-MM-DD-title.md
```

檔案開頭需包含 Front Matter：

```yaml
---
layout: post
title: "文章標題"
date: YYYY-MM-DD HH:MM:SS +0800
categories: [分類名稱]
tags: [標籤1, 標籤2]
---

文章內容...
```

## 部署

### GitHub Pages（自動部署）

推送到 `main` 分支後，GitHub Actions 會自動執行部署：

```bash
git add .
git commit -m "更新內容"
git push origin main
```

部署完成後，網站會發布到：`https://chenpoyu.github.io/my-blog/`

### 手動建置

```bash
# 建置靜態網站
bundle exec jekyll build

# 輸出至 _site/ 目錄
```

## 配置說明

### _config.yml 主要設定

- `title` - 網站標題
- `description` - 網站描述
- `baseurl` - 子路徑（GitHub Pages 專案頁面需設定）
- `url` - 網站完整網址
- `permalink` - 文章連結格式

### Gemfile 套件

- `jekyll` - 核心套件
- `jekyll-feed` - RSS feed 生成
- `jekyll-seo-tag` - SEO 優化
- `jekyll-sitemap` - 自動生成 sitemap.xml
- `jekyll-redirect-from` - URL 重導向支援
- `csv`, `base64`, `logger` - Ruby 標準庫支援

## SEO 與網站地圖

### Sitemap.xml
網站會自動生成 `sitemap.xml`，由 `jekyll-sitemap` 插件處理：

- **自動生成**：每次建置時自動更新
- **訪問路徑**：`https://chenpoyu.github.io/my-blog/sitemap.xml`
- **配置位置**：`_config.yml` 中的 `sitemap` 區塊
- **排除頁面**：404 頁面已從 sitemap 中排除

### robots.txt
已配置 `robots.txt` 文件，包含：
- 允許所有搜尋引擎爬蟲
- 指向 sitemap.xml 的位置
- 設定爬蟲延遲為 1 秒

### SEO 優化
使用 `jekyll-seo-tag` 插件自動生成：
- Meta 標籤（title, description）
- Open Graph 標籤（社群分享）
- Twitter Card 標籤
- Schema.org JSON-LD 結構化資料

## 開發注意事項

### 本地預覽

使用 `bundle exec jekyll serve` 時，`baseurl` 設定會影響資源路徑。確保本地開發時的路徑正確。

### 快取問題

如果修改後沒有即時反映，可清除快取：

```bash
bundle exec jekyll clean
bundle exec jekyll serve
```

### 排除檔案

`.gitignore` 已設定排除：
- `_site/` - 生成的靜態檔案
- `.jekyll-cache/` - Jekyll 快取
- `vendor/` - Bundler 套件
- `.DS_Store` - macOS 系統檔案

## 頁面功能

### 🏠 首頁 (`/`)
- 顯示最新 10 篇文章
- 包含文章標題、日期、分類和摘要
- 提供「查看所有文章」連結

### 📚 分類與標籤 (`/taxonomy/`) ⭐ 新功能
統一的分類與標籤管理頁面，提供強大的瀏覽和搜尋功能：

#### 核心功能
- **三種視圖切換**：
  - 📁 分類檢視 - 僅顯示文章分類
  - 🏷️ 標籤檢視 - 僅顯示文章標籤
  - 📚 全部檢視 - 同時顯示分類和標籤

- **折疊展開設計**：
  - 所有區塊預設為折疊狀態
  - 點擊標題即可展開/折疊內容
  - 減少頁面滾動，提升瀏覽效率

- **快速搜尋** 🔍：
  - 即時搜尋功能（300ms 防抖）
  - 可搜尋分類名稱、標籤名稱或文章標題
  - 搜尋時自動展開匹配的區塊

- **重點標註** ⭐：
  - 重要分類/標籤會有金色邊框和星號標記
  - 預設展開狀態，方便快速存取
  - 重點項目包括：Java, Swift, iOS, Flutter, .NET, DevOps, Kubernetes 等

- **智慧導航**：
  - 從文章頁面點擊分類/標籤會自動跳轉並展開對應區塊
  - 支援 URL hash 定位（如 `/taxonomy/#java`）

- **統計資訊**：
  - 頁面頂部顯示分類總數、標籤總數和文章總數
  - 每個區塊顯示包含的文章數量

#### 自訂重點項目
可在 `taxonomy.html` 中修改以下變數來調整重點標註：

```liquid
{% assign featured_categories = "Java,Swift,iOS,Flutter,.NET,DevOps,Project Management" | split: "," %}
{% assign featured_tags = "Java,Swift,iOS,Flutter,.NET Core,Kubernetes,CI/CD,Performance" | split: "," %}
```

### 📅 時間軸 (`/archive/`)
- 依年份和月份歸檔所有文章
- 時間順序由新到舊

## 導航結構

網站主導航包含：
- **首頁** - 最新文章列表
- **分類 & 標籤** - 統一的分類與標籤瀏覽頁面
- **時間軸** - 依時間排序的文章歸檔

## 文章分類與標籤

## 文章分類與標籤

### 主要分類
- **Java** - Java 核心技術與框架（Spring, Hibernate 等）
- **Swift & iOS** - iOS 開發技術
- **Flutter** - 跨平台行動開發
- **.NET** - .NET Core 與 ASP.NET Core
- **DevOps** - CI/CD、容器化、自動化部署
- **Project Management** - 專案管理與團隊協作
- **設計模式** - 設計模式實作與應用
- **心法** - 技術學習心得與思考
- **前端開發** - JavaScript、React、Vue.js
- **系統架構** - 微服務、雲端架構、安全性

### 熱門標籤
包含但不限於：
- 程式語言：Java, Swift, JavaScript, Python
- 框架：Spring Boot, Flutter, React, Vue.js
- DevOps：Kubernetes, Docker, CI/CD, Jenkins, ArgoCD
- 雲端：AWS, Azure
- 資料庫：PostgreSQL, MySQL, MongoDB
- 工具：Git, n8n, Prometheus, Grafana

## UI/UX 設計理念

### 極簡風格
- 採用簡潔的卡片式設計
- 綠金配色（深綠 + 金色點綴）
- 清晰的視覺層次和留白

### 互動設計
- 折疊/展開動畫（平滑過渡）
- 懸停效果（顏色變化、位移）
- 即時搜尋反饋

### 響應式設計
- 完整支援桌面、平板和手機
- 自適應排版和字體大小
- 觸控友善的互動元素

## 最近更新

### 2026-02-16
- ✨ 新增統一的「分類與標籤」頁面 (`taxonomy.html`)
- 🔍 實作即時搜尋功能
- ⭐ 新增重點分類/標籤標註
- 📁 實作折疊/展開功能
- 🎯 智慧導航：支援 URL hash 自動定位
- 🎨 優化 UI/UX：採用極簡設計風格
- 📱 完整響應式支援

## 授權

個人技術筆記，僅供學習參考。
