---
layout: post
title: "把 n8n 機器人搬到 LINE——自己的 LINE Bot 自己做"
date: 2025-12-19 19:55:00 +0800
categories: [技術筆記, Side Project]
tags: [n8n, LINE Bot, LINE Business Message API, 自動化]
---

## Slack 很好，但我還是比較常看 LINE

前兩週做了兩個 n8n 機器人——一個每天教英文，一個每天推技術快報。都是推到 Slack 上面。

Slack 當然方便，尤其在工作場景，很多公司都在用。但說真的，這兩個 bot 產出的東西比較偏我個人的，不是團隊的。下班之後我不太會特別開 Slack，但 LINE 是一定會滑的。

每天要看的東西放在每天一定會打開的地方，這個邏輯很簡單。

所以這週的目標就是：串 LINE，做一個自己的 LINE Bot，把之前的東西都接過去。

## LINE Business Messaging API

LINE 要做 Bot 的話，要走 LINE Business Messaging API。流程大概是：

1. 去 [LINE Developers](https://developers.line.biz/) 開一個 Provider 跟 Channel
2. 拿到 Channel Access Token 跟 Channel Secret
3. 設定 Webhook URL，讓 LINE 的訊息可以打到你的 n8n

前面開帳號設定的部分不難，LINE 的開發者後台做得還算清楚。比較要注意的是 Webhook 的部分——n8n 要有一個外部可以存取的 URL 讓 LINE 打進來。如果 n8n 跑在本地的話，得用 ngrok 之類的工具做 tunnel，或者直接把 n8n 部署到有固定 IP 的地方。

## 在 n8n 裡怎麼接

n8n 本身有 LINE 的節點，但我最後是用比較通用的做法：

- **接收訊息**：用 Webhook 節點接 LINE 打過來的 event
- **發送訊息**：用 HTTP Request 節點打 LINE 的 Push Message API

為什麼不直接用 LINE 節點？因為我想要比較細的控制權。LINE 的訊息格式支援蠻多種的——純文字、Flex Message（可以做卡片式排版）、圖文選單等等。用 HTTP Request 的話我可以自己組 JSON，想怎麼排就怎麼排。

## 把英文教學跟技術快報都接上去

這部分反而是最快的。前兩週的 flow 架構都不用動，只要在最後面把「推到 Slack」的節點，多拉一條線出來接「推到 LINE」就好。

所以現在的流程變成：

- **每日英文**：n8n 排程 → 讀 CSV → Ollama 生成 → 同時推 Slack + LINE
- **技術快報**：n8n 排程 → 抓 RSS → Ollama 摘要 → 同時推 Slack + LINE

Slack 那邊我沒有拿掉，因為部門裡有人在看。LINE 是推給我自己的，兩邊並行。

## 在 LINE 上的呈現

Slack 的排版本來就比較適合長文，Markdown 支援也好。LINE 就不一樣了，純文字塞太多會變成一大坨，很難看。

所以推到 LINE 的時候我做了一些調整：

- 英文教學的部分，精簡成**關鍵詞 + 情境對話 + 每日挑戰**三個區塊，深度解析那段先拿掉（有興趣再去 Slack 看完整版）
- 技術快報的部分，只推**總結 + 前五篇**的標題跟摘要，不把十幾篇全部塞進來

訊息長度控制在滑個兩三下就能看完的範圍。這不是偷懶，是因為使用情境不同——我在 LINE 上看東西的時候通常是通勤或零碎時間，沒有要坐下來慢慢讀。

## 三週下來

從第一週的英文 bot、第二週的快報 bot、到這週串上 LINE，三個禮拜做了三件事，但本質上是同一套東西在長大。

n8n 當作中樞，Ollama 負責動腦，Slack 跟 LINE 負責推送。要再加新的 bot 或新的推送管道，都是在這個骨架上面接就好。

每天早上 LINE 推英文、下午推快報。我什麼都不用做，打開 LINE 就有東西在等我。

之前工作忙到喘不過氣的時候，覺得連自己的東西都沒空弄。現在回頭看，其實不是沒時間，是沒有找到那個**「忙碌中的喘息空間」**。做這些小工具的過程本身就是一種調劑，做完之後又能讓日常生活方便一點。

算是壓力大的這段時間裡，少數讓我覺得開心的事。
