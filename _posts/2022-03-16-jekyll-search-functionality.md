---
layout: post
title: "為 Jekyll 部落格加入搜尋功能"
date: 2022-03-16 22:10:00 +0800
categories: [心法]
tags: [Jekyll, JavaScript]
---

文章數量多了之後，需要一個搜尋功能來快速找內容。Jekyll 是靜態網站，沒有後端資料庫，所以要用前端 JavaScript 來實作。

思路是先產生一個包含所有文章資料的 JSON 檔，然後用 JavaScript 載入並搜尋。

在根目錄建立 `search.json`：

```liquid
---
layout: null
---
[
  {% for post in site.posts %}
    {
      "title": "{{ post.title | escape }}",
      "url": "{{ post.url | relative_url }}",
      "date": "{{ post.date | date: '%Y-%m-%d' }}",
      "categories": {{ post.categories | jsonify }},
      "content": {{ post.content | strip_html | jsonify }}
    }{% unless forloop.last %},{% endunless %}
  {% endfor %}
]
```

Jekyll 會把這個檔案編譯成純 JSON。

然後寫一個簡單的搜尋頁面 `search.html`：

```html
<input type="text" id="search-input" placeholder="輸入關鍵字...">
<ul id="search-results"></ul>

<script>
  let posts = [];
  
  fetch('{{ "/search.json" | relative_url }}')
    .then(response => response.json())
    .then(data => {
      posts = data;
    });
  
  document.getElementById('search-input').addEventListener('input', function(e) {
    const query = e.target.value.toLowerCase();
    const results = posts.filter(post => 
      post.title.toLowerCase().includes(query) ||
      post.content.toLowerCase().includes(query)
    );
    
    displayResults(results);
  });
</script>
```

功能很基本，就是在標題和內文裡找關鍵字。之後可以考慮用 Lunr.js 或 Algolia 來做更進階的搜尋，不過現階段這樣就夠用了。

測試了一下，搜尋速度還可以接受。文章再多一點的話可能要做效能優化。
