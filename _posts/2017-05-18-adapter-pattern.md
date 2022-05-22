---
layout: post 
title: "轉接器模式：統一電商多樣化物流 API 的轉換藝術" 
date: 2017-05-18 09:50:00 +0800 
categories: [設計模式, Java] 
tags: [適配器模式, 系統集成]
---

電商系統中，一定都會串接物流系統，我們系統中就要要呼叫外單位的物流系統來建立託運單。在我理想中的標準介面非常簡潔：

```java
public interface LogisticsService {
    void createShipment(String orderId, String address);
}

```

然而，現實是殘酷的。我們對接的 A 物流公司提供的 SDK 類別長這樣：

```java
public class AAALogisticsSDK {
    public void sendPackage(Map<String, String> data) {
        System.out.println("傳送 XML 封裝的資料到快速物流...");
    }
}

```

而另一家 B 物流公司 的 API 則是：

```java
public class BBBDeliveryAPI {
    public void postToCloud(String id, String addr, String sign) {
        System.out.println("執行 JSON 簽名並上傳到全省宅配...");
    }
}

```

如果直接在 `OrderService` 裡寫 `if (type == AAA) { ... }`，我的核心業務邏輯就會被這些第三方 SDK 的細節給污染。

## 轉接器模式 (Adapter Pattern) 的解法

轉接器模式的精神是：**「將一個類別的介面轉換成客戶端希望的另一個介面，使得原本由於介面不相容而不能一起工作的類別可以一起工作。」**

### 1. 實作轉接器

我為每一家物流商寫一個專用的轉接器，讓它們看起來都符合 `LogisticsService`。

#### A 物流轉接器

```java
public class AAALogisticsAdapter implements LogisticsService {
    private AAALogisticsSDK sdk = new AAALogisticsSDK();

    @Override
    public void createShipment(String orderId, String address) {
        // 負責繁瑣的參數轉換邏輯
        Map<String, String> params = new HashMap<>();
        params.add("ID", orderId);
        params.add("ADDR", address);
        sdk.sendPackage(params);
    }
}

```

#### B 物流轉接器

```java
public class BBBDeliveryAdapter implements LogisticsService {
    private BBBDeliveryAPI api = new BBBDeliveryAPI();

    @Override
    public void createShipment(String orderId, String address) {
        // 負責簽名運算與 JSON 轉換
        String signature = MD5Util.sign(orderId);
        api.postToCloud(orderId, address, signature);
    }
}

```

## 物件轉接器 vs. 類別轉接器

其實這次的內容，我面臨兩種選擇：

* **物件轉接器 (Object Adapter)**：使用「組合」，轉接器內部持有第三方物件。
* **類別轉接器 (Class Adapter)**：使用「多重繼承」（在 Java 中是繼承第三方類別並實作介面）。

**結論：** 我幾乎 100% 選擇 **物件轉接器**。因為 Java 不支援多重繼承，且使用「組合」比「繼承」更靈活，我們不需要為了轉接而被迫繼承第三方 SDK 中那些可能有害的方法。

## 轉接器模式在電商系統中的優勢

1. **保護核心邏輯**：當第三方 SDK 改版時，你只需要修改對應的 Adapter，`OrderService` 完全不需要變動。
2. **符合單一職責**：將「參數格式轉換」與「核心業務邏輯」徹底分離。
3. **靈活切換**：更換物流商就像換個外插組件一樣簡單，實現了真正的「插拔式架構」。
