---
layout: post
title: "效能優化筆記"
date: 2017-11-17 13:20:00 +0800
categories: [設計模式, Java]
tags: [效能優化, JVM, 資料庫優化, 快取]
---

上週我們學習了測試策略(請參考 [測試策略](/posts/2017/11/10/testing-strategies/)),確保了程式品質。但系統上線後,如何應對高併發?如何讓查詢更快?效能優化是關鍵!

## 效能優化的層次

```
1. 程式碼層級:演算法、資料結構
2. JVM 層級:記憶體、垃圾回收
3. 資料庫層級:索引、查詢優化
4. 架構層級:快取、非同步、分散式
```

## 一、程式碼層級優化

### 1. 集合選擇

```java
// (失敗) 效能差
public List<Product> getProductsByIds(List<Long> ids) {
    List<Product> result = new ArrayList<>();
    for (Long id : ids) {
        for (Product product : allProducts) {  // O(n²)
            if (product.getId().equals(id)) {
                result.add(product);
                break;
            }
        }
    }
    return result;
}

// (成功) 效能好
public List<Product> getProductsByIds(List<Long> ids) {
    // 建立 Map 索引,O(n)
    Map<Long, Product> productMap = allProducts.stream()
        .collect(Collectors.toMap(Product::getId, p -> p));
    
    // 查詢 O(m),總複雜度 O(n+m)
    return ids.stream()
        .map(productMap::get)
        .filter(Objects::nonNull)
        .collect(Collectors.toList());
}
```

### 2. Stream vs. 傳統迴圈

```java
// 小數據量 (<1000):傳統迴圈較快
List<Product> expensive = new ArrayList<>();
for (Product product : products) {
    if (product.getPrice().compareTo(new BigDecimal("10000")) > 0) {
        expensive.add(product);
    }
}

// 大數據量或複雜操作:Stream 可讀性好,且可並行
List<Product> expensive = products.parallelStream()
    .filter(p -> p.getPrice().compareTo(new BigDecimal("10000")) > 0)
    .collect(Collectors.toList());
```

### 3. 避免重複計算

```java
// (失敗) 效能差:每次都計算
for (Order order : orders) {
    if (order.getTotalAmount().compareTo(getVipThreshold()) > 0) {
        // ...
    }
}

// (成功) 效能好:提前計算
BigDecimal vipThreshold = getVipThreshold();
for (Order order : orders) {
    if (order.getTotalAmount().compareTo(vipThreshold) > 0) {
        // ...
    }
}
```

### 4. StringBuilder vs. String

```java
// (失敗) 效能差:大量 String 物件
String result = "";
for (Product product : products) {
    result += product.getName() + ",";
}

// (成功) 效能好:可變字串
StringBuilder sb = new StringBuilder();
for (Product product : products) {
    sb.append(product.getName()).append(",");
}
String result = sb.toString();

// (成功) 更好:Stream API
String result = products.stream()
    .map(Product::getName)
    .collect(Collectors.joining(","));
```

## 二、JVM 調優

### 1. 記憶體配置

```bash
# 堆記憶體設定
java -Xms2g -Xmx4g \
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -jar app.jar

# -Xms:初始堆大小
# -Xmx:最大堆大小
# 建議:Xms = Xmx,避免動態調整
```

### 2. 垃圾回收器選擇

```bash
# G1GC (推薦,適合大堆記憶體)
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -jar app.jar

# ZGC (Java 8+,超低延遲)
java -XX:+UseZGC \
     -Xlog:gc \
     -jar app.jar

# Parallel GC (吞吐量優先)
java -XX:+UseParallelGC \
     -XX:ParallelGCThreads=8 \
     -jar app.jar
```

### 3. GC 日誌分析

```bash
# 開啟 GC 日誌
java -Xlog:gc*:file=gc.log:time,level,tags \
     -XX:+UseG1GC \
     -jar app.jar

# 分析 GC 日誌
# 使用工具:GCeasy (https://gceasy.io/)
```

### 4. 記憶體洩漏排查

```java
// (失敗) 記憶體洩漏:static 集合一直增長
public class CacheManager {
    private static Map<String, Object> cache = new HashMap<>();
    
    public static void put(String key, Object value) {
        cache.put(key, value);  // 永不清除!
    }
}

// (成功) 使用有過期時間的快取
@Configuration
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return CaffeineCacheManager()
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .maximumSize(10000)
            .build();
    }
}
```

工具排查:
```bash
# 1. 查看堆記憶體使用
jmap -heap <pid>

# 2. 產生堆轉儲
jmap -dump:format=b,file=heap.bin <pid>

# 3. 分析堆轉儲 (使用 Eclipse MAT)
```

