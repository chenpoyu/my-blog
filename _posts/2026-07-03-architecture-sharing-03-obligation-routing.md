---
layout: post
title: "分散式履約架構（三）：Obligation 是怎麼被拆開的"
date: 2026-07-03 11:10:00 +0800
categories: [技術筆記, 系統設計]
tags: [系統架構, 分散式系統, Saga, 訂單系統]
---

## Order 建起來之後

前兩篇說了這個架構的整體輪廓，
以及 Catalog、Eligibility、Permission 三關通過之後建立 Order 的過程。

Order 建起來，代表系統已經確認：

這個請求是合法的、業務上可行的、操作上被允許的。

接下來要面對的問題是：

這一張 Order，要怎麼讓多個 domain 的 service 各自去執行屬於自己的部分？

## Obligation Factory

Order 建完之後，第一件事是交給 Obligation Factory。

它的工作就是把 Order 拆解成多個 Obligation。

拆解的依據來自 Catalog 的結果：
這個 bundle 包含幾個元件、各自屬於哪個 domain、是必要還是選填的。

以「網路 + 電視」套餐為例，
Obligation Factory 會產出三個 Obligation：

- INTERNET\_CONTRACT\_CREATE，由 contract domain 負責
- TV\_SUBSCRIPTION\_CREATE，由 tv domain 負責
- DEVICE\_RESERVE，由 asset domain 負責

Obligation Factory 只做拆解，不呼叫任何 service。
它的邊界是「把責任定義清楚」，不是「去執行它」。

## Obligation 的狀態機

每個 Obligation 從建立開始，就有自己的生命週期：

```
CREATED → ROUTED → PROCESSING → DONE
                              → FAILED
                                → COMPENSATING → COMPENSATED
                                → MANUAL_REVIEW
```

這個狀態機讓 Orchestrator 在任何時間點，
都能知道每個 Obligation 走到哪裡了。

## Routing Metadata

Obligation 定義好之後，
接下來要解決一個問題：

同一種 Obligation，在不同公司、不同服務區域、讀寫需求不同的情況下，
可能需要打到不同的資料庫。

這件事交給 Routing Metadata 來決定。

它接收 Obligation 的上下文（類型、公司範圍、服務區域、讀寫方向），
然後回傳一個 datasource key。

它只決定「應該用哪個資料來源的識別碼」，
不決定要呼叫哪個 service，也不包含業務判斷。

service 是誰，Obligation 的 ownerDomain 已經決定了。
Routing Metadata 只管資料來源的邊界要怎麼切。

這個分工很重要，也很容易被搞混。

## Datasource Resolver

有了 datasource key 之後，
Datasource Resolver 把它對應到實際的技術目標：

哪個 connection pool、哪個 schema、主庫還是讀副本。

它是基礎設施層的邏輯，
不知道業務是什麼、不知道 bundle 的組成，
也不管 permission 和 eligibility 的結果。

它只做一件事：key 進去，技術目標出來。

## Order Orchestrator 在串全場

上面說的這些步驟，全部都是 Order Orchestrator 在控制的。

它負責：

- 維護 Order 的狀態機
- 決定 Obligation 的執行順序
- 追蹤每個 Obligation 的結果
- 決定是否重試、是否補償、是否進人工佇列
- 產生 Outbox 事件和 Audit Log

Order 的狀態也有自己的一套：

```
ACCEPTED → PROCESSING → COMPLETED
                      → COMPLETED_WITH_WARNING
                      → FAILED
                      → COMPENSATING → COMPENSATED
                      → MANUAL_REVIEW
```

Orchestrator 彙整所有 Obligation 的結果，
才決定最終 Order 的狀態。

## Orchestrator 的邊界

有一件事要特別說。

Orchestrator 控制流程，但不擁有 domain 資料。

它知道 Order 走到哪、每個 Obligation 狀態是什麼，
但它不應該去碰 contract domain 的資料、tv domain 的訂閱資料。

這些資料，各自屬於對應的 Owner Service。

Orchestrator 的角色像是交通管制，
它指揮誰先走、誰等待、誰出了問題要怎麼處理，
但路上跑的車、車裡的貨，不是它的事。

---

下一篇會接著聊 Owner Service 的執行和邊界，
以及失敗的時候，補償是怎麼做的。
