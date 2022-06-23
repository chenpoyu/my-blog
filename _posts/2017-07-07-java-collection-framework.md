---
layout: post
title: "Java 集合框架使用筆記"
date: 2017-07-07 09:15:00 +0800
categories: [設計模式, Java]
tags: [Java, 集合框架, 資料結構]
---

在開發電商系統時，我們時常需要處理大量的商品、訂單、用戶資料。選擇合適的集合容器不僅影響程式的效能，更關係到代碼的可讀性與維護性。

## 常見的集合選擇困境

剛開始寫 Java 時，我習慣所有東西都用 `ArrayList`。但後來發現：
- 當需要快速查找商品時，`ArrayList` 的 O(n) 查詢效率讓系統變慢
- 處理購物車時，重複商品的判斷變得複雜
- 訂單排程需要優先隊列，但 `ArrayList` 不支援

## Collection Framework 架構

Java 集合框架主要分為兩大體系：

### 1. Collection 介面家族
```java
// List: 有序、可重複
List<Product> productList = new ArrayList<>();

// Set: 無序、不可重複
Set<String> uniqueCategories = new HashSet<>();

// Queue: 佇列操作
Queue<Order> orderQueue = new LinkedList<>();
```

### 2. Map 介面家族
```java
// 鍵值對映射
Map<String, Product> productCache = new HashMap<>();
```

## 實戰案例：電商購物車設計

### 場景一：商品列表
使用 `ArrayList` 適合需要依索引存取、順序遍歷的場景：

```java
public class ProductCatalog {
    private List<Product> products = new ArrayList<>();
    
    public void addProduct(Product product) {
        products.add(product);
    }
    
    public Product getProductByIndex(int index) {
        return products.get(index);  // O(1) 時間複雜度
    }
    
    public void displayProducts() {
        for (Product product : products) {
            System.out.println(product.getName());
        }
    }
}
```

### 場景二：快速查找商品
當需要頻繁根據 ID 查找商品時，使用 `HashMap`：

```java
public class ProductRepository {
    private Map<String, Product> productMap = new HashMap<>();
    
    public void addProduct(Product product) {
        productMap.put(product.getId(), product);
    }
    
    public Product findById(String id) {
        return productMap.get(id);  // O(1) 平均時間複雜度
    }
    
    public boolean hasProduct(String id) {
        return productMap.containsKey(id);
    }
}
```

### 場景三：購物車去重
使用 `HashSet` 確保商品類別不重複：

```java
public class ShoppingCart {
    private Map<Product, Integer> items = new HashMap<>();
    private Set<String> appliedCoupons = new HashSet<>();
    
    public void addItem(Product product, int quantity) {
        items.put(product, items.getOrDefault(product, 0) + quantity);
    }
    
    public boolean applyCoupon(String couponCode) {
        if (appliedCoupons.contains(couponCode)) {
            System.out.println("優惠券已使用過！");
            return false;
        }
        appliedCoupons.add(couponCode);
        return true;
    }
    
    public Set<String> getUniqueCategories() {
        Set<String> categories = new HashSet<>();
        for (Product product : items.keySet()) {
            categories.add(product.getCategory());
        }
        return categories;
    }
}
```

## 效能比較表

| 操作 | ArrayList | LinkedList | HashSet | HashMap |
|------|-----------|------------|---------|---------|
| 新增 | O(1) | O(1) | O(1) | O(1) |
| 刪除 | O(n) | O(1)* | O(1) | O(1) |
| 查找 | O(n) | O(n) | O(1) | O(1) |
| 索引存取 | O(1) | O(n) | - | - |

*LinkedList 刪除已知節點為 O(1)，查找後刪除為 O(n)



## 實測結果

為了驗證上述效能差異,我寫了一個測試程式:

```java
public class CollectionPerformanceTest {
    public static void main(String[] args) {
        int size = 100000;
        
        // 測試 ArrayList 與 LinkedList 的插入效能
        testInsert(size);
        
        // 測試隨機存取效能
        testRandomAccess(size);
        
        // 測試查找效能
        testSearch(size);
    }
    
    static void testInsert(int size) {
        long start = System.currentTimeMillis();
        List<Integer> arrayList = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
        }
        long arrayListTime = System.currentTimeMillis() - start;
        
        start = System.currentTimeMillis();
        List<Integer> linkedList = new LinkedList<>();
        for (int i = 0; i < size; i++) {
            linkedList.add(i);
        }
        long linkedListTime = System.currentTimeMillis() - start;
        
        System.out.println("插入 " + size + " 個元素:");
        System.out.println("ArrayList: " + arrayListTime + "ms");
        System.out.println("LinkedList: " + linkedListTime + "ms");
    }
    
    static void testRandomAccess(int size) {
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }
        
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10000; i++) {
            arrayList.get((int)(Math.random() * size));
        }
        long arrayListTime = System.currentTimeMillis() - start;
        
        start = System.currentTimeMillis();
        for (int i = 0; i < 10000; i++) {
            linkedList.get((int)(Math.random() * size));
        }
        long linkedListTime = System.currentTimeMillis() - start;
        
        System.out.println("\n隨機存取 10000 次:");
        System.out.println("ArrayList: " + arrayListTime + "ms");
        System.out.println("LinkedList: " + linkedListTime + "ms");
    }
    
    static void testSearch(int size) {
        List<Integer> list = new ArrayList<>();
        Set<Integer> set = new HashSet<>();
        for (int i = 0; i < size; i++) {
            list.add(i);
            set.add(i);
        }
        
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10000; i++) {
            list.contains((int)(Math.random() * size));
        }
        long listTime = System.currentTimeMillis() - start;
        
        start = System.currentTimeMillis();
        for (int i = 0; i < 10000; i++) {
            set.contains((int)(Math.random() * size));
        }
        long setTime = System.currentTimeMillis() - start;
        
        System.out.println("\n查找 10000 次:");
        System.out.println("ArrayList: " + listTime + "ms");
        System.out.println("HashSet: " + setTime + "ms");
    }
}
```

執行結果:

```
插入 100000 個元素:
ArrayList: 8ms
LinkedList: 12ms

隨機存取 10000 次:
ArrayList: 3ms
LinkedList: 18642ms

查找 10000 次:
ArrayList: 2847ms
HashSet: 2ms
```

從實測數據可以看出:
- **插入效能**: ArrayList 和 LinkedList 差不多,ArrayList 略快
- **隨機存取**: ArrayList 快了 6000+ 倍! LinkedList 的 O(n) 複雜度在大數據量時很致命
- **查找效能**: HashSet 快了 1400+ 倍,O(1) vs O(n) 的差距非常明顯

這也驗證了為什麼我們說:
- 需要頻繁隨機存取 → 用 ArrayList
- 需要快速查找存在性 → 用 HashSet/HashMap

## 選擇建議

1. **需要依序排列、頻繁讀取** → `ArrayList`
2. **頻繁插入刪除、不在乎順序** → `LinkedList`
3. **需要去重、快速查找存在性** → `HashSet`
4. **鍵值對應、快速查找** → `HashMap`
5. **需要排序** → `TreeSet` / `TreeMap`
6. **執行緒安全** → `ConcurrentHashMap`

## 小結

選擇集合容器就像選擇設計模式一樣，沒有銀彈。了解每種容器的特性與適用場景，才能在實際開發中做出正確的決策。

下一篇，我們將深入探討 **Java 泛型**，看看如何讓這些集合容器更加型別安全。
