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
├── archive.html         # 歸檔頁面
├── categories.html      # 分類頁面
├── tags.html            # 標籤頁面
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
- `csv`, `base64`, `logger` - Ruby 標準庫支援

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

## 文章分類

目前收錄的文章分類：
- **公告** - 網站相關公告
- **心法** - 技術學習心得
- **設計模式** - Java 設計模式實作筆記

## 授權

個人技術筆記，僅供學習參考。
