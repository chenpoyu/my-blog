---
layout: post 
title: "享元模式：在高併發電商系統中優化大規模物件的記憶體佔用" 
date: 2017-01-14 15:45:00 +0800 
categories: [設計模式, Java] 
tags: [享元模式, 效能優化]
---

在去年的雙 11 促銷活動中，系統監控顯示記憶體使用量異常升高。分析後發現，購物車與商品列表系統中，存在大量重複的物件。例如，「iPhone 8 黑色 128G」這個商品，雖然有 1000 個使用者把它加入購物車，但在 JVM 裡卻產生了 1000 個幾乎完全相同的物件實例。

## 核心問題：內在狀態與外在狀態

在 OOAD 中，物件的狀態可以分為兩類：

1. **內在狀態 (Internal State)**：不隨環境改變、可以共享的資訊（如：商品名稱、型號、基本規格、圖片 URL）。
2. **外在狀態 (External State)**：隨環境改變、不能共享的資訊（如：購買數量、使用者 ID、加入時間）。

享元模式的核心就是：**剝離外在狀態，共享內在狀態。**

## 享元模式 (Flyweight Pattern) 的重構實踐

### 1. 定義享元物件 (內在狀態)

我們將商品的固定資訊封裝在 `ProductMetaData` 中。

```java
public class ProductMetaData {
    private final String name;
    private final String model;
    private final String manufacturer;
    private final String imageUrl;

    public ProductMetaData(String name, String model, String manufacturer, String imageUrl) {
        this.name = name;
        this.model = model;
        this.manufacturer = manufacturer;
        this.imageUrl = imageUrl;
    }

    public void display(int quantity, String userId) {
        System.out.println("使用者 " + userId + " 的購物車：[" + name + "] 數量：" + quantity);
    }
}

```

### 2. 建立享元工廠 (Flyweight Factory)

工廠負責管理與緩存這些共享物件，確保相同的元數據只會被建立一次。

```java
import java.util.HashMap;
import java.util.Map;

public class ProductFactory {
    private static final Map<String, ProductMetaData> productCache = new HashMap<>();

    public static ProductMetaData getProductMetaData(String name, String model, String manufacturer, String imageUrl) {
        String key = name + ":" + model;
        if (!productCache.containsKey(key)) {
            System.out.println("== 建立新的商品元數據物件: " + key + " ==");
            productCache.put(key, new ProductMetaData(name, model, manufacturer, imageUrl));
        }
        return productCache.get(key);
    }
}

```

### 3. 客戶端物件 (包含外在狀態)

購物車中的每一項（CartItem）現在只需引用共享的 MetaData。

```java
public class CartItem {
    private int quantity;    // 外在狀態
    private String userId;   // 外在狀態
    private ProductMetaData metaData; // 共享的內在狀態

    public CartItem(int quantity, String userId, ProductMetaData metaData) {
        this.quantity = quantity;
        this.userId = userId;
        this.metaData = metaData;
    }

    public void show() {
        metaData.display(quantity, userId);
    }
}

```

## Java String Pool 的啟示

其實我們每天都在用享元模式。Java 的 **String Pool** 就是最經典的例子。當寫 `String s1 = "abc"; String s2 = "abc";` 時，JVM 只會在常量池建立一個 "abc" 物件。

## 注意事項與缺點

享元模式並非萬靈丹：

1. **複雜度提升**：程式碼變得比較破碎，需要區分內外狀態。
2. **執行時間換空間**：每次獲取物件需要透過 Factory 查詢，雖然節省了記憶體，但會微幅增加運算開銷。
3. **多執行緒安全**：工廠內的 `Map` 必須處理併發問題（例如使用 `ConcurrentHashMap`）。

## 結語

享元模式是「空間換時間」的反向操作，它教導我們如何在物件導向的優雅度與硬體限制之間取得平衡。這個模式保住了我們系統的穩定性，避免了因為 OOM (Out Of Memory) 而導致的災難。
