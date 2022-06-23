---
layout: post
title: "Java 反射與註解使用筆記"
date: 2017-12-08 09:40:00 +0800
categories: [設計模式, Java]
tags: [Java, 反射, 註解, Reflection, Annotation]
---

在使用 Spring 框架時,會發現很多神奇的功能:@Autowired 自動注入、@Transactional 事務管理、@RequestMapping 路由映射。這些都是透過**反射**和**註解**實現的。

## 反射機制 (Reflection)

反射讓程式在執行時能夠檢查和操作類別、方法、屬性等資訊。

### 基本用法

```java
public class Product {
    private Long id;
    private String name;
    private Double price;
    
    public Product() {}
    
    public Product(Long id, String name, Double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
    
    public void updatePrice(Double newPrice) {
        this.price = newPrice;
    }
    
    private void internalMethod() {
        System.out.println("私有方法");
    }
    
    // getters/setters...
}
```

### 獲取 Class 物件

```java
// 方式1: 透過類別名稱
Class<?> clazz1 = Product.class;

// 方式2: 透過實例
Product product = new Product();
Class<?> clazz2 = product.getClass();

// 方式3: 透過類別名稱字串
Class<?> clazz3 = Class.forName("com.example.Product");
```

### 建立實例

```java
public class ReflectionExample {
    
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Product.class;
        
        // 使用無參建構子
        Product product1 = (Product) clazz.newInstance();
        
        // 使用有參建構子
        Constructor<?> constructor = clazz.getConstructor(
            Long.class, String.class, Double.class
        );
        Product product2 = (Product) constructor.newInstance(
            1L, "iPhone 8", 21900.0
        );
        
        System.out.println("產品名稱: " + product2.getName());
    }
}
```

執行結果:
```
產品名稱: iPhone 8
```

### 操作屬性

```java
public class FieldExample {
    
    public static void main(String[] args) throws Exception {
        Product product = new Product(1L, "MacBook", 65900.0);
        Class<?> clazz = product.getClass();
        
        // 獲取所有屬性
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            System.out.println("屬性: " + field.getName() + 
                             ", 類型: " + field.getType().getSimpleName());
        }
        
        // 修改私有屬性
        Field priceField = clazz.getDeclaredField("price");
        priceField.setAccessible(true);  // 允許存取私有屬性
        
        Double oldPrice = (Double) priceField.get(product);
        System.out.println("原價: " + oldPrice);
        
        priceField.set(product, 59900.0);
        System.out.println("新價: " + product.getPrice());
    }
}
```

執行結果:
```
屬性: id, 類型: Long
屬性: name, 類型: String
屬性: price, 類型: Double
原價: 65900.0
新價: 59900.0
```

### 呼叫方法

```java
public class MethodExample {
    
    public static void main(String[] args) throws Exception {
        Product product = new Product(1L, "iPad", 9900.0);
        Class<?> clazz = product.getClass();
        
        // 呼叫公開方法
        Method updateMethod = clazz.getMethod("updatePrice", Double.class);
        updateMethod.invoke(product, 8900.0);
        System.out.println("更新後價格: " + product.getPrice());
        
        // 呼叫私有方法
        Method privateMethod = clazz.getDeclaredMethod("internalMethod");
        privateMethod.setAccessible(true);
        privateMethod.invoke(product);
        
        // 獲取所有方法
        Method[] methods = clazz.getDeclaredMethods();
        System.out.println("\n所有方法:");
        for (Method method : methods) {
            System.out.println("- " + method.getName());
        }
    }
}
```

執行結果:
```
更新後價格: 8900.0
私有方法

所有方法:
- updatePrice
- internalMethod
- getId
- getName
- getPrice
- setId
- setName
- setPrice
```

## 註解 (Annotation)

註解是一種元資料,可以附加在類別、方法、屬性上,提供額外資訊。

### 內建註解

```java
public class BuiltInAnnotations {
    
    @Override
    public String toString() {
        return "BuiltInAnnotations";
    }
    
    @Deprecated
    public void oldMethod() {
        System.out.println("舊方法,不建議使用");
    }
    
    @SuppressWarnings("unchecked")
    public void suppressWarning() {
        List list = new ArrayList();
        list.add("test");
    }
}
```

### 自定義註解

```java
// 定義註解
@Target(ElementType.METHOD)  // 可用於方法
@Retention(RetentionPolicy.RUNTIME)  // 執行時期可取得
public @interface PerformanceLog {
    String value() default "";
    boolean enabled() default true;
}
```

```java
// 使用註解
public class OrderService {
    
    @PerformanceLog("建立訂單")
    public Order createOrder(OrderDTO dto) {
        // 建立訂單邏輯
        return new Order();
    }
    
    @PerformanceLog(value = "查詢訂單", enabled = true)
    public Order getOrder(Long id) {
        // 查詢訂單邏輯
        return new Order();
    }
}
```

