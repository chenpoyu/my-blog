---
layout: post
title: "LINE Flex Message：被我忽略太久的好東西"
date: 2025-12-26 12:00:00 +0800
categories: [技術筆記, Side Project]
tags: [LINE Bot, Flex Message, LINE Messaging API, 自動化]
---

## 其實這東西我早就該寫了

上週把 n8n 機器人串上 LINE 之後，花了不少時間在調訊息的排版。過程中重新認真研究了一次 LINE 的 Flex Message，然後才發現——這東西我用了這麼久，居然一直沒有好好聊過它。

Flex Message 不是什麼新東西了，LINE 推出來也好幾年。但我之前的使用方式比較粗糙，就是照著文件範例改一改能用就好。這次因為要讓每日英文跟技術快報在 LINE 上看起來像樣一點，才真的花時間把它搞清楚。

## 先講一下背景：從 LINE Notify 到 Messaging API

有在用 LINE 做推播的人應該都經歷過這個轉換——LINE Notify 停用了。

以前 LINE Notify 真的超方便，申請一個 token 就能往群組裡推訊息，完全免費，門檻低到不行。很多人（包括我）拿來做監控通知、排程提醒、自動推播各種東西。

結果 LINE 把它收了。

收了之後就得改用 LINE Messaging API。要開 Bot Channel、要設 Webhook、要處理 token，整個流程比以前複雜不少。但也因為這樣，才接觸到 Flex Message 這個更強大的東西。

LINE Notify 時代只能推純文字跟圖片，排版非常有限。Messaging API 搭配 Flex Message，等於是讓你在 LINE 的對話框裡塞一張小網頁進去。

## Flex Message 是什麼

簡單講，Flex Message 就是 LINE 版的 HTML + CSS。

你用 JSON 定義版面結構，LINE 負責渲染。可以有標題、內文、圖片、按鈕、分隔線、多欄排版，甚至可以做到類似卡片的效果。

一個基本的 Flex Message 結構長這樣：

```json
{
  "type": "flex",
  "altText": "今日英文教學",
  "contents": {
    "type": "bubble",
    "header": { ... },
    "body": { ... },
    "footer": { ... }
  }
}
```

- **bubble**：單張卡片
- **carousel**：多張卡片可以左右滑動
- **header / body / footer**：卡片的上中下三個區塊
- 每個區塊裡面放 **box**，box 裡面放 **text、image、button、separator** 等元件

概念上就是 flexbox 排版。橫的排、直的排、設間距、設顏色、設字級，全部在 JSON 裡面定義。

## 我怎麼用在每日推播上

### 英文教學卡片

之前純文字推送的時候，關鍵詞、對話、挑戰題全部擠在一起，看起來就是一坨。

改成 Flex Message 之後：

- **Header**：深藍底色，白字標題「📘 今日主題：商務談判策略」
- **Body**：三個關鍵詞各自是一個 box，單字加粗放大、音標灰字、中文解釋正常字級、例句用斜體。中間用 separator 隔開
- **Footer**：一個按鈕「查看完整教學」，點了跳到 Slack 的完整版

這樣一張卡片在 LINE 上看起來乾乾淨淨的，資訊層次分明。通勤的時候掃一眼就能抓到重點。

### 技術快報卡片

快報用的是 carousel 格式——每篇文章一張卡片，可以左右滑。

每張卡片：
- 上面放文章的封面圖（RSS 裡抓的）
- 中間是標題 + 摘要
- 下面一個按鈕「閱讀原文」直接開連結

比起純文字把十幾篇文章的標題跟摘要全部列出來，carousel 的體驗好非常多。想看的就點進去，不想看的滑過去，不會被一整面文字淹沒。

## Flex Message Simulator

LINE 官方有一個 [Flex Message Simulator](https://developers.line.biz/flex-simulator/)，這東西一定要用。

它就是一個線上的視覺化編輯器，左邊寫 JSON、右邊即時預覽。你改一個數字，右邊馬上更新。

我調卡片版面的時候基本上都開著這個。因為 Flex Message 的 JSON 結構巢狀很深，光看 code 很難想像最後長什麼樣。有了 Simulator，邊寫邊看，效率差很多。

唯一的缺點是它不能直接存檔或匯出成模板，每次還是得自己把 JSON 複製出來。但瑕不掩瑜。

## 一些踩過的小坑

### JSON 結構的巢狀地獄

Flex Message 的 JSON 很容易寫到五六層巢狀。一個 bubble 裡面有 body，body 裡面有 box，box 裡面有 box，box 裡面有 text⋯⋯改到後來很容易迷路。

我的做法是在 n8n 裡用 Code 節點，先用 JavaScript 把資料結構組好，最後再塞進 Flex Message 的模板。不要試圖在一個巨大的 JSON 模板裡面用 expression 插值，會崩潰。

### 文字太長會被截斷

Flex Message 的 text 元件有 `maxLines` 屬性，超過的部分會用刪節號帶過。這個如果沒注意到，摘要常常會被截得很奇怪。

我後來在 Ollama 的 prompt 裡就限制了摘要長度，從源頭控制，而不是在 Flex Message 端裁切。

### 不同裝置的顯示差異

同一個 Flex Message 在 iPhone 跟 Android 上的渲染會有微妙的差異，主要是字體大小跟間距。不至於爆版，但如果你是那種像素級要求的人，會很煩。

我的態度是：差不多就好，不值得花時間在這上面。

## 每月 200 則，私人用途很夠

LINE Messaging API 的免費方案是每月 200 則推播訊息。聽起來不多，但算一下：

- 每日英文：一天一則，一個月 30 則
- 技術快報：一天一則，一個月 30 則
- 偶爾手動觸發一些通知：大概 10-20 則

加起來一個月大概用 70-80 則，連一半都不到。

除非你拿來做那種要推給上千人的商業用途，不然個人使用 200 則真的綽綽有餘。省下來的額度以後要接新的 bot 也還有空間。

而且比起以前 LINE Notify 只能推文字，現在有 Flex Message 可以做漂亮的卡片，升級之後的體驗其實更好。只是設定門檻高了一點，但搞定一次之後就不用再管了。

## 這一個月做下來

從 12 月初到現在，四週做了四件事：

1. 英文教學 bot（n8n + Ollama + Slack）
2. 技術快報 bot（n8n + Ollama + Slack + RSS）
3. 串接 LINE Bot
4. Flex Message 卡片排版

每一步都是在前一步的基礎上加東西，不是從零開始。這種漸進式的做法很適合下班後零碎時間搞的 side project——每次只做一件事，但每次做完都能立刻用。

現在每天打開 LINE，英文跟快報都整整齊齊排在那邊等我。比起一個月前什麼都沒有的狀態，方便太多了。

年底忙成狗的這段時間，能有這些小東西陪著，挺好的。
