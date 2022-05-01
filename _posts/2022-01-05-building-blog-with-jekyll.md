---
layout: post
title: "決定用 Jekyll 架設個人技術部落格"
date: 2022-01-05 23:15:00 +0800
categories: [心法]
tags: [Jekyll, 部落格]
---

最近開始思考要把這幾年累積的技術筆記整理出來，一直在猶豫要用什麼平台。Medium 太商業化，WordPress 又太肥大，後來看到同事推薦 Jekyll，研究了一下發現很適合我的需求。

Jekyll 是一個靜態網站生成器，用 Ruby 寫的。最大的優點是可以用 Markdown 寫文章，然後自動轉成靜態 HTML。不需要資料庫，也不用擔心伺服器被打掛。而且可以免費部署到 GitHub Pages，連主機費都省了。

今天花了一個晚上把基本環境架起來。先在 Mac 上裝 Ruby 3.1，然後用 gem 安裝 Jekyll 和 Bundler。過程中遇到一些權限問題，後來用 `--user-install` 參數解決了。

```bash
gem install --user-install bundler jekyll
```

接著建立新專案：

```bash
jekyll new my-blog
cd my-blog
bundle exec jekyll serve
```

打開 `http://localhost:4000` 就看到預設的部落格畫面了。雖然還很陽春，但至少跑起來了。

明天來研究一下怎麼客製化版面和樣式。
