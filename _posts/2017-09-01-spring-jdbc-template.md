---
layout: post
title: "Spring JDBC Template 使用筆記"
date: 2017-09-01 16:10:00 +0800
categories: [設計模式, Java]
tags: [Spring, JDBC, 資料庫, 資料存取]
---

前面幾週我們建構了完整的 Spring MVC 應用（請參考 [Spring MVC](/posts/2017/08/25/spring-mvc-restful-api/)），但資料都還存在記憶體中。實際應用需要持久化到資料庫，這週我們來看 Spring 如何簡化資料庫操作。

## 傳統 JDBC 的痛苦

```java
public class ProductRepositoryImpl {
    
    public Product findById(String id) {
        Connection conn = null;
        PreparedStatement stmt = null;
        ResultSet rs = null;
        
        try {
            // 1. 取得連線
            conn = DriverManager.getConnection(url, username, password);
            
            // 2. 建立 Statement
            stmt = conn.prepareStatement("SELECT * FROM products WHERE id = ?");
            stmt.setString(1, id);
            
            // 3. 執行查詢
            rs = stmt.executeQuery();
            
            // 4. 處理結果
            if (rs.next()) {
                Product product = new Product();
                product.setId(rs.getString("id"));
                product.setName(rs.getString("name"));
                product.setPrice(rs.getDouble("price"));
                product.setCategory(rs.getString("category"));
                return product;
            }
            return null;
            
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            // 5. 關閉資源（容易忘記！）
            try {
                if (rs != null) rs.close();
                if (stmt != null) stmt.close();
                if (conn != null) conn.close();
            } catch (SQLException e) {
                // 吞掉例外？還是再拋出？
            }
        }
    }
}
```

問題太多了：
- 大量樣板代碼
- 容易忘記關閉資源導致連線洩漏
- 例外處理複雜
- 手動映射結果集

## Spring JDBC Template 的優雅

```java
@Repository
public class ProductRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public Product findById(String id) {
        String sql = "SELECT * FROM products WHERE id = ?";
        
        return jdbcTemplate.queryForObject(sql, 
            (rs, rowNum) -> {
                Product product = new Product();
                product.setId(rs.getString("id"));
                product.setName(rs.getString("name"));
                product.setPrice(rs.getDouble("price"));
                product.setCategory(rs.getString("category"));
                return product;
            }, 
            id
        );
    }
}
```

簡潔多了！Spring 會自動：
- 管理連線
- 處理例外
- 關閉資源

## 實戰案例：電商資料存取層

### 配置資料源

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/ecommerce");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 場景一：商品查詢操作

```java
@Repository
public class ProductRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // RowMapper 可以重複使用
    private final RowMapper<Product> productRowMapper = (rs, rowNum) -> {
        Product product = new Product();
        product.setId(rs.getString("id"));
        product.setName(rs.getString("name"));
        product.setPrice(rs.getDouble("price"));
        product.setCategory(rs.getString("category"));
        product.setDescription(rs.getString("description"));
        product.setStock(rs.getInt("stock"));
        product.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
        return product;
    };
    
    // 查詢單一商品
    public Optional<Product> findById(String id) {
        String sql = "SELECT * FROM products WHERE id = ?";
        
        try {
            Product product = jdbcTemplate.queryForObject(sql, productRowMapper, id);
            return Optional.ofNullable(product);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
    
    // 查詢所有商品
    public List<Product> findAll() {
        String sql = "SELECT * FROM products ORDER BY created_at DESC";
        return jdbcTemplate.query(sql, productRowMapper);
    }
    
    // 按類別查詢
    public List<Product> findByCategory(String category) {
        String sql = "SELECT * FROM products WHERE category = ?";
        return jdbcTemplate.query(sql, productRowMapper, category);
    }
    
    // 搜尋商品（支援模糊搜尋）
    public List<Product> search(String keyword) {
        String sql = "SELECT * FROM products " +
                    "WHERE name LIKE ? OR description LIKE ?";
        String pattern = "%" + keyword + "%";
        return jdbcTemplate.query(sql, productRowMapper, pattern, pattern);
    }
    
    // 查詢商品總數
    public int count() {
        String sql = "SELECT COUNT(*) FROM products";
        return jdbcTemplate.queryForObject(sql, Integer.class);
    }
    
    // 分頁查詢
    public List<Product> findAll(int page, int size) {
        String sql = "SELECT * FROM products " +
                    "ORDER BY created_at DESC " +
                    "LIMIT ? OFFSET ?";
        return jdbcTemplate.query(sql, productRowMapper, size, page * size);
    }
}
```

### 場景二：商品增刪改操作

