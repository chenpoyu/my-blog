---
layout: post
title: "Spring MVC RESTful API 開發筆記"
date: 2017-08-25 09:50:00 +0800
categories: [設計模式, Java]
tags: [Spring, Spring MVC, RESTful API, Web 開發]
---

前兩週我們學習了 Spring 的 IoC 和 AOP（請參考 [Spring 框架入門](/posts/2017/08/11/spring-ioc-di/) 和 [Spring AOP](/posts/2017/08/18/spring-aop/)），現在是時候將這些知識應用到 Web 開發中了。Spring MVC 讓我們能夠輕鬆建構 RESTful API。

## 什麼是 RESTful API？

**REST（Representational State Transfer）** 是一種 API 設計風格，使用 HTTP 動詞來表達操作：

| HTTP 動詞 | 操作 | 範例 |
|-----------|------|------|
| GET | 查詢 | GET `/api/products/123` |
| POST | 新增 | POST `/api/products` |
| PUT | 完整更新 | PUT `/api/products/123` |
| PATCH | 部分更新 | PATCH `/api/products/123` |
| DELETE | 刪除 | DELETE `/api/products/123` |

## 第一個 Spring MVC Controller

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    // GET /api/products - 取得所有商品
    @GetMapping
    public List<Product> getAllProducts() {
        return productService.findAll();
    }
    
    // GET /api/products/123 - 取得單一商品
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable String id) {
        return productService.findById(id);
    }
    
    // POST /api/products - 建立商品
    @PostMapping
    public Product createProduct(@RequestBody ProductRequest request) {
        return productService.create(request);
    }
    
    // PUT /api/products/123 - 更新商品
    @PutMapping("/{id}")
    public Product updateProduct(@PathVariable String id, 
                                @RequestBody ProductRequest request) {
        return productService.update(id, request);
    }
    
    // DELETE /api/products/123 - 刪除商品
    @DeleteMapping("/{id}")
    public void deleteProduct(@PathVariable String id) {
        productService.delete(id);
    }
}
```

## 實戰案例：電商 API 完整實作

### 場景一：商品管理 API

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    // 查詢商品（支援分頁與搜尋）
    @GetMapping
    public ResponseEntity<PageResponse<Product>> getProducts(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        ProductSearchCriteria criteria = ProductSearchCriteria.builder()
            .keyword(keyword)
            .category(category)
            .page(page)
            .size(size)
            .build();
        
        PageResponse<Product> result = productService.search(criteria);
        return ResponseEntity.ok(result);
    }
    
    // 取得單一商品
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable String id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 建立商品
    @PostMapping
    public ResponseEntity<Product> createProduct(@Valid @RequestBody ProductRequest request) {
        Product product = productService.create(request);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(product.getId())
            .toUri();
        
        return ResponseEntity.created(location).body(product);
    }
    
    // 更新商品
    @PutMapping("/{id}")
    public ResponseEntity<Product> updateProduct(
            @PathVariable String id,
            @Valid @RequestBody ProductRequest request) {
        
        return productService.update(id, request)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 部分更新商品（例如只更新價格）
    @PatchMapping("/{id}/price")
    public ResponseEntity<Product> updatePrice(
            @PathVariable String id,
            @RequestBody @Valid PriceUpdateRequest request) {
        
        return productService.updatePrice(id, request.getPrice())
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 刪除商品
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable String id) {
        boolean deleted = productService.delete(id);
        return deleted ? ResponseEntity.noContent().build() 
                      : ResponseEntity.notFound().build();
    }
}
```

