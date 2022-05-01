---
layout: post
title: "Jekyll 部落格加入流量分析與監控"
date: 2022-04-22 21:15:00 +0800
categories: [心法]
tags: [Jekyll, Analytics]
---

部落格運作一段時間了，想知道有多少人在看，哪些文章比較受歡迎。今天來加上流量分析工具。

Google Analytics 是標準選項，但聽說對隱私不太友善。找了一下發現 Plausible Analytics 是個不錯的替代方案，輕量、尊重隱私、符合 GDPR。

不過 Plausible 要付費，小流量的話一個月 9 美金。還有一個開源的選項是 Umami，可以自己架。但我懶得維護伺服器，最後還是用了 Google Analytics。

申請 GA4 帳號，拿到 Measurement ID。在 `_layouts/default.html` 的 `</head>` 前面加上追蹤碼：

```html
<!-- Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-XXXXXXXXXX');
</script>
```

為了不要在本地開發時觸發追蹤，加上判斷：

```liquid
{% if jekyll.environment == "production" %}
  <!-- Google Analytics code -->
{% endif %}
```

這樣只有在 production build 時才會載入 GA。

另外也設定了 Google Search Console，可以看到搜尋關鍵字、點擊率、索引狀態等資訊。把 sitemap 提交上去，讓 Google 更容易爬到文章。

現在可以追蹤流量數據了。雖然可能沒什麼人看，但至少有個數字可以參考。

至此，整個部落格系統算是完成了。從一月初開始研究 Jekyll，到現在四個月過去，該有的功能都有了。接下來就是持續寫文章，累積內容。
