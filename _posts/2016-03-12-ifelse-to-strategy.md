---
layout: post 
title: "從 if-else 到策略模式：重構複雜的折扣計算法" 
date: 2016-03-12 10:30:00 +0800 
categories: [設計模式, Java] 
tags: [策略模式, 重構]
---

在開發電商系統時，最常見的邏輯莫過於「折扣計算」。隨著業務成長，各種促銷活動接踵而至：週年慶、會員折扣、限時搶購...。如果我們不夠謹慎，程式碼很快就會被無窮無盡的 `if-else` 給淹沒。

## 遇到的問題

最初的程式碼可能長得像這樣：

```java
public double calculatePrice(double price, String type) {
    if (type.equals("VIP")) {
        return price * 0.8;
    } else if (type.equals("CHILDREN")) {
        return price * 0.5;
    } else if (type.equals("NEW_YEAR")) {
        return price - 100;
    }
    return price;
}

```

這樣的寫法違反了 **SOLID** 中的 **開閉原則 (Open-Closed Principle)**：每當增加一種折扣，我們就必須修改這個方法，導致系統變得難以維護且容易出錯。

## 使用策略模式 (Strategy Pattern)

為了將「演算法」與「使用環境」分離，我們可以定義一個策略介面。

### 1. 定義策略介面

```java
public interface DiscountStrategy {
    double applyDiscount(double price);
}

```

### 2. 實作具體的策略

```java
public class VipDiscount implements DiscountStrategy {
    @Override
    public double applyDiscount(double price) {
        return price * 0.8;
    }
}

public class NewYearDiscount implements DiscountStrategy {
    @Override
    public double applyDiscount(double price) {
        return price - 100;
    }
}

```

### 3. 環境類別 (Context)

```java
public class OrderProcessor {
    private DiscountStrategy strategy;

    public void setDiscountStrategy(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public double getFinalPrice(double price) {
        return strategy.applyDiscount(price);
    }
}

```

## 重構後的優點

1. **易於擴展**：新增折扣只需要建立新的 Class，完全不需更動原有的邏輯。
2. **單一職責**：每種折扣邏輯都被封裝在獨立的類別中。
3. **可測試性**：我們可以針對單一策略進行單元測試。

![策略模式結構圖]({{ '/assets/images/strategy-pattern-diagram.jpg' | relative_url }})

## 結語

從 `if-else` 到策略模式的重構，本質上是將「變動的部分」與「穩定的部分」切開。
