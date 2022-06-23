---
layout: post
title: "Spring Cloud 微服務入門筆記"
date: 2017-10-20 14:40:00 +0800
categories: [設計模式, Java]
tags: [Spring Cloud, 微服務, 分散式系統]
---

隨著電商業務成長,單體應用變得越來越龐大、難以維護。微服務架構將應用拆分成多個小服務,每個服務獨立開發、部署、擴展。

> 本文使用版本: **Spring Cloud Dalston.SR4** + **Spring Boot 1.5.x** + **Java 8**

## 單體應用的問題

```
電商單體應用
├── 商品管理
├── 訂單管理
├── 用戶管理
├── 支付管理
├── 庫存管理
└── 物流管理
```

問題:
- **部署困難**:修改一個功能要重新部署整個應用
- **擴展受限**:無法針對特定功能擴展
- **技術債務**:很難引入新技術
- **團隊協作**:多個團隊修改同一個代碼庫容易衝突

## 微服務架構

```
├── product-service (商品服務)
├── order-service (訂單服務)
├── user-service (用戶服務)
├── payment-service (支付服務)
├── inventory-service (庫存服務)
└── logistics-service (物流服務)
```

優點:
- **獨立部署**:各服務獨立上線
- **彈性擴展**:熱門服務可多實例部署
- **技術異構**:不同服務可用不同技術棧
- **故障隔離**:一個服務掛掉不影響其他服務

## Spring Cloud 組件

### 1. 服務註冊與發現 - Eureka

#### Eureka Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false
```

#### Eureka Client

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: product-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

### 2. 服務呼叫 - Feign

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@FeignClient(name = "product-service")
public interface ProductClient {
    
    @GetMapping("/api/products/{id}")
    Product getProduct(@PathVariable Long id);
    
    @PostMapping("/api/products")
    Product createProduct(@RequestBody Product product);
    
    @PutMapping("/api/products/{id}/stock")
    void updateStock(@PathVariable Long id, @RequestParam int quantity);
}

@Service
public class OrderService {
    
    @Autowired
    private ProductClient productClient;
    
    public Order createOrder(OrderRequest request) {
        // 透過 Feign 呼叫 product-service
        Product product = productClient.getProduct(request.getProductId());
        
        if (product.getStock() < request.getQuantity()) {
            throw new InsufficientStockException();
        }
        
        // 建立訂單...
        Order order = new Order();
        
        // 扣除庫存
        productClient.updateStock(product.getId(), -request.getQuantity());
        
        return order;
    }
}
```

### 3. 負載均衡 - Ribbon

Feign 已內建 Ribbon,會自動負載均衡:

```yaml
product-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule  # 隨機
    # RoundRobinRule: 輪詢
    # WeightedResponseTimeRule: 根據回應時間加權
```

### 4. 斷路器 - Hystrix

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableCircuitBreaker
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

@Service
public class OrderService {
    
    @Autowired
    private ProductClient productClient;
    
    @HystrixCommand(fallbackMethod = "getProductFallback",
                    commandProperties = {
                        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", 
                                       value = "3000")
                    })
    public Product getProduct(Long id) {
        return productClient.getProduct(id);
    }
    
    // 降級方法
    private Product getProductFallback(Long id) {
        // 返回預設商品或快取資料
        Product product = new Product();
        product.setId(id);
        product.setName("商品暫時無法取得");
        return product;
    }
}
```

### 5. API 閘道 - Zuul

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableZuulProxy
public class ZuulGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulGatewayApplication.class, args);
    }
}
```

```yaml
server:
  port: 8080

zuul:
  routes:
    product-service:
      path: /api/products/**
      serviceId: product-service
      strip-prefix: false
      
    order-service:
      path: /api/orders/**
      serviceId: order-service
      strip-prefix: false
  
  # 敏感頭過濾
  sensitive-headers: Cookie,Set-Cookie
  
  # 超時設定
  host:
    connect-timeout-millis: 10000
    socket-timeout-millis: 60000
```

### 6. 配置中心 - Config Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mycompany/config-repo
          search-paths: '{application}'
