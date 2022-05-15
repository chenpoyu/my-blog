---
layout: post 
title: "觀察者模式的應用：從強耦合到事件驅動" 
date: 2016-06-14 11:10:00 +0800 
categories: [設計模式, Java] 
tags: [觀察者模式, 事件驅動, 解耦合]
---

在開發電子商務的大型系統時，我們常遇到一個情境：當一個核心動作發生時（例如：使用者完成付款），後續有一連串的附帶動作需要執行。

最直覺但糟糕的寫法，就是把這些邏輯全部塞在一起，執行速度慢、東西又耦合。

## 原始需求：付款後的連鎖反應

假設我們有一個 `OrderService`，在付款成功後需要：

1. 更新訂單狀態。
2. 發送推播通知給使用者。
3. 通知倉庫準備出貨。
4. 累計會員積分。

### 糟糕的實作 (強耦合)

```java
public class OrderService {
    private NotificationService notificationService = new NotificationService();
    private WarehouseService warehouseService = new WarehouseService();
    private RewardService rewardService = new RewardService();

    public void completePayment(Order order) {
        order.setStatus(Status.PAID);
        // 開始一連串的硬編碼呼叫
        notificationService.sendPush(order.getUser());
        warehouseService.prepareShipping(order);
        rewardService.addPoints(order.getUser(), order.getAmount());
        
        System.out.println("訂單處理完成");
    }
}

```

**這段代碼的問題在於：**

* **違背 SRP**：`OrderService` 關心了太多它不該關心的事（發送通知、積分計算）。
* **難以維護**：如果明天要增加「發送行銷簡訊」，就得再改一次 `OrderService`。
* **同步阻塞**：如果 `WarehouseService` 響應很慢，整個付款流程就會卡住。

## 觀察者模式 (Observer Pattern) 的介入

觀察者模式定義了一對多的依賴關係，讓一個物件（Subject）的狀態改變時，所有依賴它的物件（Observers）都能自動收到通知。

### 1. 定義觀察者介面

```java
public interface OrderObserver {
    void onPaymentCompleted(Order order);
}

```

### 2. 實作具體的觀察者

```java
public class NotificationObserver implements OrderObserver {
    @Override
    public void onPaymentCompleted(Order order) {
        System.out.println("發送推播給：" + order.getUser().getName());
    }
}

public class RewardObserver implements OrderObserver {
    @Override
    public void onPaymentCompleted(Order order) {
        System.out.println("為使用者累計積分");
    }
}

```

### 3. 被觀察的主體 (Subject)

```java
import java.util.ArrayList;
import java.util.List;

public class PaymentSubject {
    private List<OrderObserver> observers = new ArrayList<>();

    public void attach(OrderObserver observer) {
        observers.add(observer);
    }

    public void notifyObservers(Order order) {
        for (OrderObserver observer : observers) {
            observer.onPaymentCompleted(order);
        }
    }
}

```

## 進階應用：使用 EventBus

除了觀察者模式，其實還有一種利用框架來實現**事件驅動架構 (EDA)**。

以下是以 Google Guava 的 `EventBus` 為例的現代化重構：

### 核心事件定義

```java
public class OrderPaidEvent {
    private final Order order;
    public OrderPaidEvent(Order order) { this.order = order; }
    public Order getOrder() { return order; }
}

```

### 事件訂閱者 (Subscriber)

```java
public class InventoryListener {
    @Subscribe
    public void handleOrderPaid(OrderPaidEvent event) {
        System.out.println("庫存系統收到通知，保留商品 ID: " + event.getOrder().getId());
    }
}

```

### 最終的 OrderService

```java
public class OrderService {
    private final EventBus eventBus;

    public OrderService(EventBus eventBus) {
        this.eventBus = eventBus;
    }

    public void completePayment(Order order) {
        order.setStatus(Status.PAID);
        // 只需發布事件，不需關心誰在聽
        eventBus.post(new OrderPaidEvent(order));
    }
}

```

## 實戰心得

透過這種方式，`OrderService` 變回了輕量級的組件，它只專注於處理訂單狀態。其餘的附帶邏輯（通知、積分、物流）全部解耦到了獨立的 Listener 中。

**優點總結：**

1. **擴展性極強**：如果又有新的後續動作，就寫一個新的 Listener 並註冊到 EventBus 即可，完全不需要動到 `OrderService`。
2. **併發處理**：可以輕鬆地將 EventBus 配置為非同步，讓發送通知等耗時操作在背景執行，大幅提升 API 回應速度。
3. **代碼清晰**：業務邏輯邊界清晰，每個類別只專注於一件事情。
