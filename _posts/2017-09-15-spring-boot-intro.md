---
layout: post
title: "Spring Boot 入門筆記"
date: 2017-09-15 10:20:00 +0800
categories: [設計模式, Java]
tags: [Spring Boot, 快速開發, 自動配置]
---

經過前面幾週對 Spring 和 Tomcat 的學習(請參考 [Tomcat 架構](/posts/2017/09/08/tomcat-architecture/)),我們發現傳統 Spring 應用需要大量的 XML 配置或 Java 配置。Spring Boot 的出現徹底改變了這一切。

> 本文使用版本: **Spring Boot 1.5.x** + **Spring Framework 4.3.x** + **Java 8**

## 傳統 Spring 應用的痛點

建立一個簡單的 Spring MVC 應用需要:
1. 配置 DispatcherServlet
2. 配置 ViewResolver
3. 配置資料源
4. 配置事務管理器
5. 配置 Hibernate/JPA
6. 打包成 WAR 部署到 Tomcat
7. ...

光配置就要寫幾百行 XML 或 Java 代碼!

## Spring Boot 的哲學

**約定優於配置 (Convention over Configuration)**

只要遵循約定,大部分配置都可以省略:
- 預設配置涵蓋 80% 的場景
- 需要時才覆蓋預設值
- 內嵌伺服器,無需部署 WAR

## 第一個 Spring Boot 應用

### 1. 建立專案結構

```
my-shop/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/myshop/
│   │   │       ├── MyShopApplication.java
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       └── model/
│   │   └── resources/
│   │       ├── application.properties
│   │       └── static/
│   └── test/
├── pom.xml
```

### 2. Maven 依賴

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.0</version>
</parent>

<dependencies>
    <!-- Web 開發 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- 資料庫 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
</dependencies>
```

### 3. 啟動類

```java
@SpringBootApplication
public class MyShopApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(MyShopApplication.class, args);
    }
}
```

就這樣!一個 Web 應用就建立完成了!

### 4. 建立 Controller

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping
    public List<Product> getAllProducts() {
        return productService.findAll();
    }
    
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
    
    @PostMapping
    public Product createProduct(@RequestBody Product product) {
        return productService.save(product);
    }
}
```

### 5. 執行應用

```bash
mvn spring-boot:run

# 或打包成 jar
mvn package
java -jar target/my-shop-1.0.0.jar
```

訪問 `http://localhost:8080/api/products` 就能看到結果!

## 自動配置魔法

`@SpringBootApplication` 包含三個關鍵註解:

```java
@SpringBootConfiguration  // 等同於 @Configuration
@EnableAutoConfiguration  // 自動配置
@ComponentScan           // 元件掃描
public @interface SpringBootApplication {
}
```

### 自動配置原理

Spring Boot 會根據 classpath 中的 jar 自動配置:
- 發現 `spring-boot-starter-web` → 配置 Tomcat + Spring MVC
- 發現 `spring-boot-starter-data-jpa` → 配置 Hibernate + DataSource
- 發現 `mysql-connector-java` → 配置 MySQL 驅動

## 實戰案例:電商應用配置

### application.properties 配置

```properties
# 伺服器配置
server.port=8080
server.servlet.context-path=/api

# 資料庫配置
spring.datasource.url=jdbc:mysql://localhost:3306/ecommerce
spring.datasource.username=root
spring.datasource.password=secret
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# HikariCP 連線池配置
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000

# JPA 配置
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

# 日誌配置
logging.level.root=INFO
logging.level.com.example.myshop=DEBUG
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n
logging.file.name=logs/myshop.log

# JSON 配置
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8
spring.jackson.default-property-inclusion=non_null

# 檔案上傳配置
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB
```

### application.yml 格式(推薦)

```yaml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce
    username: root
    password: secret
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect

logging:
  level:
    root: INFO
    com.example.myshop: DEBUG
```

### 多環境配置

```
application.yml                  # 共用配置
application-dev.yml             # 開發環境
application-test.yml            # 測試環境
application-prod.yml            # 生產環境
```

```yaml
# application.yml
spring:
  profiles:
    active: dev  # 啟用開發環境
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce_dev
    
logging:
  level:
    root: DEBUG
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/ecommerce
    
logging:
  level:
    root: WARN
    
server:
  port: 80
```

啟動時指定環境:
```bash
java -jar myshop.jar --spring.profiles.active=prod
```

## 常用 Starter

```xml
<!-- Web 開發 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- JPA 資料存取 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- 驗證 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- 測試 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Actuator 監控 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## Actuator 健康檢查

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
```

訪問監控端點:
- `http://localhost:8080/actuator/health` - 健康狀態
- `http://localhost:8080/actuator/info` - 應用資訊
- `http://localhost:8080/actuator/metrics` - 指標

## 自訂配置類

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    // 跨域配置
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("*")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .maxAge(3600);
    }
    
    // 攔截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
            .addPathPatterns("/api/**");
    }
}
```

## 自訂 Starter

如果有重複的配置,可以封裝成自訂 Starter:

```java
@Configuration
@ConditionalOnClass(ProductService.class)
@EnableConfigurationProperties(ProductProperties.class)
public class ProductAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public ProductService productService(ProductProperties properties) {
        return new ProductService(properties);
    }
}
```

## 小結

Spring Boot 帶來的優勢:
- **快速開發**:幾分鐘建立可執行的應用
- **約定優於配置**:減少 80% 的配置代碼
- **內嵌容器**:不需要部署 WAR
- **生產就緒**:內建監控和健康檢查
- **微服務友善**:輕量級,易於容器化

掌握 Spring Boot,你的開發效率會大幅提升!下週我們將深入 **Hibernate 與 JPA**,看看如何優雅地處理物件關係映射。