### 場景二：訂單管理 API

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    // 建立訂單
    @PostMapping
    public ResponseEntity<ApiResponse<Order>> createOrder(
            @Valid @RequestBody OrderRequest request) {
        
        try {
            Order order = orderService.createOrder(request);
            return ResponseEntity.ok(ApiResponse.success(order));
            
        } catch (InsufficientStockException e) {
            return ResponseEntity.badRequest()
                .body(ApiResponse.error("庫存不足", "INSUFFICIENT_STOCK"));
                
        } catch (PaymentFailedException e) {
            return ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED)
                .body(ApiResponse.error("付款失敗", "PAYMENT_FAILED"));
        }
    }
    
    // 查詢使用者的訂單
    @GetMapping("/my-orders")
    public ResponseEntity<List<Order>> getMyOrders(
            @AuthenticationPrincipal User currentUser,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        List<Order> orders = orderService.findByUser(currentUser.getId(), page, size);
        return ResponseEntity.ok(orders);
    }
    
    // 查詢訂單詳情
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDetail> getOrderDetail(
            @PathVariable String orderId,
            @AuthenticationPrincipal User currentUser) {
        
        return orderService.findById(orderId)
            .filter(order -> order.getUserId().equals(currentUser.getId()))
            .map(order -> orderService.getOrderDetail(order))
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 取消訂單
    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<ApiResponse<Void>> cancelOrder(
            @PathVariable String orderId,
            @AuthenticationPrincipal User currentUser) {
        
        try {
            orderService.cancelOrder(orderId, currentUser.getId());
            return ResponseEntity.ok(ApiResponse.success(null));
            
        } catch (OrderNotFoundException e) {
            return ResponseEntity.notFound().build();
            
        } catch (OrderCannotBeCancelledException e) {
            return ResponseEntity.badRequest()
                .body(ApiResponse.error("訂單無法取消", "CANNOT_CANCEL"));
        }
    }
}
```

### 場景三：購物車 API

```java
@RestController
@RequestMapping("/api/cart")
public class CartController {
    
    @Autowired
    private CartService cartService;
    
    // 查看購物車
    @GetMapping
    public ResponseEntity<Cart> getCart(@AuthenticationPrincipal User currentUser) {
        Cart cart = cartService.getCart(currentUser.getId());
        return ResponseEntity.ok(cart);
    }
    
    // 加入商品到購物車
    @PostMapping("/items")
    public ResponseEntity<Cart> addItem(
            @AuthenticationPrincipal User currentUser,
            @Valid @RequestBody AddToCartRequest request) {
        
        Cart cart = cartService.addItem(
            currentUser.getId(),
            request.getProductId(),
            request.getQuantity()
        );
        
        return ResponseEntity.ok(cart);
    }
    
    // 更新購物車項目數量
    @PutMapping("/items/{productId}")
    public ResponseEntity<Cart> updateItemQuantity(
            @AuthenticationPrincipal User currentUser,
            @PathVariable String productId,
            @RequestBody @Valid QuantityUpdateRequest request) {
        
        Cart cart = cartService.updateQuantity(
            currentUser.getId(),
            productId,
            request.getQuantity()
        );
        
        return ResponseEntity.ok(cart);
    }
    
    // 移除購物車項目
    @DeleteMapping("/items/{productId}")
    public ResponseEntity<Cart> removeItem(
            @AuthenticationPrincipal User currentUser,
            @PathVariable String productId) {
        
        Cart cart = cartService.removeItem(currentUser.getId(), productId);
        return ResponseEntity.ok(cart);
    }
    
    // 清空購物車
    @DeleteMapping
    public ResponseEntity<Void> clearCart(@AuthenticationPrincipal User currentUser) {
        cartService.clear(currentUser.getId());
        return ResponseEntity.noContent().build();
    }
}
```

## 請求參數處理

### 1. 路徑變數 @PathVariable

```java
// GET /api/products/123
@GetMapping("/{id}")
public Product getProduct(@PathVariable String id) {
    return productService.findById(id);
}

// GET /api/orders/ORD-001/items/ITEM-001
@GetMapping("/{orderId}/items/{itemId}")
public OrderItem getOrderItem(@PathVariable String orderId, 
                              @PathVariable String itemId) {
    return orderService.findItem(orderId, itemId);
}
```

### 2. 查詢參數 @RequestParam

```java
// GET /api/products?keyword=iPhone&minPrice=1000&maxPrice=50000
@GetMapping
public List<Product> searchProducts(
        @RequestParam String keyword,
        @RequestParam(required = false) Double minPrice,
        @RequestParam(required = false) Double maxPrice,
        @RequestParam(defaultValue = "0") int page) {
    
    return productService.search(keyword, minPrice, maxPrice, page);
}
```

### 3. 請求體 @RequestBody

```java
// POST /api/products
// Content-Type: application/json
// Body: { "name": "iPhone", "price": 30000 }
@PostMapping
public Product createProduct(@RequestBody ProductRequest request) {
    return productService.create(request);
}
```

### 4. 表頭 @RequestHeader

```java
@GetMapping("/{id}")
public Product getProduct(@PathVariable String id,
                         @RequestHeader("Accept-Language") String language) {
    return productService.findById(id, language);
}
```

## 參數驗證

使用 Bean Validation 進行參數驗證：

```java
public class ProductRequest {
    
