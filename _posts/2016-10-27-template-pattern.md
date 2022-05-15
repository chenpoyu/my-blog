---
layout: post 
title: "範本方法模式：標準化電商訂單處理流程" 
date: 2016-10-27 10:15:00 +0800 
categories: [設計模式, Java] 
tags: [模板方法, 電商系統, 重構]
---

明年初，我們的電商平台開始引入「跨境電商」與「數位虛擬商品」業務。原有的訂單處理邏輯程式碼中，充滿了大量的重複結構。每種訂單都要經過：驗證庫存、計算稅額、扣款、通知。

雖然流程大同小異，但細節卻很折磨人：跨境訂單需要計算關稅，數位商品則完全不需要物流。

## 核心問題：程式碼邏輯的重複與分竄

最初我們為了快速上線，複製貼上（Copy-Paste）了大量的程式碼。這導致當想修改「付款成功後的通知邏輯」時，必須在三個不同的類別中同步修改，這明顯違反了 **DRY (Don't Repeat Yourself)** 原則。

## 範本方法模式 (Template Method Pattern) 的解法

這個模式的精神是：**「在一個方法中定義一個演算法的骨架，而將一些步驟延遲到子類別中實作。」** 它讓子類別可以在不改變演算法結構的情況下，重新定義該演算法的某些步驟。

### 1. 定義抽象骨架

我們建立一個 `OrderProcessor` 抽象類別，定義了訂單處理的固定流程（final 方法）。

```java
public abstract class OrderProcessor {

    // 範本方法，定義了不可變的演算法骨架
    public final void processOrder(long orderId) {
        verifyStock(orderId);
        calculateTax();
        calculateShipping();
        executePayment();
        if (isNotificationRequired()) { // Hook (鉤子方法)
            sendNotification();
        }
    }

    // 固定不變的步驟直接實作
    private void executePayment() {
        System.out.println("執行金流扣款程式...");
    }

    // 由子類別實作的具體差異步驟
    protected abstract void verifyStock(long orderId);
    protected abstract void calculateTax();
    protected abstract void calculateShipping();
    protected abstract void sendNotification();

    // Hook 方法：子類別可以選擇性覆寫，預設為 true
    protected boolean isNotificationRequired() {
        return true;
    }
}

```

### 2. 實作具體的訂單類型

#### 實體商品訂單 (Physical Product)

```java
public class PhysicalOrderProcessor extends OrderProcessor {
    @Override
    protected void verifyStock(long orderId) {
        System.out.println("檢查倉庫 WMS 庫存...");
    }

    @Override
    protected void calculateTax() {
        System.out.println("計算一般營業稅 5%...");
    }

    @Override
    protected void calculateShipping() {
        System.out.println("計算黑貓宅急便運費 100 元...");
    }

    @Override
    protected void sendNotification() {
        System.out.println("發送簡訊通知用戶已出貨...");
    }
}

```

#### 數位虛擬商品訂單 (Digital Product)

```java
public class DigitalOrderProcessor extends OrderProcessor {
    @Override
    protected void verifyStock(long orderId) {
        System.out.println("檢查數位授權碼數量...");
    }

    @Override
    protected void calculateTax() {
        System.out.println("虛擬商品免稅...");
    }

    @Override
    protected void calculateShipping() {
        System.out.println("數位發貨，運費為 0...");
    }

    @Override
    protected void sendNotification() {
        System.out.println("發送 Email 內含下載連結...");
    }

    // 數位商品不需要發送簡訊，我們覆寫 Hook
    @Override
    protected boolean isNotificationRequired() {
        return true; // 或是根據條件決定是否通知
    }
}

```

## 深度分析：Hook (鉤子) 的巧妙運用

在範本方法中，**Hook** 是非常強大的設計。在上面的例子中，`isNotificationRequired()` 就是一個鉤子。

預設情況下，演算法會執行通知，但如果某些特定類型的訂單（例如：系統測試訂單）不需要通知，子類別只需覆寫該方法並回傳 `false`。這讓框架具有極高的靈活性，而不需要在父類別寫一堆 `if-else`。

## 這與策略模式 (Strategy) 的區別

在討論Hook的時候，很常有人會問這和策略模式的差異是什麼？

* **策略模式 (Strategy)**：使用「組合」。你將整個演算法（策略）抽換掉。它更強調**物件的行為切換**。
* **範本方法 (Template Method)**：使用「繼承」。它定義了演算法的**結構**，只讓子類別去填補其中的小步驟。它更強調**流程的標準化**。

## 結語

這種範本方法模式在電商系統中其實非常常見，它有效地防止了「流程遺漏」。透過將關鍵步驟定義為 `abstract`，會強迫你在新增訂單類型時必須實作庫存檢查或稅務計算，從而減少了 Bug 的產生。
