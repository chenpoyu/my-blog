---
layout: post
title: "Java 例外處理筆記"
date: 2017-07-21 16:40:00 +0800
categories: [設計模式, Java]
tags: [Java, 例外處理, 錯誤處理]
---

在電商系統中，錯誤無所不在：庫存不足、付款失敗、網路中斷、資料格式錯誤...。如何優雅地處理這些例外情況，是區分初學者與資深開發者的關鍵。

## 常見的錯誤處理方式

### 反模式：吞掉例外

```java
public Product getProduct(String id) {
    try {
        return productRepository.findById(id);
    } catch (Exception e) {
        // 什麼都不做，假裝沒事發生
        return null;
    }
}
```

這是最糟糕的做法！錯誤被隱藏，日後除錯困難。

### 反模式：過度捕捉

```java
public void processOrder(Order order) {
    try {
        validateOrder(order);
        checkInventory(order);
        processPayment(order);
        sendNotification(order);
    } catch (Exception e) {  // 捕捉所有例外
        e.printStackTrace();  // 只印出來
        throw new RuntimeException("訂單處理失敗");  // 包裝後丟出
    }
}
```

這樣做會失去具體的錯誤資訊，讓除錯變得困難。

## Java 例外體系

```
Throwable
├── Error (系統級錯誤，如 OutOfMemoryError)
└── Exception
    ├── RuntimeException (非檢查型例外)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   └── ...
    └── 檢查型例外 (Checked Exception)
        ├── IOException
        ├── SQLException
        └── ...
```

### 檢查型 vs 非檢查型

- **檢查型例外 (Checked Exception)**：必須明確處理，否則編譯錯誤
- **非檢查型例外 (Unchecked Exception)**：不強制處理，通常是程式邏輯錯誤

## 實戰案例：電商例外處理

### 場景一：自定義業務例外

```java
// 基礎業務例外
public class BusinessException extends RuntimeException {
    private String errorCode;
    
    public BusinessException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public String getErrorCode() {
        return errorCode;
    }
}

// 具體的業務例外
public class InsufficientStockException extends BusinessException {
    private String productId;
    private int requestedQuantity;
    private int availableQuantity;
    
    public InsufficientStockException(String productId, int requested, int available) {
        super(String.format("商品 %s 庫存不足：需要 %d，可用 %d", 
            productId, requested, available), "INSUFFICIENT_STOCK");
        this.productId = productId;
        this.requestedQuantity = requested;
        this.availableQuantity = available;
    }
    
    // Getters...
}

public class PaymentFailedException extends BusinessException {
    private String paymentId;
    
    public PaymentFailedException(String paymentId, String reason) {
        super("付款失敗：" + reason, "PAYMENT_FAILED");
        this.paymentId = paymentId;
    }
    
    public String getPaymentId() {
        return paymentId;
    }
}
```

### 場景二：訂單處理的例外處理

```java
public class OrderService {
    private InventoryService inventoryService;
    private PaymentService paymentService;
    private NotificationService notificationService;
    
    public OrderResult processOrder(Order order) {
        try {
            // 1. 驗證訂單
            validateOrder(order);
            
            // 2. 檢查並扣除庫存
            inventoryService.reserveStock(order.getItems());
            
            // 3. 處理付款
            Payment payment = paymentService.processPayment(order);
            
            // 4. 更新訂單狀態
            order.setStatus(OrderStatus.COMPLETED);
            order.setPaymentId(payment.getId());
            
            // 5. 發送通知
            notificationService.sendOrderConfirmation(order);
            
            return OrderResult.success(order);
            
        } catch (InsufficientStockException e) {
            // 庫存不足：回傳明確訊息
            return OrderResult.failure(
                "庫存不足：" + e.getMessage(),
                e.getErrorCode()
            );
            
        } catch (PaymentFailedException e) {
            // 付款失敗：需要回滾庫存
            inventoryService.releaseStock(order.getItems());
            return OrderResult.failure(
                "付款失敗：" + e.getMessage(),
                e.getErrorCode()
            );
            
        } catch (Exception e) {
            // 其他未預期的錯誤：記錄日誌並回滾
            logger.error("訂單處理失敗：" + order.getId(), e);
            inventoryService.releaseStock(order.getItems());
            return OrderResult.failure(
                "系統錯誤，請稍後再試",
                "SYSTEM_ERROR"
            );
        }
    }
    
    private void validateOrder(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("訂單不得為 null");
        }
        if (order.getItems() == null || order.getItems().isEmpty()) {
            throw new BusinessException("訂單項目不得為空", "EMPTY_ORDER");
        }
        if (order.getTotalAmount() <= 0) {
            throw new BusinessException("訂單金額必須大於 0", "INVALID_AMOUNT");
        }
    }
}
```

### 場景三：Try-with-resources 處理資源

```java
public class OrderExporter {
    
    // 舊的做法：手動關閉資源
    public void exportOrdersOldWay(List<Order> orders, String filePath) {
        FileWriter writer = null;
        try {
            writer = new FileWriter(filePath);
            for (Order order : orders) {
                writer.write(order.toCsv() + "\n");
            }
        } catch (IOException e) {
            logger.error("匯出訂單失敗", e);
            throw new BusinessException("匯出失敗", "EXPORT_FAILED");
        } finally {
            if (writer != null) {
                try {
                    writer.close();
                } catch (IOException e) {
                    logger.error("關閉檔案失敗", e);
                }
            }
        }
    }
    
    // 新的做法：try-with-resources
    public void exportOrders(List<Order> orders, String filePath) {
        try (FileWriter writer = new FileWriter(filePath);
             BufferedWriter buffered = new BufferedWriter(writer)) {
            
            for (Order order : orders) {
                buffered.write(order.toCsv());
                buffered.newLine();
            }
            
        } catch (IOException e) {
            logger.error("匯出訂單失敗", e);
            throw new BusinessException("匯出失敗", "EXPORT_FAILED");
        }
        // 資源自動關閉，不需要 finally
    }
}
```

## 例外處理的實踐經驗

### 1. 只捕捉你能處理的例外

```java
//  不好的做法
try {
    // ...
} catch (Exception e) {
    // 不知道該怎麼處理
}

//  好的做法
try {
    // ...
} catch (InsufficientStockException e) {
    // 明確知道如何處理庫存不足
    notifyUser("商品庫存不足，請選擇其他商品");
}
```

### 2. 不要使用例外來控制流程

```java
//  不好的做法
try {
    Product p = findProduct(id);
    return p.getPrice();
} catch (ProductNotFoundException e) {
    return 0.0;  // 使用例外來處理正常邏輯
}

//  好的做法
Product p = findProduct(id);
if (p == null) {
    return 0.0;
}
return p.getPrice();
```

### 3. 提供有意義的錯誤訊息

```java
//  不好的做法
throw new Exception("錯誤");

//  好的做法
throw new InsufficientStockException(
    productId, 
    requestedQty, 
    availableQty
);
```

### 4. 記錄例外日誌

```java
public void processOrder(Order order) {
    try {
        // ...
    } catch (PaymentFailedException e) {
        logger.error("訂單 {} 付款失敗，付款ID：{}", 
            order.getId(), e.getPaymentId(), e);
        throw e;
    }
}
```

## 小結

優雅的例外處理應該：
1. **明確分類**：使用自定義例外表達業務意圖
2. **適度捕捉**：只處理能夠處理的錯誤
3. **保留資訊**：不要吞掉例外，記錄完整上下文
4. **及時清理**：使用 try-with-resources 管理資源

下週我們將進入 **Java 8 新特性**，探討 Lambda 表達式和 Stream API 如何讓代碼更簡潔。