## 三、資料庫優化

### 1. 索引優化

```sql
-- (失敗) 慢查詢:全表掃描
SELECT * FROM products WHERE name LIKE '%iPhone%';

-- (成功) 建立索引
CREATE INDEX idx_product_name ON products(name);

-- 查詢改寫
SELECT * FROM products WHERE name LIKE 'iPhone%';  -- 可以用到索引

-- 組合索引
CREATE INDEX idx_product_category_price 
ON products(category_id, price);

-- 查詢會用到索引
SELECT * FROM products 
WHERE category_id = 1 AND price > 10000;
```

### 2. 分頁優化

```java
// (失敗) 深分頁效能差
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 1000000, 20;  -- 需要掃描 1000020 行!

// (成功) 使用遊標分頁
SELECT * FROM orders 
WHERE id > last_id 
ORDER BY id 
LIMIT 20;
```

Spring Data JPA:
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // 深分頁優化
    @Query("SELECT o FROM Order o WHERE o.id > :lastId ORDER BY o.id")
    List<Order> findByIdGreaterThan(@Param("lastId") Long lastId, Pageable pageable);
}

// 使用
Long lastId = 0L;
while (true) {
    List<Order> orders = orderRepository.findByIdGreaterThan(
        lastId, 
        PageRequest.of(0, 100)
    );
    if (orders.isEmpty()) break;
    
    // 處理訂單
    processOrders(orders);
    
    lastId = orders.get(orders.size() - 1).getId();
}
```

### 3. N+1 查詢問題

```java
// (失敗) N+1 問題:1 次查詢訂單 + N 次查詢商品
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    for (OrderItem item : order.getItems()) {
        Product product = item.getProduct();  // 每次都查詢!
        System.out.println(product.getName());
    }
}

// (成功) 使用 JOIN FETCH
@Query("SELECT DISTINCT o FROM Order o " +
       "LEFT JOIN FETCH o.items i " +
       "LEFT JOIN FETCH i.product")
List<Order> findAllWithItems();

// (成功) 使用 EntityGraph
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findAll();
```

### 4. 批次操作

```java
// (失敗) 逐筆插入
for (Product product : products) {
    productRepository.save(product);  // 1000 次資料庫操作
}

// (成功) 批次插入
@Transactional
public void batchSave(List<Product> products) {
    int batchSize = 100;
    for (int i = 0; i < products.size(); i++) {
        productRepository.save(products.get(i));
        
        if (i % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}

// application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 100
        order_inserts: true
        order_updates: true
```

### 5. 連線池調優

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 10
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      
      # 連線測試
      connection-test-query: SELECT 1
      validation-timeout: 5000
```

## 四、快取優化

### 1. 多級快取

```
用戶端 → Nginx 靜態快取
       ↓
     API Gateway
       ↓
   本地快取 (Caffeine)
       ↓
   分散式快取 (Redis)
       ↓
     資料庫
```

### 2. 本地快取 + Redis

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    // 本地快取
    private Cache<Long, Product> localCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();
    
    public Product getProduct(Long id) {
        // 1. 查本地快取
        Product product = localCache.getIfPresent(id);
        if (product != null) {
            return product;
        }
        
        // 2. 查 Redis
        String key = "product:" + id;
        product = redisTemplate.opsForValue().get(key);
        if (product != null) {
            localCache.put(id, product);
            return product;
        }
        
        // 3. 查資料庫
        product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        
        // 4. 更新快取
        redisTemplate.opsForValue().set(key, product, 10, TimeUnit.MINUTES);
        localCache.put(id, product);
        
        return product;
    }
}
```

### 3. 快取預熱

```java
@Component
public class CacheWarmer implements ApplicationRunner {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    @Override
    public void run(ApplicationArguments args) {
        log.info("開始預熱快取...");
        
        // 預載熱門商品
        List<Product> hotProducts = productService.getHotProducts(100);
        for (Product product : hotProducts) {
            String key = "product:" + product.getId();
            redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS);
        }
        
        log.info("快取預熱完成,載入 {} 個商品", hotProducts.size());
    }
}
```

## 五、非同步處理

### 1. CompletableFuture 並行

```java
// (失敗) 序列執行:3 秒
Product product = productService.getProduct(id);      // 1 秒
List<Review> reviews = reviewService.getReviews(id);  // 1 秒
List<Product> related = productService.getRelated(id); // 1 秒

// (成功) 並行執行:1 秒
CompletableFuture<Product> productFuture = 
    CompletableFuture.supplyAsync(() -> productService.getProduct(id));
    
