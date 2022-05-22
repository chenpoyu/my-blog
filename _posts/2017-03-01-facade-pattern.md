---
layout: post 
title: "外觀模式：簡化電商多系統對接的複雜性" 
date: 2017-03-01 10:00:00 +0800 
categories: [設計模式, Java] 
tags: [外觀模式, 系統集成]
---

一個簡單的「下單按鈕」，背後隱藏了極其複雜的互動：

```java
// 原始的混亂呼叫
inventoryService.checkStock(productId);
paymentGateway.initiate(amount, "LINE_PAY");
shippingService.calculateRate("TAIPEI", "SF_EXPRESS");
orderRepository.save(order);
smsService.send(userPhone, "訂單已建立");

```

這違反了 **最少知識原則 (Least Knowledge Principle / Law of Demeter)**。前端或其他的微服務不應該知道這麼多子系統的細節。只要其中一個子系統的介面改了，所有呼叫端都得跟著改。

## 外觀模式 (Facade Pattern) 的解法

外觀模式的精神在於：**「為子系統中的一組介面提供一個一致的界面，此模式定義了一個高層介面，這個介面使得這一子系統更加容易使用。」**

### 1. 建立外觀類別：OrderFacade

我們建立一個統一的門面，將複雜的互動封裝在裡面。

```java
public class OrderFacade {
    private InventoryService inventoryService = new InventoryService();
    private PaymentService paymentService = new PaymentService();
    private ShippingService shippingService = new ShippingService();
    private NotificationService notificationService = new NotificationService();

    /**
     * 一鍵下單的門面方法
     */
    public void placeOrder(String productId, int qty, String paymentType, String shippingType) {
        System.out.println("--- 開始執行門面封裝的下單流程 ---");

        // 1. 檢查庫存
        if (!inventoryService.isAvailable(productId, qty)) {
            throw new RuntimeException("庫存不足");
        }

        // 2. 處理支付
        paymentService.pay(paymentType);

        // 3. 安排物流
        shippingService.arrange(shippingType);

        // 4. 發送通知
        notificationService.notifyCustomer("您已下單成功！");

        System.out.println("--- 下單流程順利完成 ---");
    }
}

```

### 2. 客戶端的使用

現在，開發者不需要知道物流怎麼安排，也不需要知道簡訊怎麼發送。

```java
public class CheckoutController {
    private OrderFacade orderFacade = new OrderFacade();

    public void onCheckoutButtonClick() {
        // 只需呼叫一個簡單的方法
        orderFacade.placeOrder("IPHONE_14", 1, "LINE_PAY", "SF_EXPRESS");
    }
}

```

## 外觀模式 vs. 代理模式

這次的 Code Review 中，有新人問：「這跟代理模式 (Proxy) 有什麼差別？」

* **代理模式 (Proxy)**：通常是一對一，它是為了**控制對物件的存取**（例如：權限檢查、延遲載入）。
* **外觀模式 (Facade)**：通常是一對多，它是為了**簡化介面**，將多個複雜的動作包裝成一個簡單的操作。

## 外觀模式在電商系統中的優勢

1. **降低耦合度**：子系統內部的修改不會影響到客戶端，只要 Facade 的介面保持不變即可。
2. **安全性提升**：Facade 可以只暴露子系統中「安全」且「必要」的方法，隱藏敏感的內部邏輯。
3. **提升開發效率**：新進同事只需要看 `OrderFacade` 的方法定義，就能快速上手業務流程，不需要去研究五個子系統的原始碼。
