---
layout: post
title: "整合留言系統到 Jekyll 部落格"
date: 2022-04-13 22:30:00 +0800
categories: [心法]
tags: [Jekyll, 第三方服務]
---

部落格上線這麼久，一直缺少互動功能。今天來加上留言系統。

靜態網站沒辦法自己處理留言，要用第三方服務。比較了幾個選項：

- Disqus：最多人用，但廣告很煩，而且追蹤使用者隱私
- Gitalk：用 GitHub Issues 當後端，蠻有趣的
- Utterances：也是用 GitHub Issues，但更輕量

最後選 Utterances，因為介面簡潔，而且開源。

到 [utteranc.es](https://utteranc.es/) 設定，步驟很簡單：

1. 在 GitHub 建立一個 public repo 存放 issues
2. 安裝 Utterances GitHub App
3. 產生嵌入程式碼

在 `_layouts/post.html` 的文章內容下方加上：

```html
<script src="https://utteranc.es/client.js"
        repo="chenpoyu/blog-comments"
        issue-term="pathname"
        theme="github-dark"
        crossorigin="anonymous"
        async>
</script>
```

`issue-term="pathname"` 會用文章路徑當作 issue 標題，這樣每篇文章的留言會對應到不同的 issue。

推上去後測試了一下。留言時需要用 GitHub 帳號登入，這樣可以避免垃圾留言，而且技術部落格的讀者大多有 GitHub 帳號。

唯一的小問題是 GitHub Issues 有 rate limit，但對小流量的個人部落格應該不會是問題。

現在讀者可以留言討論了，感覺完整度又提升了一些。
