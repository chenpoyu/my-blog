---
layout: post
title: "部署 Jekyll 到 GitHub Pages"
date: 2022-02-09 23:05:00 +0800
categories: [心法]
tags: [Jekyll, GitHub Pages]
---

本地測試差不多了，今天把部落格部署到 GitHub Pages 上線。

先在 GitHub 建立一個新 repo，命名為 `my-blog`。因為不是 `username.github.io` 這種 User Page，所以網址會是 `chenpoyu.github.io/my-blog/`。

這裡有個重點，`_config.yml` 要設定 `baseurl`：

```yaml
baseurl: "/my-blog"
url: "https://chenpoyu.github.io"
```

不設定的話，所有的連結和資源路徑都會壞掉。

把本地的 repo 推上去：

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin git@github.com:chenpoyu/my-blog.git
git push -u origin main
```

接著到 repo 的 Settings > Pages，Source 選 `main` branch，資料夾選 `/ (root)`。按下 Save 後等個幾分鐘，GitHub 就會自動用 Jekyll 編譯並部署。

打開 `https://chenpoyu.github.io/my-blog/` 看到網站成功上線，心情還蠻爽的。

不過發現一個問題，每次要推上去都要等 GitHub 編譯，有點慢。查了一下可以用 GitHub Actions 自動化，改天來研究。
