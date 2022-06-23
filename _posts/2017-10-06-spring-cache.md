---
layout: post
title: "Spring 快取使用筆記"
date: 2017-10-06 11:10:00 +0800
categories: [設計模式, Java]
tags: [Spring, Cache, Redis, 效能優化]
---

電商系統中有大量重複查詢:熱門商品、類別清單、使用者資訊...每次都查資料庫會拖慢效能。Spring Cache 讓快取變得超級簡單。

## 為什麼需要快取?

```java
// 沒有快取:每次都查資料庫
@GetMapping("/{id}")
public Product getProduct(@PathVariable Long id) {
    return productRepository.findById(id).orElse(null);  // 查詢資料庫
}
```

當商品詳情頁每秒被訪問 1000 次,資料庫壓力巨大!

## Spring Cache 抽象

Spring 提供統一的快取 API,支援多種快取實作:
- 本機快取:ConcurrentHashMap、Caffeine、Ehcache
- 分散式快取:Redis、Memcached

## 快速開始

### 1. 添加依賴

```xml
<!-- Redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### 2. 啟用快取

```java
@SpringBootApplication
@EnableCaching
public class MyShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyShopApplication.class, args);
    }
}
```

### 3. 使用註解

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    // 查詢時快取結果
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        System.out.println("查詢資料庫: " + id);
        return productRepository.findById(id).orElse(null);
    }
    
    // 更新時刪除快取
    @CacheEvict(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);
    }
    
    // 刪除時清空快取
    @CacheEvict(value = "products", allEntries = true)
    public void deleteAll() {
        productRepository.deleteAll();
    }
    
    // 更新時同步快取
    @CachePut(value = "products", key = "#result.id")
    public Product save(Product product) {
        return productRepository.save(product);
    }
}
```

## Redis 配置

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password:
    database: 0
    timeout: 5000ms
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 2000ms
        
  cache:
    type: redis
    redis:
      time-to-live: 3600000  # 1 小時
      cache-null-values: false
      key-prefix: "myshop:"
      use-key-prefix: true
```

### 自訂 Redis 配置

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        
        // 預設配置
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();
        
        // 針對不同快取設定不同過期時間
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // 商品快取: 1 小時
        cacheConfigurations.put("products",
            defaultConfig.entryTtl(Duration.ofHours(1)));
        
        // 類別快取: 24 小時 (很少變動)
        cacheConfigurations.put("categories",
            defaultConfig.entryTtl(Duration.ofHours(24)));
        
        // 熱門商品: 5 分鐘
        cacheConfigurations.put("hotProducts",
            defaultConfig.entryTtl(Duration.ofMinutes(5)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }
}
```

## 實戰案例:電商快取策略

### 場景一:商品詳情快取

```java
@Service
public class ProductService {
    
    // 基礎快取
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
    
    // 條件快取:只快取上架商品
    @Cacheable(value = "products", key = "#id", 
               condition = "#result != null and #result.status == 'ACTIVE'")
    public Product findByIdIfActive(Long id) {
        return productRepository.findById(id).orElse(null);
    }
    
    // 複雜的 key:多個參數
    @Cacheable(value = "productSearch", 
               key = "#category + ':' + #keyword + ':' + #page")
    public List<Product> search(String category, String keyword, int page) {
        return productRepository.search(category, keyword, page);
    }
}
```

### 場景二:庫存快取

```java
@Service
public class InventoryService {
    
    @Autowired
    private RedisTemplate<String, Integer> redisTemplate;
    
    // 將庫存載入到 Redis
    public void loadInventory(Long productId) {
        Integer stock = inventoryRepository.getStock(productId);
        String key = "inventory:" + productId;
        redisTemplate.opsForValue().set(key, stock, 1, TimeUnit.HOURS);
    }
    
    // 從 Redis 取得庫存
    public Integer getStock(Long productId) {
        String key = "inventory:" + productId;
        Integer stock = redisTemplate.opsForValue().get(key);
        
        if (stock == null) {
            // 快取未命中,查詢資料庫
            stock = inventoryRepository.getStock(productId);
            redisTemplate.opsForValue().set(key, stock, 1, TimeUnit.HOURS);
        }
        
        return stock;
    }
    
    // 扣除庫存(使用 Redis 原子操作)
    public boolean deductStock(Long productId, int quantity) {
        String key = "inventory:" + productId;
        Long newStock = redisTemplate.opsForValue().decrement(key, quantity);
        
        if (newStock < 0) {
            // 庫存不足,回滾
            redisTemplate.opsForValue().increment(key, quantity);
            return false;
        }
        
        // 非同步更新資料庫
        asyncUpdateDatabase(productId, quantity);
        
        return true;
    }
    
    @Async
    private void asyncUpdateDatabase(Long productId, int quantity) {
        inventoryRepository.deductStock(productId, quantity);
    }
}
```