    @NotBlank(message = "商品名稱不得為空")
    @Size(min = 1, max = 100, message = "商品名稱長度需在 1-100 之間")
    private String name;
    
    @NotNull(message = "價格不得為空")
    @Min(value = 0, message = "價格不得小於 0")
    private Double price;
    
    @NotBlank(message = "類別不得為空")
    private String category;
    
    @Size(max = 500, message = "描述長度不得超過 500")
    private String description;
    
    @Min(value = 0, message = "庫存不得小於 0")
    private Integer stock;
    
    // Getters and Setters
}

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @PostMapping
    public ResponseEntity<Product> createProduct(@Valid @RequestBody ProductRequest request) {
        // 如果驗證失敗，Spring 會自動回傳 400 Bad Request
        Product product = productService.create(request);
        return ResponseEntity.ok(product);
    }
}
```

## 統一例外處理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    // 處理資源找不到
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException e) {
        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .error("NOT_FOUND")
            .message(e.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    // 處理業務邏輯例外
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error(e.getErrorCode())
            .message(e.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.badRequest().body(error);
    }
    
    // 處理參數驗證失敗
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException e) {
        
        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(error -> 
            errors.put(error.getField(), error.getDefaultMessage())
        );
        
        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("VALIDATION_FAILED")
            .message("參數驗證失敗")
            .details(errors)
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.badRequest().body(error);
    }
    
    // 處理未預期的例外
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        logger.error("未預期的例外", e);
        
        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("INTERNAL_ERROR")
            .message("系統錯誤，請稍後再試")
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## CORS 設定

允許前端跨域存取 API：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000", "https://myshop.com")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

或使用註解：

```java
@RestController
@RequestMapping("/api/products")
@CrossOrigin(origins = "http://localhost:3000")
public class ProductController {
    // ...
}
```

## HTTP 狀態碼實踐經驗

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    // 200 OK：查詢成功
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable String id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());  // 404 Not Found
    }
    
    // 201 Created：建立成功
    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody ProductRequest request) {
        Product product = productService.create(request);
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(product);
    }
    
    // 204 No Content：刪除成功
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable String id) {
        productService.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    // 400 Bad Request：請求參數錯誤
    @PostMapping
    public ResponseEntity<?> createProduct(@Valid @RequestBody ProductRequest request) {
        // Spring 自動處理驗證失敗，回傳 400
    }
    
    // 401 Unauthorized：未登入
    // 403 Forbidden：沒有權限
    // 404 Not Found：資源不存在
    // 409 Conflict：資源衝突（例如重複建立）
    // 500 Internal Server Error：伺服器錯誤
}
```

## API 版本控制

```java
// 方式一：路徑版本
@RestController
@RequestMapping("/api/v1/products")
public class ProductV1Controller {
    // 舊版 API
}

@RestController
@RequestMapping("/api/v2/products")
public class ProductV2Controller {
    // 新版 API
}

// 方式二：使用 Header
@GetMapping(headers = "API-Version=1")
public List<Product> getProductsV1() {
    // 舊版
}

@GetMapping(headers = "API-Version=2")
public List<Product> getProductsV2() {
    // 新版
}
```

## 小結

Spring MVC 提供了完整的 Web 開發支援：
- **@RestController**：建構 RESTful API
- **參數綁定**：自動將請求參數轉換為 Java 物件
- **參數驗證**：使用 Bean Validation 驗證輸入
- **例外處理**：統一處理例外並回傳適當的 HTTP 狀態碼
- **CORS 支援**：解決跨域問題

掌握 Spring MVC，你就能建構出專業的 RESTful API。下週我們將進入 **Spring Boot**，看看它如何簡化 Spring 應用的開發與部署。
