---
layout: post
title: "分散式履約架構（四）：失敗之後，誰來收拾"
date: 2026-07-10 13:55:00 +0800
categories: [技術筆記, 系統設計]
tags: [系統架構, 分散式系統, Saga, 補償機制, Outbox]
---

## Owner Service 的工作只有一件事

Order 拆解成 Obligation、路由決定好資料來源之後，
Orchestrator 把執行命令分發給各個 Owner Service。

Owner Service 做什麼？

很單純：執行自己的那一個 Obligation，寫自己的資料庫，回傳結果。

僅此而已。

每個 Owner Service 只負責自己的 domain 資料，
也只做自己的本地 transaction。

## Owner Service 最容易踩的地方

有一件事在這個架構裡要特別留意。

Owner Service 在執行自己的 Obligation 時，
**不能去呼叫其他 domain 的寫入操作**。

舉例：
Contract Service 可以去查 address service 取得標準化地址，這沒問題。
但 Contract Service 不能去呼叫 TV Service 建訂閱、不能去叫 Billing Service 建帳戶。

如果 Owner Service 開始跨域寫入，
它就變成了一個藏在底層的 Orchestrator。

這會讓整個流程的追蹤和補償變得極難管理，
因為你根本不知道某個地方寫入了什麼，什麼時候寫入的，失敗了該怎麼回。

跨域寫入的動作，應該全部透過 Orchestrator 和 Obligation 來完成。
這是這個架構最重要的邊界之一。

## 失敗的時候怎麼辦

每個 Obligation 執行完之後，會回傳結果給 Orchestrator。

成功就是 DONE。
失敗會帶上幾個欄位：

- 失敗原因
- 這個失敗可不可以重試
- 這個 Obligation 可不可以補償
- 需不需要進人工佇列

Orchestrator 根據這些資訊決定下一步。

如果是可重試的失敗，就重試。
如果失敗的 Obligation 是選填的，Order 可能繼續完成，狀態標為有警告。
如果失敗的 Obligation 是必要的，而且之前已經有 Obligation 執行成功，就要觸發補償。

## 補償不是 rollback

這件事很多人一開始會搞混。

資料庫的 rollback，是在 transaction 還沒提交之前取消操作。

補償是完全不同的概念。

補償是「已經成功的動作，在業務層面做反向處理」。

比如說：
- 網路合約已經建立成功
- 電視訂閱失敗，而且電視是必要的
- Order 無法完成
- 所以要補償網路合約：呼叫 INTERNET\_CONTRACT\_CANCEL

補償是一個獨立的業務動作，有自己的邏輯、有自己的錯誤處理，
跟資料庫層的事務回滾沒有關係。

## Outbox 和 Audit Log

流程跑完之後，有兩件事要記錄。

**Outbox** 負責事件的可靠發布。

Orchestrator 把事件寫進 Outbox table，
再由獨立的 publisher 把事件推送到 Event Bus。
這樣即使 Event Bus 短暫不可用，事件也不會遺失。

事件的類型包括：
order.created、obligation.created、obligation.completed、obligation.failed、obligation.compensated，
這些都是有版本、有 schema 定義的契約，不是隨便 log 一筆。

**Audit Log** 負責決策紀錄。

每一筆 Audit Log 會記錄這次流程用了哪些版本：
Catalog version、Eligibility version、Policy version、Routing version，
以及 datasource key、錯誤碼、Trace ID。

這是為了讓事後可以還原當時的判斷脈絡，
不只是知道「失敗了」，而是知道「用什麼條件判斷的、在哪個版本下失敗的」。

## 錯誤碼不要做成全局大表

最後說一下錯誤碼。

這個架構的 service 數量不少，
如果所有 service 的錯誤碼全部塞進同一張表，
很快就會失控，而且各個 domain 搶名稱互相干擾。

比較好的做法是 namespace 分開：

```
CATALOG.BUNDLE_NOT_FOUND
ELIGIBILITY.ADDRESS_NOT_SERVICEABLE
PERMISSION.ACTION_DENIED
ASSET.OUT_OF_STOCK
CONTRACT.CREATE_FAILED
```

每個 domain 管自己的錯誤碼，namespace 全局唯一就夠了。

錯誤回傳的 envelope 保持一致：
包含 traceId、source、category、code、retryable、compensatable，
讓 Orchestrator 能用統一的方式解讀結果。

## 這個系列的收尾

這四篇大致把這個架構的主要部分都走過一遍了。

從「為什麼需要這個架構」、
「請求進來之前的三關」、
「Obligation 怎麼被拆解和路由」，
到「Owner Service 的邊界和失敗後的補償」。

這個架構的本質，
是把一個複雜的跨域業務請求，
拆成一組有狀態、有責任、可追蹤、可補償的執行單元。

它不試圖用一個大 transaction 把所有事情收起來，
而是把複雜度顯性化，讓每一個環節都有地方記錄、有辦法追、出問題有辦法回。

這不是最簡單的做法，
但在跨域、多公司、有稽核需求的系統裡，
它通常是走得最穩的路。
