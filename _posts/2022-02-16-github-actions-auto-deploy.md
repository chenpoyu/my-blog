---
layout: post
title: "用 GitHub Actions 自動化部署"
date: 2022-02-16 22:20:00 +0800
categories: [心法]
tags: [GitHub Actions, CI/CD]
---

上週部署後發現 GitHub Pages 的內建編譯有些限制，而且速度不快。研究了一下 GitHub Actions，決定自己寫一個 workflow 來處理編譯和部署。

在 `.github/workflows/` 建立 `jekyll.yml`：

```yaml
name: Deploy Jekyll site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.1'
          bundler-cache: true
      
      - name: Build with Jekyll
        run: bundle exec jekyll build
        env:
          JEKYLL_ENV: production
      
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v1

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v1
```

這個 workflow 會在推送到 main branch 時觸發。先 checkout 程式碼，安裝 Ruby 和相依套件，執行 Jekyll 編譯，最後上傳到 GitHub Pages。

推上去後到 Actions 頁面看執行狀況。第一次跑大概花了 2 分鐘，之後因為有快取會更快。

這樣設定的好處是可以用任何 Jekyll 套件，不受 GitHub Pages 白名單限制。而且編譯過程透明，有問題也比較好除錯。

現在只要 `git push` 就能自動部署，整個流程順暢多了。
