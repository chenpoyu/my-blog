---
layout: post
title: "Java 泛型使用筆記"
date: 2017-07-14 14:25:00 +0800
categories: [設計模式, Java]
tags: [Java, 泛型, 型別安全]
---

上週我們學習了集合框架（請參考 [Java 集合框架](/posts/2017/07/07/java-collection-framework/)），但有個問題一直困擾著我：如何確保放入容器的資料型別是正確的？

## 沒有泛型的黑暗時代

在 Java 1.5 之前，集合只能儲存 `Object`：

```java
List products = new ArrayList();
products.add(new Product("iPhone", 30000));
products.add("不小心放錯了");  // 編譯器不會報錯！

Product p = (Product) products.get(0);  // 需要強制轉型
Product p2 = (Product) products.get(1); // 執行時爆炸！ClassCastException
```

這種做法有兩大問題：
1. **沒有編譯時期的型別檢查**：錯誤要到執行時才會發現
2. **需要大量的型別轉換**：代碼冗長且容易出錯

## 泛型的救贖

泛型讓我們可以在編譯時期就指定容器的型別：

```java
List<Product> products = new ArrayList<>();
products.add(new Product("iPhone", 30000));  // OK
products.add("字串");  // 編譯錯誤！型別不符

Product p = products.get(0);  // 不需要轉型
```

## 實戰案例：電商泛型工具類

### 場景一：通用的分頁結果

```java
public class PageResult<T> {
    private List<T> items;
    private int currentPage;
    private int totalPages;
    private long totalCount;
    
    public PageResult(List<T> items, int currentPage, int totalPages, long totalCount) {
        this.items = items;
        this.currentPage = currentPage;
        this.totalPages = totalPages;
        this.totalCount = totalCount;
    }
    
    public List<T> getItems() {
        return items;
    }
    
    public boolean hasNextPage() {
        return currentPage < totalPages;
    }
}

// 使用時可以套用到任何實體
PageResult<Product> productPage = productService.getProducts(1, 20);
PageResult<Order> orderPage = orderService.getOrders(1, 20);
PageResult<User> userPage = userService.getUsers(1, 20);
```

### 場景二：通用的 Repository 介面

```java
public interface Repository<T, ID> {
    T save(T entity);
    T findById(ID id);
    List<T> findAll();
    void delete(ID id);
    boolean exists(ID id);
}

// 實作商品 Repository
public class ProductRepository implements Repository<Product, String> {
    private Map<String, Product> storage = new HashMap<>();
    
    @Override
    public Product save(Product entity) {
        storage.put(entity.getId(), entity);
        return entity;
    }
    
    @Override
    public Product findById(String id) {
        return storage.get(id);
    }
    
    @Override
    public List<Product> findAll() {
        return new ArrayList<>(storage.values());
    }
    
    @Override
    public void delete(String id) {
        storage.remove(id);
    }
    
    @Override
    public boolean exists(String id) {
        return storage.containsKey(id);
    }
}
```

### 場景三：型別安全的 API 回應

```java
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private String errorCode;
    
    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.success = true;
        response.data = data;
        return response;
    }
    
    public static <T> ApiResponse<T> error(String message, String errorCode) {
        ApiResponse<T> response = new ApiResponse<>();
        response.success = false;
        response.message = message;
        response.errorCode = errorCode;
        return response;
    }
    
    // Getters...
}

// 使用範例
public ApiResponse<Product> getProduct(String id) {
    Product product = productRepository.findById(id);
    if (product == null) {
        return ApiResponse.error("商品不存在", "PRODUCT_NOT_FOUND");
    }
    return ApiResponse.success(product);
}
```

## 泛型的限制與通配符

### 問題：為什麼不能這樣寫？

```java
List<Product> products = new ArrayList<>();
List<Object> objects = products;  // 編譯錯誤！
```

雖然 `Product` 繼承自 `Object`，但 `List<Product>` 並不是 `List<Object>` 的子類。這是為了保證型別安全。

### 解決方案：使用通配符

```java
// 上界通配符：只能讀取，不能寫入（除了 null）
public double calculateTotalPrice(List<? extends Product> products) {
    double total = 0;
    for (Product p : products) {
        total += p.getPrice();
    }
    return total;
}

// 可以接受 List<Product>、List<DigitalProduct>、List<PhysicalProduct>
List<DigitalProduct> digitalProducts = new ArrayList<>();
calculateTotalPrice(digitalProducts);  // OK
```

```java
// 下界通配符：可以寫入，但讀取時只能當作 Object
public void addProducts(List<? super Product> products) {
    products.add(new Product("新商品", 100));
    products.add(new DigitalProduct("電子書", 50));
}
```

### PECS 原則（Producer Extends Consumer Super）

- **Producer（生產者）** 用 `extends`：當你只需要從集合中讀取資料
- **Consumer（消費者）** 用 `super`：當你需要往集合中寫入資料

```java
// 複製商品清單的方法
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T item : src) {
        dest.add(item);
    }
}
```

## 泛型方法

有時候我們只需要讓某個方法支援泛型：

```java
public class CollectionUtils {
    
    // 泛型方法：尋找第一個符合條件的元素
    public static <T> T findFirst(List<T> list, Predicate<T> condition) {
        for (T item : list) {
            if (condition.test(item)) {
                return item;
            }
        }
        return null;
    }
    
    // 使用範例
    public static void main(String[] args) {
        List<Product> products = getProducts();
        
        // 尋找第一個價格超過 1000 的商品
        Product expensiveProduct = findFirst(products, 
            p -> p.getPrice() > 1000);
    }
}
```

## 小結

泛型是 Java 型別系統的重要組成部分，它提供：
1. **編譯時期的型別檢查**：提早發現錯誤
2. **消除型別轉換**：讓代碼更簡潔
3. **提高代碼重用性**：一次實作，多處使用

掌握泛型，是寫出優雅、安全 Java 代碼的關鍵。下週我們將探討 **Java 例外處理**，看看如何優雅地處理電商系統中的各種異常情況。
