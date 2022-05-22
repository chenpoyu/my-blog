---
layout: post 
title: "原型模式：解決電商促銷活動的複雜物件複製問題" 
date: 2017-04-13 11:20:00 +0800 
categories: [設計模式, Java] 
tags: [原型模式, 深層複製]
---

上週我收到一個需求：想要建立一個「跟昨天類似，但只改日期」的活動。此時，最直覺的做法是從資料庫讀取舊活動，然後手動將屬性一個個 Set 到新物件上。

但問題來了：`PromotionActivity` 內部包含了一個 `List<Product>`。如果你只是簡單地複製引用，修改新活動的商品清單時，舊活動也會被影響。

## 原型模式 (Prototype Pattern) 的解法

原型模式的精神是：**「用原型實例指定建立物件的種類，並且透過拷貝這些原型建立新的物件。」** 在 Java 中，這通常涉及到 `Cloneable` 介面，但背後的「深淺拷貝」才是設計的核心。

### 1. 定義可複製的原型

```java
import java.util.ArrayList;
import java.util.List;

public class PromotionActivity implements Cloneable {
    private String name;
    private List<String> applicableProducts;
    private DiscountPolicy policy;

    public PromotionActivity(String name) {
        this.name = name;
        this.applicableProducts = new ArrayList<>();
    }

    // 省略 Getter/Setter...

    @Override
    public PromotionActivity clone() {
        try {
            // 預設是淺拷貝 (Shallow Copy)
            PromotionActivity cloned = (PromotionActivity) super.clone();
            
            // 實作深拷貝 (Deep Copy) - 關鍵點！
            // 必須手動複製內部的集合，否則新舊活動會共享同一個 List
            cloned.applicableProducts = new ArrayList<>(this.applicableProducts);
            
            // 如果 policy 也是可變物件，也要進行深拷貝
            // cloned.policy = this.policy.clone();
            
            return cloned;
        } catch (CloneNotSupportedException e) {
            return null;
        }
    }
}

```

## 淺拷貝 vs. 深拷貝

這在這次的調整中，我花了很多的時間思考和驗證：

* **淺拷貝 (Shallow Copy)**：只複製基本型別（int, double）以及物件的「引用」。這意味著新舊物件指向堆疊中的同一個實例。
* **深拷貝 (Deep Copy)**：不僅複製基本型別，連物件內部的引用物件也會遞迴地建立新的複本。

### Java 的另一種選擇：序列化 (Serialization)

如果物件結構非常深，手寫 `clone()` 裡的深拷貝邏輯會變得像地獄。當時我們常用的一個技巧是透過序列化：

```java
// 使用 Apache Commons Lang 的 SerializationUtils
PromotionActivity newActivity = SerializationUtils.clone(oldActivity);

```

這雖然方便且安全（保證完全深拷貝），但效能較差。在需要極速複製的大規模系統中，我們仍傾向於手寫 `clone()` 或使用 **Copy Constructor**。

## 為什麼在電商系統中重要？

1. **效能優化**：有些物件的初始化需要查資料庫或進行複雜運算，透過 `clone()` 直接拷貝記憶體快照會快得多。
2. **狀態快照**：在處理訂單結帳時，我們需要「凍結」當時的促銷規則。透過原型模式複製一份當下的規則存入訂單，可以防止未來規則修改時影響到歷史訂單。
3. **減少重複代碼**：不需要再寫冗長的 `new` 與 `set` 邏輯。

## 結語

原型模式雖然在 Java 中因為 `Cloneable` 介面的設計缺陷（沒有定義 clone 方法）而飽受爭議，但其「以現有物件為範本」的思想在複雜業務建模中是不可或缺的。