```

## 實戰案例:電商微服務

### 商品服務

```java
@SpringBootApplication
@EnableEurekaClient
public class ProductServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProductServiceApplication.class, args);
    }
}

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id);
    }
    
    @PostMapping
    public Product createProduct(@RequestBody Product product) {
        return productService.save(product);
    }
    
    @PutMapping("/{id}/stock")
    public void updateStock(@PathVariable Long id, @RequestParam int quantity) {
        productService.updateStock(id, quantity);
    }
}
```

### 訂單服務

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
@EnableCircuitBreaker
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

@Service
public class OrderService {
    
    @Autowired
    private ProductClient productClient;
    
    @Autowired
    private UserClient userClient;
    
    @Autowired
    private PaymentClient paymentClient;
    
    @Transactional
    @HystrixCommand(fallbackMethod = "createOrderFallback")
    public Order createOrder(OrderRequest request) {
        // 1. 查詢商品資訊
        Product product = productClient.getProduct(request.getProductId());
        
        // 2. 查詢用戶資訊
        User user = userClient.getUser(request.getUserId());
        
        // 3. 檢查庫存
        if (product.getStock() < request.getQuantity()) {
            throw new InsufficientStockException();
        }
        
        // 4. 建立訂單
        Order order = new Order();
        order.setUser(user);
        order.setProduct(product);
        order.setQuantity(request.getQuantity());
        order.setTotalAmount(product.getPrice() * request.getQuantity());
        orderRepository.save(order);
        
        // 5. 處理支付
        Payment payment = paymentClient.processPayment(order.getTotalAmount());
        order.setPaymentId(payment.getId());
        
        // 6. 扣除庫存
        productClient.updateStock(product.getId(), -request.getQuantity());
        
        return order;
    }
    
    private Order createOrderFallback(OrderRequest request) {
        throw new ServiceUnavailableException("訂單服務暫時無法使用");
    }
}
```

## 分散式追蹤 - Sleuth

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

```yaml
spring:
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      probability: 1.0  # 100% 採樣
```

日誌會自動加上追蹤ID:
```
[product-service,a1b2c3d4,e5f6g7h8] 查詢商品: 123
[order-service,a1b2c3d4,i9j0k1l2] 建立訂單
```

## 分散式事務

### Saga 模式

```java
@Service
public class OrderSagaService {
    
    @Autowired
    private ProductClient productClient;
    
    @Autowired
    private PaymentClient paymentClient;
    
    public Order createOrderWithSaga(OrderRequest request) {
        Order order = new Order();
        
        try {
            // Step 1: 建立訂單
            order = orderRepository.save(order);
            
            // Step 2: 扣除庫存
            productClient.deductStock(request.getProductId(), request.getQuantity());
            
            // Step 3: 處理支付
            Payment payment = paymentClient.processPayment(order.getTotalAmount());
            order.setPaymentId(payment.getId());
            
            return order;
            
        } catch (Exception e) {
            // 補償操作
            compensate(order, request);
            throw e;
        }
    }
    
    private void compensate(Order order, OrderRequest request) {
        // 回滾庫存
        try {
            productClient.increaseStock(request.getProductId(), request.getQuantity());
        } catch (Exception e) {
            logger.error("補償失敗", e);
        }
        
        // 取消訂單
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
}
```

## 微服務實踐經驗

1. **服務拆分**:按業務領域拆分,避免過度拆分
2. **介面設計**:使用版本化 API,向下相容
3. **錯誤處理**:設計降級方案,避免級聯失敗
4. **監控告警**:完善的監控和日誌系統
5. **自動化**:CI/CD 自動化部署
6. **文件化**:完整的 API 文件

## 小結

Spring Cloud 提供了完整的微服務解決方案:
- **Eureka**:服務註冊與發現
- **Feign/Ribbon**:服務呼叫與負載均衡
- **Hystrix**:斷路器與降級
- **Zuul**:API 閘道統一入口
- **Config**:集中配置管理

微服務架構帶來靈活性,也增加了複雜度。下週我們將學習 **Docker 容器化**,看看如何部署微服務。
