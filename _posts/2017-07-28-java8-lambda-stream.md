---
layout: post
title: "Java 8 Lambda 與 Stream 使用筆記"
date: 2017-07-28 10:35:00 +0800
categories: [設計模式, Java]
tags: [Java, Lambda, Stream, 函數式程式設計]
---

Java 8 是 Java 發展史上的重要里程碑，引入了 Lambda 表達式與 Stream API，讓 Java 也能寫出函數式風格的代碼。這對處理集合資料密集的電商系統來說，簡直是神器。

## Lambda 表達式：告別匿名內部類

### 舊的寫法

```java
// 過濾價格大於 1000 的商品
List<Product> expensiveProducts = new ArrayList<>();
for (Product product : products) {
    if (product.getPrice() > 1000) {
        expensiveProducts.add(product);
    }
}

// 使用 Comparator 排序
Collections.sort(products, new Comparator<Product>() {
    @Override
    public int compare(Product p1, Product p2) {
        return Double.compare(p1.getPrice(), p2.getPrice());
    }
});
```

冗長且難以閱讀。

### Lambda 表達式寫法

```java
// 過濾價格大於 1000 的商品
List<Product> expensiveProducts = products.stream()
    .filter(p -> p.getPrice() > 1000)
    .collect(Collectors.toList());

// 使用 Lambda 排序
Collections.sort(products, (p1, p2) -> 
    Double.compare(p1.getPrice(), p2.getPrice()));

// 更簡潔的方法引用
Collections.sort(products, Comparator.comparingDouble(Product::getPrice));
```

簡潔且語意清晰！

## Lambda 語法

```java
// 完整寫法
(Product p) -> { return p.getPrice() > 1000; }

// 省略型別（編譯器會推斷）
(p) -> { return p.getPrice() > 1000; }

// 單一參數省略括號
p -> { return p.getPrice() > 1000; }

// 單一表達式省略 return 和大括號
p -> p.getPrice() > 1000
```

## 實戰案例：使用 Stream API 處理商品資料

### 場景一：篩選與轉換

```java
public class ProductService {
    
    // 找出所有電子產品且價格大於 5000 的商品名稱
    public List<String> getExpensiveElectronics(List<Product> products) {
        return products.stream()
            .filter(p -> "電子產品".equals(p.getCategory()))
            .filter(p -> p.getPrice() > 5000)
            .map(Product::getName)
            .collect(Collectors.toList());
    }
    
    // 計算所有商品的平均價格
    public double getAveragePrice(List<Product> products) {
        return products.stream()
            .mapToDouble(Product::getPrice)
            .average()
            .orElse(0.0);
    }
    
    // 找出最貴的商品
    public Optional<Product> getMostExpensive(List<Product> products) {
        return products.stream()
            .max(Comparator.comparingDouble(Product::getPrice));
    }
}
```

### 執行範例

假設我們有以下測試資料:

```java
List<Product> products = Arrays.asList(
    new Product("iPhone 8", "電子產品", 21900),
    new Product("MacBook Pro", "電子產品", 65900),
    new Product("AirPods", "電子產品", 4990),
    new Product("Nike Air", "運動用品", 3200),
    new Product("Adidas", "運動用品", 2800),
    new Product("T-shirt", "服飾", 590)
);

ProductService service = new ProductService();

// 執行篩選
List<String> result = service.getExpensiveElectronics(products);
System.out.println("價格 > 5000 的電子產品: " + result);

// 計算平均價格
double avgPrice = service.getAveragePrice(products);
System.out.println("平均價格: " + avgPrice);

// 找出最貴商品
service.getMostExpensive(products)
    .ifPresent(p -> System.out.println("最貴商品: " + p.getName() + " - " + p.getPrice()));
```

執行結果:

```
價格 > 5000 的電子產品: [iPhone 8, MacBook Pro]
平均價格: 16663.33
最貴商品: MacBook Pro - 65900.0
```

可以看到 Stream API 讓數據處理變得直觀多了。

### 場景二：分組與統計

```java
public class OrderAnalytics {
    
    // 按類別分組商品
    public Map<String, List<Product>> groupByCategory(List<Product> products) {
        return products.stream()
            .collect(Collectors.groupingBy(Product::getCategory));
    }
    
    // 計算每個類別的商品數量
    public Map<String, Long> countByCategory(List<Product> products) {
        return products.stream()
            .collect(Collectors.groupingBy(
                Product::getCategory,
                Collectors.counting()
            ));
    }
    
    // 計算每個類別的總銷售額
    public Map<String, Double> salesByCategory(List<OrderItem> orderItems) {
        return orderItems.stream()
            .collect(Collectors.groupingBy(
                item -> item.getProduct().getCategory(),
                Collectors.summingDouble(OrderItem::getTotalPrice)
            ));
    }
    
    // 找出每個類別最暢銷的商品
    public Map<String, Optional<Product>> bestSellerByCategory(
            List<Product> products, 
            Map<String, Integer> salesCount) {
        
        return products.stream()
            .collect(Collectors.groupingBy(
                Product::getCategory,
                Collectors.maxBy(
                    Comparator.comparingInt(p -> salesCount.getOrDefault(p.getId(), 0))
                )
            ));
    }
}
```

### 場景三：複雜的訂單處理