```java
@Repository
public class ProductRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // 新增商品
    public Product save(Product product) {
        String sql = "INSERT INTO products (id, name, price, category, description, stock, created_at) " +
                    "VALUES (?, ?, ?, ?, ?, ?, ?)";
        
        jdbcTemplate.update(sql,
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getCategory(),
            product.getDescription(),
            product.getStock(),
            Timestamp.valueOf(LocalDateTime.now())
        );
        
        return product;
    }
    
    // 更新商品
    public int update(Product product) {
        String sql = "UPDATE products SET " +
                    "name = ?, price = ?, category = ?, " +
                    "description = ?, stock = ? " +
                    "WHERE id = ?";
        
        return jdbcTemplate.update(sql,
            product.getName(),
            product.getPrice(),
            product.getCategory(),
            product.getDescription(),
            product.getStock(),
            product.getId()
        );
    }
    
    // 刪除商品
    public int deleteById(String id) {
        String sql = "DELETE FROM products WHERE id = ?";
        return jdbcTemplate.update(sql, id);
    }
    
    // 批次新增商品
    public void batchInsert(List<Product> products) {
        String sql = "INSERT INTO products (id, name, price, category, description, stock, created_at) " +
                    "VALUES (?, ?, ?, ?, ?, ?, ?)";
        
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Product product = products.get(i);
                ps.setString(1, product.getId());
                ps.setString(2, product.getName());
                ps.setDouble(3, product.getPrice());
                ps.setString(4, product.getCategory());
                ps.setString(5, product.getDescription());
                ps.setInt(6, product.getStock());
                ps.setTimestamp(7, Timestamp.valueOf(LocalDateTime.now()));
            }
            
            @Override
            public int getBatchSize() {
                return products.size();
            }
        });
    }
    
    // 更新庫存
    public int updateStock(String productId, int quantity) {
        String sql = "UPDATE products SET stock = stock + ? WHERE id = ?";
        return jdbcTemplate.update(sql, quantity, productId);
    }
}
```

### 場景三：訂單資料操作

```java
@Repository
public class OrderRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // 建立訂單（需要插入多張表）
    @Transactional
    public Order createOrder(Order order) {
        // 1. 插入訂單主表
        String orderSql = "INSERT INTO orders " +
                         "(id, user_id, total_amount, status, created_at) " +
                         "VALUES (?, ?, ?, ?, ?)";
        
        jdbcTemplate.update(orderSql,
            order.getId(),
            order.getUserId(),
            order.getTotalAmount(),
            order.getStatus().name(),
            Timestamp.valueOf(LocalDateTime.now())
        );
        
        // 2. 插入訂單項目
        String itemSql = "INSERT INTO order_items " +
                        "(order_id, product_id, quantity, price) " +
                        "VALUES (?, ?, ?, ?)";
        
        for (OrderItem item : order.getItems()) {
            jdbcTemplate.update(itemSql,
                order.getId(),
                item.getProductId(),
                item.getQuantity(),
                item.getPrice()
            );
        }
        
        return order;
    }
    
    // 查詢使用者的訂單
    public List<Order> findByUserId(String userId) {
        String sql = "SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC";
        
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            Order order = new Order();
            order.setId(rs.getString("id"));
            order.setUserId(rs.getString("user_id"));
            order.setTotalAmount(rs.getDouble("total_amount"));
            order.setStatus(OrderStatus.valueOf(rs.getString("status")));
            order.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
            
            // 查詢訂單項目
            order.setItems(findOrderItems(order.getId()));
            
            return order;
        }, userId);
    }
    
    private List<OrderItem> findOrderItems(String orderId) {
        String sql = "SELECT oi.*, p.name as product_name " +
                    "FROM order_items oi " +
                    "JOIN products p ON oi.product_id = p.id " +
                    "WHERE oi.order_id = ?";
        
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            OrderItem item = new OrderItem();
            item.setProductId(rs.getString("product_id"));
            item.setProductName(rs.getString("product_name"));
            item.setQuantity(rs.getInt("quantity"));
            item.setPrice(rs.getDouble("price"));
            return item;
        }, orderId);
    }
    
    // 更新訂單狀態
    public int updateStatus(String orderId, OrderStatus status) {
        String sql = "UPDATE orders SET status = ? WHERE id = ?";
        return jdbcTemplate.update(sql, status.name(), orderId);
    }
}
```

### 場景四：複雜查詢與統計

