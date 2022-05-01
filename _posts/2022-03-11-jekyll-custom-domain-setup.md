---
layout: post
title: "設定 Jekyll 部落格的自訂網域"
date: 2022-03-11 23:00:00 +0800
categories: [心法]
tags: [Jekyll, DNS]
---

用 `github.io` 的網址感覺不夠專業，今天研究怎麼綁定自訂網域。

首先要有一個網域名稱。我在某個註冊商買了一個 `.dev` 的網域，年費大概一千多塊。

到 DNS 管理介面設定：

1. 新增一筆 `CNAME` 記錄，主機名稱填 `blog`，值填 `chenpoyu.github.io`
2. 或是用 `A` 記錄指向 GitHub Pages 的 IP：
   - 185.199.108.153
   - 185.199.109.153
   - 185.199.110.153
   - 185.199.111.153

然後在 repo 的根目錄建立 `CNAME` 檔案，內容就是你的網域：

```
blog.example.dev
```

推上去後到 GitHub repo 的 Settings > Pages，Custom domain 欄位會自動帶入。勾選 "Enforce HTTPS"，讓 GitHub 自動申請 SSL 憑證。

DNS 傳播需要一點時間，我等了大概半小時才生效。之後打開自訂網域就能看到部落格了。

不過 `_config.yml` 也要改：

```yaml
url: "https://blog.example.dev"
baseurl: ""
```

因為已經是根網域了，所以 `baseurl` 要清空。改完重新部署，確認所有連結都正常。

現在部落格有專屬網址了，感覺又更進一步。
