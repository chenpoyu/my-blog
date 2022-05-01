---
layout: post
title: "Jekyll 套件與 SEO 優化"
date: 2022-02-25 21:45:00 +0800
categories: [心法]
tags: [Jekyll, SEO]
---

網站上線後開始想辦法讓它更容易被搜尋引擎找到。Jekyll 有幾個實用的套件可以幫忙。

在 `Gemfile` 加入這些套件：

```ruby
gem "jekyll-feed"
gem "jekyll-seo-tag"
gem "jekyll-sitemap"
```

`jekyll-feed` 會自動產生 RSS feed，訂閱用戶可以追蹤更新。`jekyll-seo-tag` 會在 HTML head 加入各種 meta tags，包括 Open Graph 和 Twitter Card。`jekyll-sitemap` 會產生 sitemap.xml 給搜尋引擎爬蟲。

然後在 `_config.yml` 啟用：

```yaml
plugins:
  - jekyll-feed
  - jekyll-seo-tag
  - jekyll-sitemap
```

在 `_layouts/default.html` 的 `<head>` 區塊加上：

```liquid
{% seo %}
{% feed_meta %}
```

這樣就會自動插入所有必要的標籤。

另外在 `_config.yml` 補充一些 SEO 相關資訊：

```yaml
title: Poyu 的部落格
description: 個人技術筆記部落格
author: Poyu Chen
lang: zh-TW
```

重新編譯後檢查 HTML 原始碼，確認 meta tags 都有正確產生。sitemap 可以在 `/sitemap.xml` 看到。

最後到 Google Search Console 提交 sitemap，讓 Google 知道這個網站的存在。雖然不知道會不會有人看，至少該做的都做了。
