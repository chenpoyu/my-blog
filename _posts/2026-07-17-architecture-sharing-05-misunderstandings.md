---
layout: post
title: "分散式履約架構（五）：Read Model 和幾個常被誤解的地方"
date: 2026-07-17 16:35:00 +0800
categories: [技術筆記, 系統設計]
tags: [系統架構, 分散式系統, Read Model, CQRS]
---

## 系列最後一篇，補兩件事

前四篇走完了這個架構的主要流程：
從請求進來的三關，到 Obligation 拆解路由，到失敗補償。

這篇想補兩個前面沒特別說清楚的地方。

第一個是 **Read Model** 的定位。
第二個是幾個**常見誤解**，這些誤解在設計討論裡很容易出現，但不說清楚很難察覺。

## Read Model：查詢用的，不是事實

這個架構裡，資料的事實來源在各個 Owner Service 手上。

Contract Service 的合約資料，住在 Contract Service 的資料庫。
TV Service 的訂閱資料，住在 TV Service 的資料庫。

但有一個現實問題：

後台 Admin 或客服介面，通常需要一次看到同一個客戶的合約、訂閱、帳務、設備狀態。

如果每次查詢都要同時打五個 service，組合之後再回傳，
在大量查詢的情況下，這件事的代價不小。

所以這個架構裡會有一個 **Customer 360 Read Model**，
把各個 domain 的資料投影成一個查詢用的資料結構。

它的資料來源是 Outbox 事件。
各個 Owner Service 和 Orchestrator 產生事件之後，
Read Model 訂閱這些事件，把必要的欄位同步進來，提供給後台查詢。

這裡有兩件事要特別記住。

第一，Read Model 不是事實來源。

它的資料有時間差，高風險的操作，例如退費、強制停用，
執行之前一定要回頭查 Owner Service，不能只看 Read Model。

第二，Read Model 只存查詢需要的欄位。

不是把整個 domain 的資料都複製進來，
只存 Admin 和客服畫面需要的摘要：訂單狀態、合約摘要、最近一次同步時間、資料來源 service。

Read Model 的定位是「方便查」，不是「可以寫」。

## 幾個常被誤解的地方

說完 Read Model，來說幾個在討論這個架構時，很容易出現的誤解。

**Obligation 不是任務名稱**

很多人第一次看到 Obligation，直覺會把它想成「任務清單裡的一筆」。

但 Obligation 有狀態機、有 owner domain、有路由資訊、有稽核紀錄、有補償定義。
它是一個有完整生命週期的責任執行單元，不是一個字串標記。

這個差別很重要，因為它決定了你在設計時，要為每個 Obligation 準備多少附帶資訊。

**Routing Metadata 不是 service routing**

Routing Metadata 決定的是「這個 Obligation 要用哪個 datasource key」。

要呼叫哪個 service，Obligation 的 ownerDomain 已經決定了，
Routing Metadata 不管這件事。

這兩個東西如果混在一起設計，
之後資料庫切換的邏輯和 service 路由邏輯會互相干擾，很難拆開。

**補償不是 rollback**

這個前一篇提過，但值得再說一次，因為真的很容易混淆。

資料庫 rollback 是在 transaction 尚未提交時取消操作。

補償是在動作已經成功執行之後，
因為業務流程整體失敗，對已完成的動作做反向處理。

補償有自己的邏輯、有自己的失敗處理、有可能需要人工介入，
跟資料庫層的機制沒有直接關係。

**Customer 360 不是事實來源**

這個剛說過，但它是這個架構最容易被踩到的地方之一。

Read Model 方便查，所以很多人會直接信任它。
但它和 Owner Service 之間有時間差，
在資料一致性要求高的情境下，一定要查 Owner Service。

**Owner Service 不能偷偷去做別人的事**

Owner Service 執行自己的 Obligation，沒問題。
Owner Service 去查詢其他 service 的資料（read），沒問題。

但 Owner Service 去呼叫其他 domain 做寫入，就不行。

跨域寫入一旦散落在 Owner Service 裡，
Orchestrator 就失去了追蹤和補償的能力，
整個分散式流程的可觀測性會迅速崩掉。

## 這個架構適合什麼情境

最後說一下適用範圍，因為這個系列一直在聊架構，
但不同情境下，適不適合用是很不一樣的。

這個架構適合的情境：

- 一個使用者請求會觸發多個跨 domain 的寫入動作
- 每個 domain 各自管理自己的資料庫，不能共享 transaction
- 有多公司或多資料來源的路由需求
- 需要完整的稽核紀錄和事後追蹤能力
- 失敗需要有明確的重試、補償、或人工介入機制

這個架構**不適合**的情境：

- 單純的 CRUD，domain 之間沒有耦合
- 資料都在同一個資料庫，transaction 夠用
- 團隊規模小，還沒有多 service 的需求

引入這個架構是有成本的。
要管更多 state、更多事件契約、更多 service 邊界的協議。

但如果你的系統已經在跨域寫入、已經在用記憶和口頭約定管理狀態、
已經在出事的時候找不到根因，
那這個架構帶來的清晰度，會比它的複雜度更值得。

## 系列收尾

這五篇把這個架構從頭到尾走了一遍。

它的核心思路很直接：
不用一個大 transaction 把所有事情包住，
而是把一個複雜的業務請求，
拆成一組有狀態、有責任、可追蹤、能補償的執行單元。

然後讓每個 domain 只做自己的事，
讓 Orchestrator 串起全場，
讓失敗有地方記錄，讓決策有辦法還原。

這不是最輕的做法，
但在多 domain、多公司、需要稽核的系統裡，
它是一條走得比較穩的路。
