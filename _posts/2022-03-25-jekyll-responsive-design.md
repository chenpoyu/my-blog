---
layout: post
title: "Jekyll 部落格的響應式設計調整"
date: 2022-03-25 21:00:00 +0800
categories: [心法]
tags: [Jekyll, CSS, RWD]
---

之前都在電腦上看，今天用手機打開發現版面整個跑掉。該來處理響應式設計了。

先在 `default.html` 加上 viewport meta tag：

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

然後在 CSS 加上 media query。主要處理這幾個地方：

1. 導覽列在小螢幕改成垂直排列
2. 文章寬度用百分比而不是固定像素
3. 字體大小依螢幕大小調整
4. 程式碼區塊要能水平捲動

```css
@media (max-width: 768px) {
  nav {
    flex-direction: column;
  }
  
  article {
    width: 95%;
    padding: 10px;
  }
  
  pre {
    overflow-x: auto;
  }
}
```

另外發現表格在手機上很容易超出螢幕。加上一個 wrapper div 並設定 `overflow-x: auto` 解決。

測試了 iPhone、iPad、各種 Android 裝置的模擬器，確認在不同解析度下都能正常顯示。

還有一個小細節，在手機上點選連結的觸控區域要夠大，至少 44x44px。把導覽列的連結加上 padding 增加可點擊範圍。

現在手機看起來也很順了。響應式設計真的很重要，畢竟現在大家都用手機看網頁。
