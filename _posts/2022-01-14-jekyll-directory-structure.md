---
layout: post
title: "理解 Jekyll 的目錄結構"
date: 2022-01-14 22:40:00 +0800
categories: [心法]
tags: [Jekyll]
---

這週花時間把 Jekyll 的目錄結構搞清楚。一開始看到一堆底線開頭的資料夾有點困惑，不過看完官方文件後發現邏輯很清楚。

主要的目錄分成幾個部分：

`_posts/` 資料夾放所有文章，檔名格式必須是 `YYYY-MM-DD-title.md`。這個命名規則很嚴格，不符合就不會被處理。每個文章開頭要有 Front Matter，用三個 dash 包起來的 YAML 格式資料。

`_layouts/` 放頁面模板。我目前只用到 `default.html` 和 `post.html` 兩個。default 是基礎版型，post 繼承 default 然後加上文章特有的元素，像是標題、日期、分類標籤之類的。

`_site/` 是 Jekyll 編譯後的輸出資料夾。這個不用放進 git，因為每次 build 都會重新生成。記得加到 `.gitignore` 裡面。

`_config.yml` 是整個網站的設定檔。可以設定網站標題、描述、baseurl 等等。改了這個檔案要重啟 Jekyll server 才會生效，不像其他檔案可以 live reload。

另外還有 `assets/` 放 CSS、圖片等靜態資源。我把自己寫的樣式放在 `assets/css/style.css`。

目前基本架構算是掌握了，可以開始客製化版面了。
