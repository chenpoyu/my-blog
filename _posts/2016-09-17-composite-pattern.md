---
layout: post 
title: "組合模式：重構電商平台的商品套裝與促銷結構" 
date: 2016-09-17 14:20:00 +0800 
categories: [設計模式, Java] 
tags: [組合模式, 電商系統]
---

最近我遇到了有點一個複雜的開發需求：電商活動要支援「超值大禮包」。

這不是簡單的買 A 送 B，而是：一個禮包可以包含多個單品，甚至可以包含另一個小禮包（例如：中秋禮盒組包含月餅，以及一個茶葉禮盒）。對於購物車來說，不論是「單一商品」還是「組合套裝」，都應該能一致地計算價格、顯示描述。

## 原始問題：區分單品與套裝的痛苦

最初的代碼中，購物車類別充滿了型別判斷：

```java
for (Object item : cartItems) {
    if (item instanceof SingleProduct) {
        total += ((SingleProduct)item).getPrice();
    } else if (item instanceof BundlePackage) {
        total += ((BundlePackage)item).getBundlePrice();
    }
}

```

當我想在禮包內再嵌套禮包時，這種 `if-else` 或 `instanceof` 的寫法會讓邏輯陷入地獄。

## 組合模式 (Composite Pattern) 的解法

組合模式的精神在於：**「讓客戶端對單一物件和組合物件的使用具有一致性。」** 將商品與套裝抽象化為同一個介面。

### 1. 定義元件介面 (Component)

不論是單品還是套裝，都必須實作這個介面。

```java
public interface CartItem {
    double getPrice();
    String getName();
    void showDetails(int depth);
}

```

### 2. 實作單一商品 (Leaf)

```java
public class Product implements CartItem {
    private String name;
    private double price;

    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }

    @Override
    public double getPrice() { return price; }

    @Override
    public String getName() { return name; }

    @Override
    public void showDetails(int depth) {
        System.out.println("  ".repeat(depth) + "- " + name + ": $" + price);
    }
}

```

### 3. 實作組合套裝 (Composite)

套裝內部維護一個清單，可以存放任何實作了 `CartItem` 的物件。

```java
import java.util.ArrayList;
import java.util.List;

public class BundleSale implements CartItem {
    private String bundleName;
    private List<CartItem> items = new ArrayList<>();
    private double discountRate = 0.9; // 套裝 9 折

    public BundleSale(String name) {
        this.bundleName = name;
    }

    public void add(CartItem item) { items.add(item); }

    @Override
    public double getPrice() {
        return items.stream().mapToDouble(CartItem::getPrice).sum() * discountRate;
    }

    @Override
    public String getName() { return bundleName; }

    @Override
    public void showDetails(int depth) {
        System.out.println("  ".repeat(depth) + "+ 套裝: " + bundleName);
        for (CartItem item : items) {
            item.showDetails(depth + 1);
        }
    }
}

```

## 實戰中：遞迴結構

現在，我可以輕鬆建立一個「大禮包套小禮包」的結構，而客戶端（購物車）完全不需要知道細節：

```java
// 建立單品
CartItem mooncake = new Product("廣式月餅", 100);
CartItem greenTea = new Product("高山綠茶", 200);

// 建立小禮包
BundleSale teaSet = new BundleSale("精緻茶具組");
teaSet.add(greenTea);
teaSet.add(new Product("茶杯", 50));

// 建立中秋大禮包
BundleSale midAutumnBox = new BundleSale("中秋至尊禮盒");
midAutumnBox.add(mooncake);
midAutumnBox.add(teaSet); // 嵌套組合！

// 購物車結帳
System.out.println("總價: " + midAutumnBox.getPrice());
midAutumnBox.showDetails(0);

```

## 結語與省思

透過組合模式，我可以將電商系統中的商品結構轉化為一棵「樹」。這帶來了兩個巨大的好處：

1. **高擴展性**：未來要增加「福袋」、「加價購組」，只需實作 `CartItem` 即可。
2. **簡化客戶端**：計算總金額時，購物車只需要呼叫頂層物件的 `getPrice()`，剩下的遞迴邏輯由物件內部自行處理。

這套架構支撐了極其複雜的組合促銷邏輯，且完全沒有因為層層嵌套而產生邏輯 Bug。
