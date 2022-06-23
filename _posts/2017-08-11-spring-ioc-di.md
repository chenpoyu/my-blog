---
layout: post
title: "Spring IoC 與 DI 使用筆記"
date: 2017-08-11 11:45:00 +0800
categories: [設計模式, Java]
tags: [Spring, IoC, 依賴注入, DI]
---

前面幾週我們學習了 Java 的基礎特性，現在是時候進入企業級開發的核心框架：Spring。Spring 的核心理念是「控制反轉（IoC）」和「依賴注入（DI）」，這兩個概念徹底改變了我們組織代碼的方式。

> 本文使用版本: **Spring Framework 4.3.x** + **Java 8**

## 沒有 Spring 的痛苦

在電商系統中，各個元件之間有著複雜的依賴關係：

```java
public class OrderService {
    private ProductRepository productRepository;
    private InventoryService inventoryService;
    private PaymentService paymentService;
    private NotificationService notificationService;
    
    public OrderService() {
        // 😰 在建構子中建立所有依賴
        this.productRepository = new ProductRepositoryImpl();
        this.inventoryService = new InventoryServiceImpl(
            new InventoryRepository()
        );
        this.paymentService = new PaymentServiceImpl(
            new PaymentGateway(),
            new TransactionLogger()
        );
        this.notificationService = new NotificationServiceImpl(
            new EmailSender(),
            new SmsSender()
        );
    }
    
    public void createOrder(Order order) {
        // 業務邏輯...
    }
}
```

這種做法的問題：
1. **緊耦合**：`OrderService` 必須知道所有依賴的實作細節
2. **難以測試**：無法替換成 Mock 物件
3. **無法共享**：每次都要建立新實例
4. **依賴爆炸**：依賴的依賴也要自己建立

## 控制反轉（IoC）的理念

**傳統方式**：我們的代碼主動建立並管理依賴物件（我們控制一切）

**IoC 方式**：由框架（容器）負責建立並注入依賴物件（控制權反轉給框架）

> 好萊塢原則：Don't call us, we'll call you.

## 依賴注入（DI）的三種方式

### 方式一：建構子注入（推薦）

```java
@Service
public class OrderService {
    private final ProductRepository productRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    
    // Spring 會自動注入這些依賴
    @Autowired
    public OrderService(ProductRepository productRepository,
                       InventoryService inventoryService,
                       PaymentService paymentService) {
        this.productRepository = productRepository;
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
    }
    
    public void createOrder(Order order) {
        // 使用注入的依賴...
    }
}
```

優點：
- 依賴明確且不可變（`final`）
- 強制滿足依賴，避免 null
- 方便測試

### 方式二：Setter 注入

```java
@Service
public class OrderService {
    private ProductRepository productRepository;
    
    @Autowired
    public void setProductRepository(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
}
```

### 方式三：欄位注入（不推薦）

```java
@Service
public class OrderService {
    @Autowired
    private ProductRepository productRepository;
}
```

雖然簡潔，但有以下問題：
- 無法使用 `final`
- 難以測試（需要使用反射）
- 隱藏了依賴關係

## 實戰案例：重構電商系統

### 改造前：緊耦合的代碼

```java
public class OrderController {
    public void handleOrder(OrderRequest request) {
        // 到處建立物件
        ProductRepository productRepo = new ProductRepositoryImpl();
        Product product = productRepo.findById(request.getProductId());
        
        InventoryService inventory = new InventoryServiceImpl();
        if (!inventory.checkStock(product.getId(), request.getQuantity())) {
            throw new RuntimeException("庫存不足");
        }
        
        PaymentService payment = new PaymentServiceImpl();
        payment.process(request.getPaymentInfo());
        
        // ...
    }
}
```

### 改造後：使用 Spring IoC

#### 1. 定義介面

```java
public interface ProductRepository {
    Product findById(String id);
    List<Product> findAll();
    Product save(Product product);
}

public interface InventoryService {
    boolean checkStock(String productId, int quantity);
    void deductStock(String productId, int quantity);
}

public interface PaymentService {
    Payment process(PaymentInfo info);
}
```

#### 2. 實作類別加上 @Component

```java
@Repository  // 專門用於資料存取層
public class ProductRepositoryImpl implements ProductRepository {
    @Override
    public Product findById(String id) {
        // 實際的資料庫操作...
        return product;
    }
    
    @Override
    public List<Product> findAll() {
        // ...
    }
    
    @Override
    public Product save(Product product) {
        // ...
    }
}

@Service  // 用於業務邏輯層
public class InventoryServiceImpl implements InventoryService {
    private final InventoryRepository repository;
    
    @Autowired
    public InventoryServiceImpl(InventoryRepository repository) {
        this.repository = repository;
    }
    
    @Override
    public boolean checkStock(String productId, int quantity) {
        int available = repository.getStock(productId);
        return available >= quantity;
    }
    
    @Override
    public void deductStock(String productId, int quantity) {
        repository.updateStock(productId, -quantity);
    }
}
```

#### 3. Service 使用依賴注入