### 讀取註解資訊

```java
public class AnnotationProcessor {
    
    public static void main(String[] args) throws Exception {
        Class<?> clazz = OrderService.class;
        
        for (Method method : clazz.getDeclaredMethods()) {
            if (method.isAnnotationPresent(PerformanceLog.class)) {
                PerformanceLog log = method.getAnnotation(PerformanceLog.class);
                
                System.out.println("方法: " + method.getName());
                System.out.println("描述: " + log.value());
                System.out.println("啟用: " + log.enabled());
                System.out.println("---");
            }
        }
    }
}
```

執行結果:
```
方法: createOrder
描述: 建立訂單
啟用: true
---
方法: getOrder
描述: 查詢訂單
啟用: true
---
```

## 實戰案例:簡易 ORM 框架

結合反射和註解,實作一個簡單的物件關聯映射。

### 定義註解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Table {
    String value();  // 資料表名稱
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    String value();  // 欄位名稱
    boolean primaryKey() default false;
}
```

### 實體類別

```java
@Table("products")
public class Product {
    
    @Column(value = "id", primaryKey = true)
    private Long id;
    
    @Column("name")
    private String name;
    
    @Column("price")
    private Double price;
    
    @Column("category")
    private String category;
    
    // constructors, getters, setters...
}
```

### SQL 產生器

```java
public class SqlGenerator {
    
    public static String generateSelectSql(Class<?> clazz, Object id) {
        // 獲取資料表名稱
        Table table = clazz.getAnnotation(Table.class);
        if (table == null) {
            throw new RuntimeException("缺少 @Table 註解");
        }
        String tableName = table.value();
        
        // 獲取主鍵欄位
        String primaryKey = null;
        for (Field field : clazz.getDeclaredFields()) {
            Column column = field.getAnnotation(Column.class);
            if (column != null && column.primaryKey()) {
                primaryKey = column.value();
                break;
            }
        }
        
        if (primaryKey == null) {
            throw new RuntimeException("找不到主鍵");
        }
        
        return String.format("SELECT * FROM %s WHERE %s = %s", 
                           tableName, primaryKey, id);
    }
    
    public static String generateInsertSql(Object obj) throws Exception {
        Class<?> clazz = obj.getClass();
        
        // 獲取資料表名稱
        Table table = clazz.getAnnotation(Table.class);
        String tableName = table.value();
        
        // 收集欄位和值
        List<String> columns = new ArrayList<>();
        List<String> values = new ArrayList<>();
        
        for (Field field : clazz.getDeclaredFields()) {
            Column column = field.getAnnotation(Column.class);
            if (column == null) continue;
            
            field.setAccessible(true);
            Object value = field.get(obj);
            
            if (value != null) {
                columns.add(column.value());
                
                if (value instanceof String) {
                    values.add("'" + value + "'");
                } else {
                    values.add(value.toString());
                }
            }
        }
        
        return String.format("INSERT INTO %s (%s) VALUES (%s)",
                           tableName,
                           String.join(", ", columns),
                           String.join(", ", values));
    }
    
    public static String generateUpdateSql(Object obj) throws Exception {
        Class<?> clazz = obj.getClass();
        
        Table table = clazz.getAnnotation(Table.class);
        String tableName = table.value();
        
        List<String> updates = new ArrayList<>();
        String whereClause = null;
        
        for (Field field : clazz.getDeclaredFields()) {
            Column column = field.getAnnotation(Column.class);
            if (column == null) continue;
            
            field.setAccessible(true);
            Object value = field.get(obj);
            
            if (value == null) continue;
            
            String valueStr = (value instanceof String) 
                ? "'" + value + "'" 
                : value.toString();
            
            if (column.primaryKey()) {
                whereClause = column.value() + " = " + valueStr;
            } else {
                updates.add(column.value() + " = " + valueStr);
            }
        }
        
        return String.format("UPDATE %s SET %s WHERE %s",
                           tableName,
                           String.join(", ", updates),
                           whereClause);
    }
}
```

### 測試 SQL 產生

```java
public class OrmTest {
    
