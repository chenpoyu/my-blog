---
layout: post
title: "效能瓶頸分析：用 Tracing 找出慢查詢"
date: 2023-07-14 09:00:00 +0800
categories: [Observability, Tracing, Performance]
tags: [Performance Analysis, Bottleneck, Optimization, Database Query]
---

使用者抱怨：「你們的網站好慢！」

你打開 Prometheus，看到 P95 延遲是 3 秒。但**是哪個環節慢？**

今天我們用實際案例示範如何用 Tracing 分析效能瓶頸。

## 案例 1：N+1 查詢問題

### 症狀

```
API 回應時間：2-5 秒
資料庫 CPU：80%
```

### Trace 分析

在 Jaeger 搜尋慢請求：
- **Service**: `order-service`
- **Min Duration**: `2s`

看到 Timeline：

```
GET /api/orders (4.2s)
├── validate_user (45ms)
├── db: SELECT * FROM orders (50ms)
├── db: SELECT * FROM users WHERE id=1 (15ms)
├── db: SELECT * FROM users WHERE id=2 (18ms)
├── db: SELECT * FROM users WHERE id=3 (16ms)
├── ... (重複 100 次)
└── db: SELECT * FROM users WHERE id=100 (14ms)
```

**問題**：執行了 101 次資料庫查詢（1 次查訂單 + 100 次查使用者）。

### 問題程式碼

```java
@GetMapping("/api/orders")
public List<OrderDTO> getOrders() {
    List<Order> orders = orderRepository.findAll();  // 1 次查詢
    
    return orders.stream()
        .map(order -> {
            User user = userRepository.findById(order.getUserId()).get();  // N 次查詢
            return new OrderDTO(order, user);
        })
        .collect(Collectors.toList());
}
```

### 解決方案 1：JOIN 查詢

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUsers();
```

```java
@GetMapping("/api/orders")
public List<OrderDTO> getOrders() {
    List<Order> orders = orderRepository.findAllWithUsers();  // 1 次查詢
    
    return orders.stream()
        .map(order -> new OrderDTO(order, order.getUser()))
        .collect(Collectors.toList());
}
```

### 解決方案 2：Batch Fetch

```java
@GetMapping("/api/orders")
public List<OrderDTO> getOrders() {
    List<Order> orders = orderRepository.findAll();  // 1 次查詢
    
    Set<Long> userIds = orders.stream()
        .map(Order::getUserId)
        .collect(Collectors.toSet());
    
    Map<Long, User> users = userRepository.findByIdIn(userIds)  // 1 次查詢
        .stream()
        .collect(Collectors.toMap(User::getId, u -> u));
    
    return orders.stream()
        .map(order -> new OrderDTO(order, users.get(order.getUserId())))
        .collect(Collectors.toList());
}
```

### 結果

```
GET /api/orders (120ms)  ← 從 4.2s 降到 120ms
├── validate_user (45ms)
├── db: SELECT * FROM orders (50ms)
└── db: SELECT * FROM users WHERE id IN (...) (15ms)
```

**效能提升**：35 倍

## 案例 2：外部 API 逾時

### 症狀

```
API 偶爾回應時間 > 30 秒
錯誤：PaymentTimeoutException
```

### Trace 分析

```
POST /api/orders (30.5s)
├── validate_order (45ms)
├── check_inventory (120ms)
├── payment-service: POST /api/payment (30.2s)  ← 問題在這裡
│   └── stripe-api: POST /v1/charges (30.1s)  ← Stripe 很慢
└── db: INSERT orders (23ms)
```

**問題**：Stripe API 逾時（預設 30 秒）。

### 為什麼會慢？

檢查 Span 的 Tags：

```
stripe.request_id: req_abc123
stripe.error: timeout
stripe.retry_count: 3
```

原來 Stripe API **重試了 3 次**，每次等 10 秒。

### 解決方案 1：降低 Timeout

```java
@Service
public class PaymentService {
    private final RestTemplate restTemplate;
    
    public PaymentService() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(3000);  // 連線逾時 3 秒
        factory.setReadTimeout(5000);     // 讀取逾時 5 秒
        this.restTemplate = new RestTemplate(factory);
    }
    
    public Payment charge(ChargeRequest request) {
        try {
            return restTemplate.postForObject("https://api.stripe.com/v1/charges", request, Payment.class);
        } catch (ResourceAccessException e) {
            throw new PaymentTimeoutException("Stripe API timeout", e);
        }
    }
}
```

### 解決方案 2：非同步處理

```java
@PostMapping("/api/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
    // 先建立訂單（狀態：PENDING）
    Order order = orderService.createPendingOrder(request);
    
    // 非同步處理付款
    CompletableFuture.runAsync(() -> {
        try {
            Payment payment = paymentService.charge(request);
            orderService.confirmOrder(order.getId(), payment);
        } catch (Exception e) {
            orderService.cancelOrder(order.getId(), e.getMessage());
        }
    });
    
    return ResponseEntity.accepted()
        .body(new OrderResponse(order.getId(), "PENDING"));
}
```

### 結果

```
POST /api/orders (250ms)  ← 立即回應
├── validate_order (45ms)
├── check_inventory (120ms)
├── db: INSERT orders (23ms)
└── async: payment-service (背景處理)
```

使用者體驗改善：從等 30 秒變成立即回應。

## 案例 3：快取未命中

### 症狀

```
API 回應時間波動大：有時 50ms，有時 3s
```

### Trace 分析

**快的請求**：

```
GET /api/products/123 (52ms)
├── cache: GET product:123 (5ms)  ← 快取命中
└── return cached data (2ms)
```

**慢的請求**：

```
GET /api/products/456 (3.1s)
├── cache: GET product:456 (5ms)  ← 快取未命中
├── db: SELECT * FROM products WHERE id=456 (50ms)
├── external-api: GET /api/enrichment (2.8s)  ← 很慢
├── cache: SET product:456 (15ms)
└── return data (2ms)
```

**問題**：快取未命中時，要呼叫一個很慢的外部 API。

### 解決方案 1：增加快取命中率

```java
@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        Product product = productRepository.findById(id).orElseThrow();
        
        // 如果資料不完整，呼叫外部 API
        if (product.getEnrichmentData() == null) {
            EnrichmentData data = externalApiClient.getEnrichment(id);
            product.setEnrichmentData(data);
        }
        
        return product;
    }
}
```

**快取設定**：

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return RedisCacheManager.builder(redisConnectionFactory())
            .cacheDefaults(
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofHours(1))  // TTL 1 小時
                    .disableCachingNullValues()
            )
            .build();
    }
}
```

