---
layout: post
title: "客製化 Jekyll 版面與樣式"
date: 2022-01-19 21:30:00 +0800
categories: [心法]
tags: [Jekyll, CSS]
---

今天開始動手改版面。預設的 minima 主題太簡單，想要弄得更有個人風格一點。

先從 `_layouts/default.html` 下手。Jekyll 用 Liquid 模板語言，語法跟 Jinja2 有點像。可以用雙大括號輸出變數，用大括號加百分比寫邏輯。

在 header 加了導覽列，列出首頁、歸檔、分類、標籤四個頁面：

```html
<nav>
  <a href="{{ site.baseurl }}/">首頁</a>
  <a href="{{ site.baseurl }}/archive.html">歸檔</a>
  <a href="{{ site.baseurl }}/categories.html">分類</a>
  <a href="{{ site.baseurl }}/tags.html">標籤</a>
</nav>
```

接著建立這些頁面。`archive.html` 按時間倒序列出所有文章，`categories.html` 和 `tags.html` 用迴圈把文章依照 Front Matter 的分類和標籤分組顯示。

寫 CSS 的時候踩了一個坑。本地開發用 `http://localhost:4000`，但部署到 GitHub Pages 會在 `username.github.io/repo-name/` 底下。所以 CSS 路徑要用 `{{ site.baseurl }}/assets/css/style.css`，不能寫死絕對路徑。

版面配色選了深色背景配淺色文字，看起來比較不傷眼。程式碼區塊用 `pre` 和 `code` 標籤，套用 monospace 字體。

目前看起來順眼多了，改天再來調整 RWD。