```java
@Repository
public class OrderStatisticsRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // 查詢每日銷售額
    public List<DailySales> getDailySales(LocalDate startDate, LocalDate endDate) {
        String sql = "SELECT DATE(created_at) as date, " +
                    "       SUM(total_amount) as total_sales, " +
                    "       COUNT(*) as order_count " +
                    "FROM orders " +
                    "WHERE created_at BETWEEN ? AND ? " +
                    "  AND status = 'COMPLETED' " +
                    "GROUP BY DATE(created_at) " +
                    "ORDER BY date";
        
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            DailySales sales = new DailySales();
            sales.setDate(rs.getDate("date").toLocalDate());
            sales.setTotalSales(rs.getDouble("total_sales"));
            sales.setOrderCount(rs.getInt("order_count"));
            return sales;
        }, Date.valueOf(startDate), Date.valueOf(endDate));
    }
    
    // 查詢暢銷商品
    public List<ProductSales> getTopSellingProducts(int limit) {
        String sql = "SELECT p.id, p.name, " +
                    "       SUM(oi.quantity) as total_quantity, " +
                    "       SUM(oi.quantity * oi.price) as total_revenue " +
                    "FROM order_items oi " +
                    "JOIN products p ON oi.product_id = p.id " +
                    "JOIN orders o ON oi.order_id = o.id " +
                    "WHERE o.status = 'COMPLETED' " +
                    "GROUP BY p.id, p.name " +
                    "ORDER BY total_quantity DESC " +
                    "LIMIT ?";
        
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            ProductSales sales = new ProductSales();
            sales.setProductId(rs.getString("id"));
            sales.setProductName(rs.getString("name"));
            sales.setTotalQuantity(rs.getInt("total_quantity"));
            sales.setTotalRevenue(rs.getDouble("total_revenue"));
            return sales;
        }, limit);
    }
    
    // 查詢使用者消費統計
    public UserStatistics getUserStatistics(String userId) {
        String sql = "SELECT COUNT(*) as order_count, " +
                    "       SUM(total_amount) as total_spent, " +
                    "       AVG(total_amount) as avg_order_value " +
                    "FROM orders " +
                    "WHERE user_id = ? AND status = 'COMPLETED'";
        
        return jdbcTemplate.queryForObject(sql, (rs, rowNum) -> {
            UserStatistics stats = new UserStatistics();
            stats.setUserId(userId);
            stats.setOrderCount(rs.getInt("order_count"));
            stats.setTotalSpent(rs.getDouble("total_spent"));
            stats.setAvgOrderValue(rs.getDouble("avg_order_value"));
            return stats;
        }, userId);
    }
}
```

## 使用 NamedParameterJdbcTemplate

當參數很多時，使用具名參數更清晰：

```java
@Repository
public class ProductRepository {
    
    @Autowired
    private NamedParameterJdbcTemplate namedJdbcTemplate;
    
    public List<Product> search(ProductSearchCriteria criteria) {
        StringBuilder sql = new StringBuilder("SELECT * FROM products WHERE 1=1");
        Map<String, Object> params = new HashMap<>();
        
        if (criteria.getKeyword() != null) {
            sql.append(" AND name LIKE :keyword");
            params.put("keyword", "%" + criteria.getKeyword() + "%");
        }
        
        if (criteria.getCategory() != null) {
            sql.append(" AND category = :category");
            params.put("category", criteria.getCategory());
        }
        
        if (criteria.getMinPrice() != null) {
            sql.append(" AND price >= :minPrice");
            params.put("minPrice", criteria.getMinPrice());
        }
        
        if (criteria.getMaxPrice() != null) {
            sql.append(" AND price <= :maxPrice");
            params.put("maxPrice", criteria.getMaxPrice());
        }
        
        sql.append(" ORDER BY created_at DESC");
        sql.append(" LIMIT :limit OFFSET :offset");
        params.put("limit", criteria.getSize());
        params.put("offset", criteria.getPage() * criteria.getSize());
        
        return namedJdbcTemplate.query(sql.toString(), params, productRowMapper);
    }
}
```

## 交易管理

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    // 使用 @Transactional 自動管理交易
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. 建立訂單
        Order order = buildOrder(request);
        orderRepository.save(order);
        
        // 2. 扣除庫存
        for (OrderItem item : order.getItems()) {
            productRepository.updateStock(item.getProductId(), -item.getQuantity());
        }
        
        // 如果任何步驟失敗，整個交易會回滾
        return order;
    }
}
```

## 小結

Spring JDBC Template 提供了：
- **簡化的 API**：消除樣板代碼
- **自動資源管理**：不用擔心連線洩漏
- **統一的例外處理**：將 SQLException 轉換為 Spring 的例外體系
- **靈活的映射**：使用 RowMapper 輕鬆映射結果
- **交易支援**：使用 @Transactional 輕鬆管理交易

雖然 JDBC Template 很好用，但對於複雜的 ORM 需求還是不夠。下週我們將學習 **Hibernate 與 JPA**，看看如何用物件導向的方式操作資料庫。