### 場景三:使用者 Session 快取

```java
@Service
public class SessionService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // 儲存 Session
    public void saveSession(String sessionId, User user) {
        String key = "session:" + sessionId;
        String userJson = objectMapper.writeValueAsString(user);
        redisTemplate.opsForValue().set(key, userJson, 30, TimeUnit.MINUTES);
    }
    
    // 取得 Session
    public User getSession(String sessionId) {
        String key = "session:" + sessionId;
        String userJson = redisTemplate.opsForValue().get(key);
        
        if (userJson == null) {
            return null;
        }
        
        // 延長過期時間
        redisTemplate.expire(key, 30, TimeUnit.MINUTES);
        
        return objectMapper.readValue(userJson, User.class);
    }
    
    // 刪除 Session
    public void deleteSession(String sessionId) {
        String key = "session:" + sessionId;
        redisTemplate.delete(key);
    }
}
```

### 場景四:排行榜快取

```java
@Service
public class RankingService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 增加商品銷量分數
    public void incrementSales(Long productId, int quantity) {
        String key = "ranking:sales";
        redisTemplate.opsForZSet().incrementScore(key, productId.toString(), quantity);
    }
    
    // 取得熱銷商品排行
    @Cacheable(value = "hotProducts", key = "'top' + #limit")
    public List<ProductRanking> getTopSellingProducts(int limit) {
        String key = "ranking:sales";
        
        // 取得分數最高的前 N 個
        Set<ZSetOperations.TypedTuple<Object>> items = 
            redisTemplate.opsForZSet().reverseRangeWithScores(key, 0, limit - 1);
        
        List<ProductRanking> rankings = new ArrayList<>();
        int rank = 1;
        
        for (ZSetOperations.TypedTuple<Object> item : items) {
            Long productId = Long.parseLong(item.getValue().toString());
            Product product = productService.findById(productId);
            
            ProductRanking ranking = new ProductRanking();
            ranking.setRank(rank++);
            ranking.setProduct(product);
            ranking.setSales(item.getScore().intValue());
            
            rankings.add(ranking);
        }
        
        return rankings;
    }
}
```

## 快取更新策略

### 1. Cache-Aside (旁路快取)
最常用的模式,應用負責維護快取:
- 讀取:先查快取,未命中再查資料庫並更新快取
- 更新:先更新資料庫,再刪除快取

```java
public Product getProduct(Long id) {
    // 1. 查快取
    Product cached = cache.get(id);
    if (cached != null) {
        return cached;
    }
    
    // 2. 查資料庫
    Product product = db.findById(id);
    
    // 3. 更新快取
    cache.put(id, product);
    
    return product;
}

public void updateProduct(Product product) {
    // 1. 更新資料庫
    db.save(product);
    
    // 2. 刪除快取
    cache.evict(product.getId());
}
```

### 2. Write-Through (寫透快取)
更新資料時,同時更新快取和資料庫:

```java
@CachePut(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepository.save(product);
}
```

### 3. Write-Behind (寫後快取)
先更新快取,非同步更新資料庫:

```java
public void updateProduct(Product product) {
    // 1. 更新快取
    cache.put(product.getId(), product);
    
    // 2. 非同步更新資料庫
    executor.execute(() -> db.save(product));
}
```

## 快取常見問題

### 1. 快取穿透
查詢不存在的資料,快取無法生效,請求直達資料庫。

解決:快取空值或使用布隆過濾器

```java
@Cacheable(value = "products", key = "#id", unless = "#result == null")
public Product findById(Long id) {
    Product product = productRepository.findById(id).orElse(null);
    
    // 即使為 null 也快取一小段時間
    if (product == null) {
        // 可以快取一個特殊標記
        return Product.EMPTY;
    }
    
    return product;
}
```

### 2. 快取雪崩
大量快取同時過期,請求湧入資料庫。

解決:設定隨機過期時間

```java
Duration ttl = Duration.ofMinutes(30 + random.nextInt(10));  // 30-40 分鐘
```

### 3. 快取擊穿
熱點資料過期瞬間,大量請求同時查資料庫。

解決:使用鎖或永不過期

```java
public Product getProduct(Long id) {
    Product cached = cache.get(id);
    if (cached != null) {
        return cached;
    }
    
    synchronized (("product:" + id).intern()) {
        // 雙重檢查
        cached = cache.get(id);
        if (cached != null) {
            return cached;
        }
        
        Product product = db.findById(id);
        cache.put(id, product);
        return product;
    }
}
```

## 小結

Spring Cache 提供了:
- **註解驅動**:@Cacheable、@CacheEvict、@CachePut
- **多種實作**:本機或分散式快取
- **靈活配置**:不同快取不同策略
- **提升效能**:減少資料庫壓力

合理使用快取,你的應用效能會提升數倍!下週我們將學習 **Spring 非同步處理**,看看如何處理耗時任務。
