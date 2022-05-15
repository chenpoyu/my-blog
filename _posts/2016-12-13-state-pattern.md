---
layout: post 
title: "狀態模式：優化電商訂單狀態流轉與會員權限管理" 
date: 2016-12-13 09:30:00 +0800 
categories: [設計模式, Java] 
tags: [狀態模式, 電商系統]
---

最近面臨一個大問題：一個訂單從「待付款」到「已發貨」，中間可能經歷「取消」、「退款申請」、「部分退款」、「逾期關閉」等數十種狀態。

最可怕的是，不同的狀態下，使用者能做的操作（行為）完全不同。

## 原始問題：if-else 的惡夢

最初的 `Order` 類別裡一樣有充斥著這樣的程式碼：

```java
public void cancelOrder() {
    if (state == PAID) {
        // 走退款流程
    } else if (state == WAITING_PAY) {
        // 直接關閉
    } else if (state == SHIPPED) {
        // 報錯：已出貨不能取消
    }
    // ... 更多的 else if
}

```

每增加一個狀態，或是修改一個狀態的行為，都要在幾十個方法裡搜尋並修改這些 `if-else`。這不僅難以維護，還極其容易產生漏改的情況。

## 狀態模式 (State Pattern) 的解法

狀態模式的精神在於：**「允許一個物件在其內部狀態改變時改變它的行為。」** 我們不再讓 `Order` 物件去判斷狀態，而是將每個狀態封裝成一個獨立的類別。

### 1. 定義狀態介面

定義所有狀態共有的行為介面。

```java
public interface OrderState {
    void pay(OrderContext context);
    void cancel(OrderContext context);
    void ship(OrderContext context);
    String getStatusName();
}

```

### 2. 實作具體狀態類別

每個狀態類別只關心自己在該狀態下「該做什麼」。

#### 待付款狀態 (WaitPayState)

```java
public class WaitPayState implements OrderState {
    @Override
    public void pay(OrderContext context) {
        System.out.println("支付成功！");
        context.setState(new PaidState()); // 轉移到下一個狀態
    }

    @Override
    public void cancel(OrderContext context) {
        System.out.println("取消訂單成功。");
        context.setState(new ClosedState());
    }

    @Override
    public void ship(OrderContext context) {
        System.out.println("錯誤：未付款不能發貨。");
    }

    @Override
    public String getStatusName() { return "待付款"; }
}

```

#### 已付款狀態 (PaidState)

```java
public class PaidState implements OrderState {
    @Override
    public void pay(OrderContext context) {
        System.out.println("錯誤：請勿重複支付。");
    }

    @Override
    public void cancel(OrderContext context) {
        System.out.println("發起退款申請...");
        context.setState(new RefundingState());
    }

    @Override
    public void ship(OrderContext context) {
        System.out.println("發貨成功！");
        context.setState(new ShippedState());
    }

    @Override
    public String getStatusName() { return "已付款"; }
}

```

### 3. 環境類別 (Context)

`OrderContext` 負責維持目前的狀態，並將請求委託（Delegate）給目前的狀態物件。

```java
public class OrderContext {
    private OrderState currentState;

    public OrderContext() {
        this.currentState = new WaitPayState(); // 初始狀態
    }

    public void setState(OrderState state) {
        this.currentState = state;
    }

    public void pay() { currentState.pay(this); }
    public void cancel() { currentState.cancel(this); }
    public void ship() { currentState.ship(this); }
}

```

## 狀態模式 vs 策略模式

* **策略模式 (Strategy)**：通常由「外部」決定使用哪種策略，且各個策略之間通常是**獨立、無關**的（例如：不同折扣算法）。
* **狀態模式 (State)**：狀態之間通常存在**轉換關係**，且這種轉換是由狀態物件內部根據業務邏輯「自動觸發」的。

