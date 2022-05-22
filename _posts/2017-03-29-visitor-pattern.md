---
layout: post 
title: "訪問者模式：在不修改現有結構下擴展電商系統的功能" 
date: 2017-03-29 10:30:00 +0800 
categories: [設計模式, Java] 
tags: [訪問者模式, 擴展性]
---
我們系統中有一個 `PaymentChannel` 的類別階層，包含了 `CreditCardPayment`、`LinePayment`、`Atm`。現在客戶提出一個需求：希望能為每種支付方式產出特定的「日終對帳明細」。

按照一般的思維，我們會在 `PaymentChannel` 介面增加一個 `generateReport()` 方法。但這會導致：

1. 我們必須修改所有已穩定的支付實作類別。
2. 支付類別原本只該負責「付錢」，現在卻得負責「報表」，違反了 **SRP (單一職責原則)**。

## 訪問者模式 (Visitor Pattern) 的解法

訪問者模式的精神是：**「將資料結構與作用於結構上的操作分離。」** 它允許你在不改變元素類別的前提下，定義作用於這些元素的新操作。

### 1. 定義元素介面 (Element)

關鍵在於 `accept` 方法，這是「雙分派 (Double Dispatch)」的精髓。

```java
public interface PaymentChannel {
    void accept(PaymentVisitor visitor); // 接受訪問者
}

```

### 2. 實作具體的支付管道

```java
public class CreditCardPayment implements PaymentChannel {
    public String getCardType() { return "VISA"; }
    
    @Override
    public void accept(PaymentVisitor visitor) {
        visitor.visit(this); // 將自己傳給訪問者
    }
}

public class LinePayPayment implements PaymentChannel {
    public String getLineId() { return "LINE_USER_001"; }

    @Override
    public void accept(PaymentVisitor visitor) {
        visitor.visit(this);
    }
}

```

### 3. 定義訪問者介面 (Visitor)

訪問者知道如何處理每一種具體的元素。

```java
public interface PaymentVisitor {
    void visit(CreditCardPayment creditCard);
    void visit(LinePayPayment linePay);
}

```

### 4. 實作具體的操作：財務報表產出

```java
public class FinanceReportVisitor implements PaymentVisitor {
    @Override
    public void visit(CreditCardPayment creditCard) {
        System.out.println("產出信用卡報表：卡片類型 " + creditCard.getCardType());
    }

    @Override
    public void visit(LinePayPayment linePay) {
        System.out.println("產出 LinePay 報表：使用者 " + linePay.getLineId());
    }
}

```

## 雙分派 (Double Dispatch) 的魔力

在客戶端，我們只需要將「訪問者」丟進「元素」裡：

```java
List<PaymentChannel> channels = Arrays.asList(new CreditCardPayment(), new LinePayPayment());
PaymentVisitor reportVisitor = new FinanceReportVisitor();

for (PaymentChannel channel : channels) {
    channel.accept(reportVisitor); 
}

```

**為什麼要這麼麻煩？**
當呼叫 `channel.accept(visitor)` 時，發生了兩次動態分派：

1. 根據 `channel` 的實際類型，決定呼叫哪個 `accept`。
2. 在 `accept` 內部，根據 `visitor` 的實際類型，決定呼叫哪個 `visit`。

這讓我們可以在不修改 `PaymentChannel` 的情況下，隨時增加 `MarketingVisitor`（計算行銷點數）或 `AuditVisitor`（稽核異常交易）。

## 什麼時候該用訪問者模式？

我總結了兩點：

* **結構穩定**：如果你的商品類型、支付類型已經固定，很少變動。
* **操作易變**：如果你經常需要對這些固定結構增加新的演算法或功能。

如果你的結構（類別）本身就天天在變，那訪問者模式會變成災難，因為你每加一個類別，就得去改所有的 `Visitor` 介面。