```java
public class OrderProcessor {
    
    // 計算訂單總金額（包含折扣）
    public double calculateTotal(Order order) {
        return order.getItems().stream()
            .mapToDouble(item -> {
                double price = item.getProduct().getPrice();
                int quantity = item.getQuantity();
                double discount = item.getDiscount();
                return price * quantity * (1 - discount);
            })
            .sum();
    }
    
    // 檢查是否所有商品都有庫存
    public boolean checkAllInStock(Order order, InventoryService inventory) {
        return order.getItems().stream()
            .allMatch(item -> 
                inventory.getStock(item.getProduct().getId()) >= item.getQuantity()
            );
    }
    
    // 找出需要補貨的商品
    public List<Product> getOutOfStockProducts(List<Product> products, 
                                                InventoryService inventory) {
        return products.stream()
            .filter(p -> inventory.getStock(p.getId()) == 0)
            .collect(Collectors.toList());
    }
    
    // 產生訂單摘要報告
    public OrderSummary generateSummary(List<Order> orders) {
        OrderSummary summary = new OrderSummary();
        
        summary.setTotalOrders(orders.size());
        
        summary.setTotalRevenue(
            orders.stream()
                .mapToDouble(Order::getTotalAmount)
                .sum()
        );
        
        summary.setAverageOrderValue(
            orders.stream()
                .mapToDouble(Order::getTotalAmount)
                .average()
                .orElse(0.0)
        );
        
        summary.setTopProducts(
            orders.stream()
                .flatMap(order -> order.getItems().stream())
                .collect(Collectors.groupingBy(
                    item -> item.getProduct().getName(),
                    Collectors.summingInt(OrderItem::getQuantity)
                ))
                .entrySet().stream()
                .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
                .limit(10)
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    Map.Entry::getValue,
                    (e1, e2) -> e1,
                    LinkedHashMap::new
                ))
        );
        
        return summary;
    }
}
```

## Stream API 常用操作

### 中間操作 (Intermediate Operations)
這些操作會回傳新的 Stream，可以鏈式調用：

```java
// filter：過濾元素
stream.filter(p -> p.getPrice() > 100)

// map：轉換元素
stream.map(Product::getName)

// flatMap：攤平巢狀結構
orders.stream()
    .flatMap(order -> order.getItems().stream())

// distinct：去除重複
stream.distinct()

// sorted：排序
stream.sorted(Comparator.comparingDouble(Product::getPrice))

// limit：限制數量
stream.limit(10)

// skip：跳過元素
stream.skip(5)
```

### 終端操作 (Terminal Operations)
這些操作會觸發實際計算並產生結果：

```java
// collect：收集到集合
stream.collect(Collectors.toList())

// forEach：遍歷元素
stream.forEach(System.out::println)

// count：計數
stream.count()

// anyMatch / allMatch / noneMatch：檢查條件
stream.anyMatch(p -> p.getPrice() > 1000)

// findFirst / findAny：尋找元素
stream.findFirst()

// reduce：歸約操作
stream.reduce(0.0, Double::sum)

// min / max：找最小/最大值
stream.max(Comparator.comparingDouble(Product::getPrice))
```

## 函數式介面

Lambda 表達式的基礎是函數式介面（只有一個抽象方法的介面）：

```java
// Java 內建的常用函數式介面
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}

@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}

@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}

@FunctionalInterface
public interface Supplier<T> {
    T get();
}

// 自定義函數式介面
@FunctionalInterface
public interface PriceCalculator {
    double calculate(Product product, int quantity);
}

// 使用
PriceCalculator calculator = (product, qty) -> product.getPrice() * qty * 0.9;
double price = calculator.calculate(product, 5);
```

## 方法引用

當 Lambda 表達式只是調用一個方法時，可以用方法引用簡化：

```java
// Lambda 表達式
products.stream().map(p -> p.getName())

// 方法引用
products.stream().map(Product::getName)

// 靜態方法引用
stream.map(Double::valueOf)

// 建構子引用
stream.map(Product::new)
```

## 效能考量

雖然 Stream API 很方便，但也要注意：

1. **小集合不見得快**：Stream 有額外開銷，小於 100 個元素時傳統 for loop 可能更快
2. **平行化需謹慎**：`parallelStream()` 不是萬能，要測試
3. **避免多次遍歷**：盡量一次 Stream 完成所有操作

```java
//  不好的做法：多次遍歷
long count = products.stream().filter(p -> p.getPrice() > 100).count();
double sum = products.stream().filter(p -> p.getPrice() > 100)
    .mapToDouble(Product::getPrice).sum();

//  好的做法：一次遍歷
DoubleSummaryStatistics stats = products.stream()
    .filter(p -> p.getPrice() > 100)
    .mapToDouble(Product::getPrice)
    .summaryStatistics();
long count = stats.getCount();
double sum = stats.getSum();
```

## 小結

Lambda 與 Stream API 讓 Java 代碼更簡潔、更具表達力：
- **Lambda**：簡化函數式介面的實作
- **Stream**：聲明式處理集合資料
- **方法引用**：進一步簡化代碼

掌握這些特性，是現代 Java 開發的必備技能。下週我們將開始進入 **Spring 框架**，看看依賴注入如何改變我們設計應用程式的方式。