```java
@Service
public class OrderService {
    private final ProductRepository productRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    @Autowired
    public OrderService(ProductRepository productRepository,
                       InventoryService inventoryService,
                       PaymentService paymentService,
                       NotificationService notificationService) {
        this.productRepository = productRepository;
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
    
    public Order createOrder(OrderRequest request) {
        // 1. 查詢商品
        Product product = productRepository.findById(request.getProductId());
        if (product == null) {
            throw new ProductNotFoundException(request.getProductId());
        }
        
        // 2. 檢查庫存
        if (!inventoryService.checkStock(product.getId(), request.getQuantity())) {
            throw new InsufficientStockException(product.getId());
        }
        
        // 3. 建立訂單
        Order order = new Order();
        order.setProduct(product);
        order.setQuantity(request.getQuantity());
        order.setTotalAmount(product.getPrice() * request.getQuantity());
        
        // 4. 處理付款
        Payment payment = paymentService.process(request.getPaymentInfo());
        order.setPaymentId(payment.getId());
        
        // 5. 扣除庫存
        inventoryService.deductStock(product.getId(), request.getQuantity());
        
        // 6. 發送通知
        notificationService.sendOrderConfirmation(order);
        
        return order;
    }
}
```

#### 4. Controller 也使用依賴注入

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;
    
    @Autowired
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        try {
            Order order = orderService.createOrder(request);
            return ResponseEntity.ok(order);
        } catch (ProductNotFoundException e) {
            return ResponseEntity.notFound().build();
        } catch (InsufficientStockException e) {
            return ResponseEntity.badRequest().body(null);
        }
    }
}
```

## Spring Bean 的作用域

Spring 容器管理的物件稱為 Bean，它們有不同的作用域：

```java
// 1. Singleton（預設）：整個應用只有一個實例
@Service
@Scope("singleton")
public class ProductService {
    // 所有注入點都共享同一個實例
}

// 2. Prototype：每次注入都建立新實例
@Service
@Scope("prototype")
public class OrderProcessor {
    // 每次需要時都建立新實例
}

// 3. Request：每個 HTTP 請求一個實例（Web 應用）
@Component
@Scope("request")
public class ShoppingCart {
    // 每個使用者請求都有自己的購物車
}

// 4. Session：每個 HTTP Session 一個實例
@Component
@Scope("session")
public class UserPreferences {
    // 每個使用者 Session 保持偏好設定
}
```

## Spring 配置方式

### 方式一：XML 配置（舊式，不推薦）

```xml
<beans>
    <bean id="productRepository" 
          class="com.example.repository.ProductRepositoryImpl"/>
    
    <bean id="orderService" 
          class="com.example.service.OrderService">
        <constructor-arg ref="productRepository"/>
    </bean>
</beans>
```

### 方式二：Java 配置（推薦）

```java
@Configuration
public class AppConfig {
    
    @Bean
    public ProductRepository productRepository() {
        return new ProductRepositoryImpl();
    }
    
    @Bean
    public OrderService orderService(ProductRepository productRepository) {
        return new OrderService(productRepository);
    }
}
```

### 方式三：Component Scan + Annotation（最推薦）

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    // Spring 會自動掃描並註冊所有帶有 @Component、@Service、@Repository 的類別
}
```

## 單元測試變得簡單

有了依賴注入，測試變得超級容易：

```java
public class OrderServiceTest {
    
    @Test
    public void testCreateOrder_Success() {
        // 建立 Mock 物件
        ProductRepository mockProductRepo = mock(ProductRepository.class);
        InventoryService mockInventory = mock(InventoryService.class);
        PaymentService mockPayment = mock(PaymentService.class);
        NotificationService mockNotification = mock(NotificationService.class);
        
        // 設定 Mock 行為
        when(mockProductRepo.findById("P001"))
            .thenReturn(new Product("P001", "iPhone", 30000));
        when(mockInventory.checkStock("P001", 1))
            .thenReturn(true);
        
        // 注入 Mock 物件
        OrderService orderService = new OrderService(
            mockProductRepo,
            mockInventory,
            mockPayment,
            mockNotification
        );
        
        // 執行測試
        OrderRequest request = new OrderRequest("P001", 1);
        Order order = orderService.createOrder(request);
        
        // 驗證結果
        assertNotNull(order);
        assertEquals("P001", order.getProduct().getId());
        verify(mockInventory).deductStock("P001", 1);
    }
}
```

## Spring 核心概念總結

| 概念 | 說明 | 註解 |
|------|------|------|
| IoC | 控制反轉，由容器管理物件 | @Configuration |
| DI | 依賴注入，自動裝配依賴 | @Autowired |
| Bean | Spring 管理的物件 | @Component |
| Service | 業務邏輯層 | @Service |
| Repository | 資料存取層 | @Repository |

## 小結

Spring 的 IoC 容器和依賴注入機制帶來的好處：
1. **鬆耦合**：元件之間通過介面互動
2. **易測試**：可以輕鬆注入 Mock 物件
3. **易維護**：集中管理物件生命週期
4. **可重用**：Singleton 作用域共享實例

理解 IoC 和 DI 是掌握 Spring 的第一步。下週我們將深入探討 **Spring AOP**，看看如何優雅地處理橫切關注點。
