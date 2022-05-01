---
layout: post
title: "實作 Jekyll 文章分頁功能"
date: 2022-03-02 20:30:00 +0800
categories: [心法]
tags: [Jekyll]
---

文章越來越多，首頁全部列出來太長了。今天來加上分頁功能。

Jekyll 有內建的 `jekyll-paginate` 套件，不過只支援簡單的分頁。查了一下發現 `jekyll-paginate-v2` 功能比較完整，支援分類、標籤的分頁。

在 `Gemfile` 加入：

```ruby
gem "jekyll-paginate-v2"
```

`_config.yml` 設定：

```yaml
plugins:
  - jekyll-paginate-v2

pagination:
  enabled: true
  per_page: 10
  permalink: '/page/:num/'
  title: ':title - 第 :num 頁'
  sort_reverse: true
```

把 `index.html` 改成這樣：

```liquid
---
layout: default
pagination:
  enabled: true
---

{% for post in paginator.posts %}
  <article>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <time>{{ post.date | date: "%Y-%m-%d" }}</time>
    <p>{{ post.excerpt }}</p>
  </article>
{% endfor %}

{% if paginator.total_pages > 1 %}
<nav class="pagination">
  {% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path | relative_url }}">上一頁</a>
  {% endif %}
  
  <span>第 {{ paginator.page }} / {{ paginator.total_pages }} 頁</span>
  
  {% if paginator.next_page %}
    <a href="{{ paginator.next_page_path | relative_url }}">下一頁</a>
  {% endif %}
</nav>
{% endif %}
```

測試了一下，分頁正常運作。不過發現本地開發時 `bundle exec jekyll serve` 不會自動偵測分頁變化，要加上 `--livereload` 參數才行。

之後如果要做分類或標籤的分頁，也可以用類似的方式處理。