### 解決方案 2：預熱快取

```java
@Component
public class CacheWarmer {
    @Scheduled(fixedRate = 3600000)  // 每小時執行
    public void warmCache() {
        List<Long> popularProductIds = productRepository.findTop100ByOrderBySalesDesc()
            .stream()
            .map(Product::getId)
            .collect(Collectors.toList());
        
        for (Long id : popularProductIds) {
            productService.findById(id);  // 預先載入快取
        }
    }
}
```

### 結果

快取命中率從 60% 提升到 95%，平均回應時間從 1.5s 降到 200ms。

## 案例 4：鎖競爭

### 症狀

```
API 回應時間隨著併發增加而暴增
10 併發：100ms
100 併發：2s
1000 併發：30s
```

### Trace 分析

```
POST /api/orders (25.3s)
├── validate_order (45ms)
├── acquire_lock: order_sequence (25.1s)  ← 等鎖等了 25 秒
├── db: INSERT orders (23ms)
└── release_lock (1ms)
```

**問題**：所有請求都在搶同一個鎖。

### 問題程式碼

```java
@Service
public class OrderService {
    private final Object lock = new Object();
    
    public Order createOrder(CreateOrderRequest request) {
        synchronized (lock) {  // 全域鎖！
            Long orderNumber = getNextOrderNumber();
            Order order = new Order(orderNumber, request);
            return orderRepository.save(order);
        }
    }
}
```

### 解決方案：分散式 ID 生成器

```java
@Service
public class OrderService {
    private final SnowflakeIdGenerator idGenerator = new SnowflakeIdGenerator();
    
    public Order createOrder(CreateOrderRequest request) {
        Long orderNumber = idGenerator.nextId();  // 無鎖
        Order order = new Order(orderNumber, request);
        return orderRepository.save(order);
    }
}
```

或使用 Redis：

```java
@Service
public class OrderService {
    private final StringRedisTemplate redisTemplate;
    
    public Order createOrder(CreateOrderRequest request) {
        Long orderNumber = redisTemplate.opsForValue().increment("order:sequence");
        Order order = new Order(orderNumber, request);
        return orderRepository.save(order);
    }
}
```

### 結果

```
POST /api/orders (120ms)  ← 不管併發多少都是 120ms
├── validate_order (45ms)
├── generate_id (2ms)
├── db: INSERT orders (23ms)
└── ...
```

## 分析技巧

### 1. 尋找異常的 Span

在 Jaeger UI，按 **Duration** 排序，找出執行時間最長的 Span。

### 2. 比較快慢請求

選擇一個快的請求和一個慢的請求，點選 **Compare**，看看差異在哪裡。

### 3. 聚合分析

用 Prometheus 聚合 Trace 資料：

```promql
histogram_quantile(0.95, 
  rate(jaeger_traces_duration_seconds_bucket{service="order-service", operation="GET /api/orders"}[5m])
)
```

### 4. 檢查 Tag

重點關注：
- `error=true`：有錯誤的 Span
- `db.statement`：資料庫查詢語句
- `http.status_code >= 500`：伺服器錯誤

### 5. 檢查依賴關係

在 Jaeger 的 **System Architecture** 中，找出呼叫次數最多或錯誤率最高的依賴。

## 效能優化的優先級

### 1. 先優化慢的 Span（20% 的 Span 佔 80% 的時間）

```
Span A: 2000ms → 優先優化
Span B: 50ms
Span C: 30ms
Span D: 20ms
```

### 2. 優化高頻呼叫

```
Span A: 100ms，每秒 1000 次 → 優化後可省 100 秒/秒
Span B: 500ms，每秒 10 次 → 優化後可省 5 秒/秒
```

### 3. 優化資料庫查詢

通常 80% 的效能問題都是資料庫。

### 4. 優化外部 API 呼叫

考慮：
- 增加 timeout
- 加入快取
- 非同步處理
- 降級策略

---

**Tracing 讓你從「感覺」變成「知道」——你不再需要猜測哪裡慢，資料會告訴你。**
