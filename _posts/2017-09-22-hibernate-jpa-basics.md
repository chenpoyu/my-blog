---
layout: post
title: "Hibernate 與 JPA 使用筆記"
date: 2017-09-22 13:45:00 +0800
categories: [設計模式, Java]
tags: [Hibernate, JPA, ORM, 資料持久化]
---

上週我們學習了 Spring Boot(請參考 [Spring Boot 入門](/posts/2017/09/15/spring-boot-intro/)),這週讓我們深入 ORM(Object-Relational Mapping),看看如何用物件導向的方式操作資料庫。

## 為什麼需要 ORM?

回想我們用 JDBC 查詢商品:

```java
String sql = "SELECT * FROM products WHERE id = ?";
ResultSet rs = statement.executeQuery(sql);
if (rs.next()) {
    Product product = new Product();
    product.setId(rs.getString("id"));
    product.setName(rs.getString("name"));
    product.setPrice(rs.getDouble("price"));
    // ...
}
```

每次都要手動映射,很繁瑣!

## JPA 與 Hibernate

- **JPA (Java Persistence API)**:Java 持久化規範
- **Hibernate**:JPA 的實作,最流行的 ORM 框架

## 實體類映射

### 基礎映射

```java
@Entity
@Table(name = "products")
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, length = 100)
    private String name;
    
    @Column(name = "price", nullable = false)
    private Double price;
    
    @Column(name = "category")
    private String category;
    
    @Column(name = "description", length = 500)
    private String description;
    
    @Column(name = "stock")
    private Integer stock;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
    
    // Getters and Setters
}
```

### 關聯映射

#### 一對多:訂單與訂單項目

```java
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", unique = true)
    private String orderNumber;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    @Column(name = "total_amount")
    private Double totalAmount;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status")
    private OrderStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // 便利方法
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
    
    // Getters and Setters
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;
    
    @Column(name = "quantity")
    private Integer quantity;
    
    @Column(name = "price")
    private Double price;
    
    // Getters and Setters
}
```

#### 多對多:商品與類別

```java
@Entity
@Table(name = "products")
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToMany
    @JoinTable(
        name = "product_categories",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "category_id")
    )
    private Set<Category> categories = new HashSet<>();
    
    // Getters and Setters
}

@Entity
@Table(name = "categories")
public class Category {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToMany(mappedBy = "categories")
    private Set<Product> products = new HashSet<>();
    
    // Getters and Setters
}
```

## Spring Data JPA Repository

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    // 方法名稱查詢
    List<Product> findByCategory(String category);
    
    List<Product> findByNameContaining(String keyword);
    
    List<Product> findByPriceBetween(Double min, Double max);
    
    List<Product> findTop10ByOrderByCreatedAtDesc();
    
    // @Query 查詢
    @Query("SELECT p FROM Product p WHERE p.price > :minPrice")
    List<Product> findExpensiveProducts(@Param("minPrice") Double minPrice);
    
    @Query("SELECT p FROM Product p JOIN p.categories c WHERE c.name = :category")
    List<Product> findByCategory Name(@Param("category") String category);
    
    // 原生 SQL
    @Query(value = "SELECT * FROM products WHERE stock < 10", nativeQuery = true)
    List<Product> findLowStockProducts();
    
    // 更新操作
    @Modifying
    @Query("UPDATE Product p SET p.stock = p.stock + :quantity WHERE p.id = :id")
    int updateStock(@Param("id") Long id, @Param("quantity") Integer quantity);
}
```

## 實戰案例:訂單服務

```java
@Service
@Transactional
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setOrderNumber(generateOrderNumber());
        order.setUser(getCurrentUser());
        order.setStatus(OrderStatus.PENDING);
        
        double totalAmount = 0.0;
        
        for (OrderItemRequest itemReq : request.getItems()) {
            Product product = productRepository.findById(itemReq.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(itemReq.getProductId()));
            
            if (product.getStock() < itemReq.getQuantity()) {
                throw new InsufficientStockException(product.getId());
            }
            
            OrderItem item = new OrderItem();
            item.setProduct(product);
            item.setQuantity(itemReq.getQuantity());
            item.setPrice(product.getPrice());
            
            order.addItem(item);
            
            totalAmount += product.getPrice() * itemReq.getQuantity();
            
            // 扣除庫存
            product.setStock(product.getStock() - itemReq.getQuantity());
        }
        
        order.setTotalAmount(totalAmount);
        
        return orderRepository.save(order);
    }
    
    @Transactional(readOnly = true)
    public Page<Order> getUserOrders(Long userId, Pageable pageable) {
        return orderRepository.findByUserId(userId, pageable);
    }
    
    public Order updateOrderStatus(Long orderId, OrderStatus status) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.setStatus(status);
        
        return orderRepository.save(order);
    }
}
```

## 複雜查詢:Specification

```java
public class ProductSpecifications {
    
    public static Specification<Product> hasCategory(String category) {
        return (root, query, cb) -> {
            if (category == null) return null;
            return cb.equal(root.get("category"), category);
        };
    }
    
    public static Specification<Product> nameLike(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null) return null;
            return cb.like(root.get("name"), "%" + keyword + "%");
        };
    }
    
    public static Specification<Product> priceBetween(Double min, Double max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("price"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("price"), min);
            return cb.between(root.get("price"), min, max);
        };
    }
}

// Repository
public interface ProductRepository extends JpaRepository<Product, Long>, 
                                           JpaSpecificationExecutor<Product> {
}

// Service
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    public Page<Product> search(ProductSearchCriteria criteria, Pageable pageable) {
        Specification<Product> spec = Specification
            .where(ProductSpecifications.hasCategory(criteria.getCategory()))
            .and(ProductSpecifications.nameLike(criteria.getKeyword()))
            .and(ProductSpecifications.priceBetween(criteria.getMinPrice(), criteria.getMaxPrice()));
        
        return productRepository.findAll(spec, pageable);
    }
}
```

## 效能優化

### 1. N+1 查詢問題

```java
//  產生 N+1 查詢
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    System.out.println(order.getUser().getName());  // 每次都查詢資料庫!
}

//  使用 JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();
```

### 2. 使用 DTO 投影

```java
// 只查詢需要的欄位
public interface ProductSummary {
    Long getId();
    String getName();
    Double getPrice();
}

public interface ProductRepository extends JpaRepository<Product, Long> {
    List<ProductSummary> findAllBy();
}
```

### 3. 批次操作

```java
@Repository
public class ProductRepository extends JpaRepository<Product, Long> {
    
    @Modifying
    @Query("UPDATE Product p SET p.stock = 0 WHERE p.id IN :ids")
    int clearStockBatch(@Param("ids") List<Long> ids);
}
```

### 4. 二級快取

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    // ...
}
```

## 小結

Hibernate/JPA 提供了:
- **物件導向的資料庫操作**:不用寫 SQL
- **自動映射**:物件與資料表自動轉換
- **關聯管理**:優雅處理表之間的關聯
- **Spring Data JPA**:進一步簡化 CRUD

掌握 ORM 是現代 Java 開發的必備技能!下週我們將學習 **Spring Security**,看看如何保護應用安全。
