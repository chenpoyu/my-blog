---
layout: post
title: "Jekyll 效能優化與編譯速度改善"
date: 2022-04-06 20:45:00 +0800
categories: [心法]
tags: [Jekyll, 效能優化]
---

文章越寫越多，發現本地編譯時間變長了。原本幾秒鐘就好，現在要等十幾秒。研究了一下 Jekyll 的效能優化方法。

首先是 `_config.yml` 的設定。加上這些可以減少不必要的處理：

```yaml
exclude:
  - Gemfile
  - Gemfile.lock
  - node_modules
  - vendor
  - .DS_Store
  - README.md

keep_files:
  - .git
  - .svn

lsi: false
safe: true
incremental: true
```

`incremental: true` 是關鍵，開啟增量編譯後，Jekyll 只會重新產生有變動的檔案。這讓開發時的重新編譯速度快很多。

另外發現 `jekyll-paginate-v2` 會拖慢編譯速度。如果只是本地開發，可以暫時停用：

```bash
bundle exec jekyll serve --config _config.yml,_config_dev.yml
```

在 `_config_dev.yml` 關閉分頁功能。

圖片也是效能殺手。之前都直接放原圖，有些檔案好幾 MB。現在用 ImageMagick 壓縮：

```bash
convert input.jpg -resize 1200x -quality 85 output.jpg
```

把所有圖片控制在 200KB 以內，網頁載入速度明顯變快。

還有一個技巧是用 `--profile` 參數找出編譯瓶頸：

```bash
bundle exec jekyll build --profile
```

會顯示每個檔案的處理時間。我發現有幾個很少用的 plugin 很耗時，就停用了。

現在編譯速度快多了，開發體驗好很多。