    public static void main(String[] args) throws Exception {
        Product product = new Product();
        product.setId(1L);
        product.setName("iPhone 8");
        product.setPrice(21900.0);
        product.setCategory("電子產品");
        
        // 產生 SELECT
        String selectSql = SqlGenerator.generateSelectSql(Product.class, 1L);
        System.out.println("SELECT: " + selectSql);
        
        // 產生 INSERT
        String insertSql = SqlGenerator.generateInsertSql(product);
        System.out.println("INSERT: " + insertSql);
        
        // 產生 UPDATE
        product.setPrice(19900.0);
        String updateSql = SqlGenerator.generateUpdateSql(product);
        System.out.println("UPDATE: " + updateSql);
    }
}
```

執行結果:
```
SELECT: SELECT * FROM products WHERE id = 1
INSERT: INSERT INTO products (id, name, price, category) VALUES (1, 'iPhone 8', 21900.0, '電子產品')
UPDATE: UPDATE products SET name = 'iPhone 8', price = 19900.0, category = '電子產品' WHERE id = 1
```

## 效能監控切面

使用反射實現類似 Spring AOP 的功能:

```java
public class PerformanceProxy {
    
    public static <T> T createProxy(T target) {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) 
                    throws Throwable {
                    
                    long start = System.currentTimeMillis();
                    
                    try {
                        return method.invoke(target, args);
                    } finally {
                        long duration = System.currentTimeMillis() - start;
                        System.out.println(method.getName() + " 執行時間: " + duration + "ms");
                    }
                }
            }
        );
    }
}
```

使用範例:

```java
public interface ProductService {
    Product getProduct(Long id);
    void saveProduct(Product product);
}

public class ProductServiceImpl implements ProductService {
    
    @Override
    public Product getProduct(Long id) {
        // 模擬資料庫查詢
        try { Thread.sleep(100); } catch (Exception e) {}
        return new Product(id, "商品", 1000.0);
    }
    
    @Override
    public void saveProduct(Product product) {
        // 模擬資料庫寫入
        try { Thread.sleep(50); } catch (Exception e) {}
    }
}

public class ProxyTest {
    
    public static void main(String[] args) {
        ProductService service = new ProductServiceImpl();
        ProductService proxy = PerformanceProxy.createProxy(service);
        
        proxy.getProduct(1L);
        proxy.saveProduct(new Product());
    }
}
```

執行結果:
```
getProduct 執行時間: 102ms
saveProduct 執行時間: 51ms
```

## Spring 中的應用

Spring 框架大量使用反射和註解:

### 1. 依賴注入

```java
@Component
public class OrderService {
    
    @Autowired  // Spring 透過反射注入
    private ProductService productService;
}
```

Spring 實現原理簡化版:

```java
public class SimpleIocContainer {
    
    private Map<Class<?>, Object> beans = new HashMap<>();
    
    public void register(Class<?> clazz) throws Exception {
        Object instance = clazz.newInstance();
        
        // 處理 @Autowired 注入
        for (Field field : clazz.getDeclaredFields()) {
            if (field.isAnnotationPresent(Autowired.class)) {
                field.setAccessible(true);
                Object dependency = beans.get(field.getType());
                field.set(instance, dependency);
            }
        }
        
        beans.put(clazz, instance);
    }
    
    public <T> T getBean(Class<T> clazz) {
        return (T) beans.get(clazz);
    }
}
```

### 2. AOP 切面

```java
@Aspect
public class LogAspect {
    
    @Around("@annotation(PerformanceLog)")
    public Object logPerformance(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - start;
        
        System.out.println(pjp.getSignature() + " 耗時: " + duration + "ms");
        return result;
    }
}
```

## 反射的效能影響

```java
public class ReflectionPerformanceTest {
    
    public static void main(String[] args) throws Exception {
        Product product = new Product(1L, "Test", 100.0);
        int iterations = 1000000;
        
        // 直接呼叫
        long start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            product.getName();
        }
        long directTime = System.nanoTime() - start;
        
        // 反射呼叫
        Method method = Product.class.getMethod("getName");
        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            method.invoke(product);
        }
        long reflectionTime = System.nanoTime() - start;
        
        System.out.println("直接呼叫: " + directTime/1_000_000 + "ms");
        System.out.println("反射呼叫: " + reflectionTime/1_000_000 + "ms");
        System.out.println("慢了: " + (reflectionTime/directTime) + "x");
    }
}
```

執行結果:
```
直接呼叫: 3ms
反射呼叫: 48ms
慢了: 16x
```

反射確實較慢,但在框架初始化階段使用是可接受的。

## 注意事項

1. **效能考量**:反射比直接呼叫慢,避免在效能敏感的地方使用
2. **安全性**:setAccessible(true) 會破壞封裝,要謹慎使用
3. **型別安全**:反射會失去編譯期的型別檢查
4. **異常處理**:反射操作會拋出很多檢查型例外

## 小結

反射和註解是 Java 進階特性:
- **反射**:執行時期檢查和操作類別資訊
- **註解**:提供元資料,配合反射實現各種功能
- **應用場景**:框架開發、ORM、AOP、序列化等

Spring 框架正是建立在這些基礎上,理解它們有助於更好地使用框架。下週我們將學習 **Maven 專案管理**。