CompletableFuture<List<Review>> reviewsFuture = 
    CompletableFuture.supplyAsync(() -> reviewService.getReviews(id));
    
CompletableFuture<List<Product>> relatedFuture = 
    CompletableFuture.supplyAsync(() -> productService.getRelated(id));

// 等待所有完成
CompletableFuture.allOf(productFuture, reviewsFuture, relatedFuture).join();

Product product = productFuture.get();
List<Review> reviews = reviewsFuture.get();
List<Product> related = relatedFuture.get();
```

### 2. 訊息佇列非同步

```java
// 訂單服務
@Service
public class OrderService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Transactional
    public Order createOrder(OrderDTO dto) {
        // 1. 建立訂單 (同步)
        Order order = new Order();
        // ... 設定訂單資訊
        orderRepository.save(order);
        
        // 2. 發送訊息 (非同步)
        rabbitTemplate.convertAndSend("order.created", order);
        
        return order;
    }
}

// 庫存服務:非同步處理
@Component
public class InventoryListener {
    
    @RabbitListener(queues = "order.created")
    public void handleOrderCreated(Order order) {
        // 扣減庫存
        for (OrderItem item : order.getItems()) {
            inventoryService.decreaseStock(item.getProductId(), item.getQuantity());
        }
    }
}
```

## 六、實戰案例:商品列表優化

### 優化前

```java
@GetMapping("/products")
public List<ProductVO> getProducts(
    @RequestParam(required = false) Long categoryId,
    @RequestParam(defaultValue = "0") int page
) {
    // 1. 查詢商品 (N+1 問題)
    List<Product> products = productRepository.findByCategory(categoryId, page);
    
    // 2. 逐個查詢庫存
    for (Product product : products) {
        int stock = inventoryService.getStock(product.getId());  // N 次查詢
        product.setStock(stock);
    }
    
    // 3. 轉換 VO
    return products.stream()
        .map(this::toVO)
        .collect(Collectors.toList());
}
```

效能:200ms

### 優化後

```java
@GetMapping("/products")
@Cacheable(value = "productList", key = "#categoryId + '_' + #page")
public List<ProductVO> getProducts(
    @RequestParam(required = false) Long categoryId,
    @RequestParam(defaultValue = "0") int page
) {
    // 1. 使用 JOIN FETCH 避免 N+1
    List<Product> products = productRepository
        .findByCategoryWithDetails(categoryId, PageRequest.of(page, 20));
    
    // 2. 批次查詢庫存
    List<Long> productIds = products.stream()
        .map(Product::getId)
        .collect(Collectors.toList());
    Map<Long, Integer> stockMap = inventoryService.batchGetStock(productIds);
    
    // 3. 並行轉換 VO
    return products.parallelStream()
        .map(product -> {
            ProductVO vo = toVO(product);
            vo.setStock(stockMap.get(product.getId()));
            return vo;
        })
        .collect(Collectors.toList());
}
```

效能:20ms (提升 10 倍!)

優化點:
1. **快取**:相同請求直接返回
2. **JOIN FETCH**:解決 N+1 問題
3. **批次查詢**:一次查所有庫存
4. **並行處理**:Stream 並行轉換

## 效能監控

### 1. Spring Boot Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

### 2. 自訂 Metrics

```java
@Component
public class PerformanceMonitor {
    
    private final MeterRegistry meterRegistry;
    
    public PerformanceMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void recordOrderCreation(long duration) {
        Timer.builder("order.creation")
            .description("Order creation time")
            .register(meterRegistry)
            .record(duration, TimeUnit.MILLISECONDS);
    }
    
    public void incrementOrderCount() {
        Counter.builder("order.count")
            .description("Total orders created")
            .register(meterRegistry)
            .increment();
    }
}
```

### 3. 慢查詢監控

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE

spring:
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
        use_sql_comments: true
```

## 效能優化檢查清單

- [ ] 合適的資料結構和演算法
- [ ] JVM 記憶體和 GC 調優
- [ ] 資料庫索引和查詢優化
- [ ] 多級快取策略
- [ ] 非同步處理耗時操作
- [ ] 連線池合理配置
- [ ] 批次操作取代逐筆
- [ ] 避免 N+1 查詢
- [ ] 分頁深度控制
- [ ] 監控和日誌

## 小結

效能優化的黃金法則:
1. **先測量,再優化**:不要猜測瓶頸
2. **優化最慢的地方**:80/20 法則
3. **權衡**:效能 vs. 可讀性 vs. 維護性
4. **持續監控**:效能會退化

效能優化是一個持續的過程,不是一次性的工作!

下週我們將學習 **生產環境實踐經驗**,確保系統穩定運行!
