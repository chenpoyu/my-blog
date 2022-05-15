---
layout: post 
title: "組合優於繼承：重構電商平台商品系統" 
date: 2016-04-21 14:20:00 +0800 
categories: [設計模式, Java] 
tags: [組合優於繼承, 裝飾者模式]
---

在剛學習物件導向時，「繼承」往往是最吸引人、也最讓人頭疼的事情。我們習慣用 `is-a` 的邏輯去建模。但在複雜的電商系統中，過度依賴繼承往往會導致「類別爆炸」。

## 繼承的陷阱

假設我們要設計一個商品系統，商品有基本款，有些可以「加購禮盒包裝」，有些需要「保固服務」。

如果使用繼承：

* `Product`
* `ProductWithGiftWrap`
* `ProductWithWarranty`
* `ProductWithGiftWrapAndWarranty` (為了組合功能，必須再開一個子類)



當功能變多（例如：保險、急件快遞），子類別的數量會呈指數成長，這就是典型的**類別爆炸**。

## 核心概念：組合 (Composition)

與其說「帶有包裝的商品**是一個**商品」，不如說「商品**擁有**一個包裝功能」。透過將行為封裝在獨立的物件中，並在執行期動態組合，能獲得更大的彈性。

### 1. 定義組件介面

```java
public interface Item {
    double getCost();
    String getDescription();
}

```

### 2. 基礎實作

```java
public class BasicProduct implements Item {
    @Override
    public double getCost() {
        return 500.0;
    }

    @Override
    public String getDescription() {
        return "標準商品";
    }
}

```

### 3. 使用裝飾者模式 (Decorator Pattern) 實踐組合

```java
public abstract class ProductDecorator implements Item {
    protected Item tempItem;

    public ProductDecorator(Item newItem) {
        tempItem = newItem;
    }
}

public class GiftWrap extends ProductDecorator {
    public GiftWrap(Item newItem) {
        super(newItem);
    }

    @Override
    public double getCost() {
        return tempItem.getCost() + 50.0;
    }

    @Override
    public String getDescription() {
        return tempItem.getDescription() + " + 禮盒包裝";
    }
}

```

## 為什麼組合更好？

1. **動態性**：你可以在執行期決定要幫商品加上什麼功能，而不必在編譯期決定。
2. **低耦合**：`GiftWrap` 不需要知道它包裝的是 `BasicProduct` 還是另一個 `WarrantyProduct`。
3. **避免冗餘**：不需要為了每一種可能的排列組合去寫一個新類別。

### 實際呼叫方式

```java
Item myOrder = new GiftWrap(new BasicProduct());
System.out.println("描述: " + myOrder.getDescription());
System.out.println("總價: " + myOrder.getCost());

```

## 結語

繼承建立了強耦合的層級關係，而組合則像積木一樣靈活。當發現類別階層開始變得難以維護時，停下來思考一下：這真的必須是繼承嗎？或許組合才是更好的解法。
