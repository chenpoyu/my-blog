---
layout: post
title: "Spring MVC 工作原理筆記"
date: 2017-12-29 16:20:00 +0800
categories: [設計模式, Java]
tags: [Spring MVC, DispatcherServlet, 工作流程]
---

使用 Spring MVC 開發 RESTful API 很方便,但你知道從請求進來到回應返回,中間經歷了什麼嗎?理解這個流程對除錯和優化很有幫助。

> 本文使用版本: **Spring Framework 4.3.x** + **Java 8**

## Spring MVC 架構

```
HTTP Request
    ↓
DispatcherServlet (前端控制器)
    ↓
HandlerMapping (處理器映射)
    ↓
Handler (Controller)
    ↓
HandlerAdapter (處理器適配器)
    ↓
ModelAndView
    ↓
ViewResolver (視圖解析器)
    ↓
View (視圖)
    ↓
HTTP Response
```

## 核心組件

### 1. DispatcherServlet

前端控制器,所有請求的入口。

**web.xml 配置** (Spring 4 傳統方式):

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

**Java Config 方式**:

```java
public class WebAppInitializer implements WebApplicationInitializer {
    
    @Override
    public void onStartup(ServletContext servletContext) {
        // 建立 Spring 容器
        AnnotationConfigWebApplicationContext context = 
            new AnnotationConfigWebApplicationContext();
        context.register(WebConfig.class);
        
        // 註冊 DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = 
            servletContext.addServlet("dispatcher", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

### 2. HandlerMapping

負責根據 URL 找到對應的 Controller。

```java
@Configuration
@EnableWebMvc
public class WebConfig {
    
    @Bean
    public RequestMappingHandlerMapping handlerMapping() {
        RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping();
        mapping.setOrder(0);
        return mapping;
    }
}
```

### 3. HandlerAdapter

執行 Controller 方法的適配器。

### 4. ViewResolver

解析視圖名稱為實際的 View 物件。

```java
@Bean
public ViewResolver viewResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix("/WEB-INF/views/");
    resolver.setSuffix(".jsp");
    return resolver;
}
```

## 請求處理流程

### 完整流程圖

```
1. 客戶端發送請求
   GET http://localhost:8080/api/products/1
   
2. DispatcherServlet 接收請求
   - 查詢 HandlerMapping
   
3. HandlerMapping 返回 HandlerExecutionChain
   - 包含 Handler(Controller) 和 Interceptors
   
4. DispatcherServlet 呼叫 HandlerAdapter
   
5. HandlerAdapter 執行 Handler
   - 參數綁定
   - 資料驗證
   - 執行業務邏輯
   
6. Handler 返回 ModelAndView
   
7. DispatcherServlet 呼叫 ViewResolver
   
8. ViewResolver 返回 View
   
9. View 渲染並返回
   
10. DispatcherServlet 將回應返回給客戶端
```

### 實例追蹤

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        System.out.println("Controller: 處理請求 GET /api/products/" + id);
        return productService.getProduct(id);
    }
}
```

當請求 `GET /api/products/1` 時:

```
[DispatcherServlet] 接收請求: GET /api/products/1
[HandlerMapping] 找到處理器: ProductController.getProduct
[HandlerAdapter] 準備執行...
[HandlerAdapter] 綁定參數: id = 1
[Controller] 處理請求 GET /api/products/1
[Service] 查詢商品 ID: 1
[HandlerAdapter] 處理返回值
[MessageConverter] 轉換為 JSON
[DispatcherServlet] 返回回應
```

## 攔截器 (Interceptor)

攔截器可以在請求處理前後加入邏輯。

### 自定義攔截器

```java
public class PerformanceInterceptor implements HandlerInterceptor {
    
    private ThreadLocal<Long> startTime = new ThreadLocal<>();
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        startTime.set(System.currentTimeMillis());
        System.out.println("請求開始: " + request.getRequestURI());
        return true;  // 返回 true 繼續處理,false 則中斷
    }
    
    @Override
    public void postHandle(HttpServletRequest request, 
                          HttpServletResponse response, 
                          Object handler, 
                          ModelAndView modelAndView) {
        System.out.println("Controller 執行完成");
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) {
        long duration = System.currentTimeMillis() - startTime.get();
        System.out.println("請求完成: " + request.getRequestURI() + 
                         ", 耗時: " + duration + "ms");
        startTime.remove();
    }
}
```

### 註冊攔截器

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new PerformanceInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

執行結果:

```
請求開始: /api/products/1
Controller: 處理請求 GET /api/products/1
Controller 執行完成
請求完成: /api/products/1, 耗時: 45ms
```

## 參數綁定

Spring MVC 支援多種參數綁定方式。

### 1. 路徑變數

```java
@GetMapping("/products/{id}")
public Product getProduct(@PathVariable Long id) {
    return productService.getProduct(id);
}

// 請求: GET /api/products/123
// id = 123
```

### 2. 請求參數

```java
@GetMapping("/products")
public List<Product> searchProducts(
    @RequestParam(required = false) String keyword,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size
) {
    return productService.search(keyword, page, size);
}

// 請求: GET /api/products?keyword=iPhone&page=0&size=20
// keyword = "iPhone", page = 0, size = 20
```

### 3. 請求體

```java
@PostMapping("/products")
public Product createProduct(@RequestBody ProductDTO dto) {
    return productService.create(dto);
}

// 請求: POST /api/products
// Content-Type: application/json
// Body: {"name":"iPhone 8","price":21900}
```

### 4. 表單綁定

```java
public class ProductForm {
    private String name;
    private Double price;
    private String category;
    // getters/setters
}

@PostMapping("/products/form")
public Product createFromForm(@ModelAttribute ProductForm form) {
    return productService.create(form);
}

// 請求: POST /api/products/form
// Content-Type: application/x-www-form-urlencoded
// Body: name=iPhone&price=21900&category=電子產品
```

### 5. Header 和 Cookie

```java
@GetMapping("/user/info")
public UserInfo getUserInfo(
    @RequestHeader("Authorization") String token,
    @CookieValue(value = "sessionId", required = false) String sessionId
) {
    // token = "Bearer xxx..."
    // sessionId = "abc123"
    return userService.getInfo(token);
}
```

## 返回值處理

### 1. 返回 JSON (@RestController)

```java
@RestController
public class ProductController {
    
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        // 自動轉換為 JSON
        return new Product(id, "iPhone 8", 21900.0);
    }
}

// 回應: {"id":1,"name":"iPhone 8","price":21900.0}
```

### 2. 返回視圖 (@Controller)

```java
@Controller
public class PageController {
    
    @GetMapping("/products")
    public String listProducts(Model model) {
        List<Product> products = productService.getAll();
        model.addAttribute("products", products);
        return "product-list";  // 解析為 /WEB-INF/views/product-list.jsp
    }
}
```

### 3. ResponseEntity

```java
@GetMapping("/products/{id}")
public ResponseEntity<Product> getProduct(@PathVariable Long id) {
    Product product = productService.getProduct(id);
    
    if (product == null) {
        return ResponseEntity.notFound().build();
    }
    
    return ResponseEntity.ok()
        .header("X-Product-Id", id.toString())
        .body(product);
}
```

### 4. @ResponseBody

```java
@Controller
public class ApiController {
    
    @GetMapping("/api/products/{id}")
    @ResponseBody  // 標記這個方法返回 JSON
    public Product getProduct(@PathVariable Long id) {
        return productService.getProduct(id);
    }
}
```

## 異常處理

### 1. @ExceptionHandler

```java
@RestController
public class ProductController {
    
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        Product product = productService.getProduct(id);
        if (product == null) {
            throw new ProductNotFoundException("商品不存在: " + id);
        }
        return product;
    }
    
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ProductNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

### 2. @ControllerAdvice (全域處理)

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ProductNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        ErrorResponse error = new ErrorResponse(500, "系統錯誤: " + ex.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## 內容協商

Spring MVC 支援根據請求返回不同格式。

```java
@RestController
public class ProductController {
    
    @GetMapping(value = "/products/{id}", 
                produces = {MediaType.APPLICATION_JSON_VALUE, 
                           MediaType.APPLICATION_XML_VALUE})
    public Product getProduct(@PathVariable Long id) {
        return productService.getProduct(id);
    }
}
```

請求:
```
GET /api/products/1
Accept: application/json
-> 返回 JSON

GET /api/products/1
Accept: application/xml
-> 返回 XML
```

## 跨域配置 (CORS)

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

或者使用註解:

```java
@RestController
@CrossOrigin(origins = "http://localhost:3000")
public class ProductController {
    // ...
}
```

## 靜態資源處理

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 映射 /static/** 到 /resources/static/
        registry.addResourceHandler("/static/**")
                .addResourceLocations("/resources/static/")
                .setCachePeriod(3600);
        
        // 映射 /images/** 到檔案系統路徑
        registry.addResourceHandler("/images/**")
                .addResourceLocations("file:/var/www/images/");
    }
}
```

## MessageConverter

負責 HTTP 訊息與 Java 物件的轉換。

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        // JSON 轉換器
        MappingJackson2HttpMessageConverter jsonConverter = 
            new MappingJackson2HttpMessageConverter();
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        jsonConverter.setObjectMapper(objectMapper);
        converters.add(jsonConverter);
        
        // String 轉換器
        StringHttpMessageConverter stringConverter = 
            new StringHttpMessageConverter(StandardCharsets.UTF_8);
        converters.add(stringConverter);
    }
}
```

## 完整範例:電商 API

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    // 建立訂單
    @PostMapping
    public ResponseEntity<Order> createOrder(
        @RequestBody @Valid OrderDTO dto,
        @RequestHeader("Authorization") String token
    ) {
        User user = authService.getUserFromToken(token);
        Order order = orderService.createOrder(user, dto);
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .header("Location", "/api/orders/" + order.getId())
            .body(order);
    }
    
    // 查詢訂單
    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(
        @PathVariable Long id,
        @RequestHeader("Authorization") String token
    ) {
        User user = authService.getUserFromToken(token);
        Order order = orderService.getOrder(id);
        
        // 檢查權限
        if (!order.getUserId().equals(user.getId())) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }
        
        return ResponseEntity.ok(order);
    }
    
    // 查詢訂單列表
    @GetMapping
    public ResponseEntity<Page<Order>> listOrders(
        @RequestHeader("Authorization") String token,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(required = false) String status
    ) {
        User user = authService.getUserFromToken(token);
        Page<Order> orders = orderService.getUserOrders(user.getId(), status, page, size);
        
        return ResponseEntity.ok(orders);
    }
    
    // 取消訂單
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> cancelOrder(
        @PathVariable Long id,
        @RequestHeader("Authorization") String token
    ) {
        User user = authService.getUserFromToken(token);
        orderService.cancelOrder(id, user.getId());
        
        return ResponseEntity.noContent().build();
    }
    
    // 異常處理
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, ex.getMessage()));
    }
    
    @ExceptionHandler(InsufficientStockException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientStock(InsufficientStockException ex) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(400, ex.getMessage()));
    }
}
```

## 效能監控

使用攔截器記錄請求效能:

```java
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(RequestLoggingInterceptor.class);
    private ThreadLocal<Long> startTime = new ThreadLocal<>();
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        startTime.set(System.currentTimeMillis());
        
        logger.info("請求: {} {}", request.getMethod(), request.getRequestURI());
        
        if (handler instanceof HandlerMethod) {
            HandlerMethod method = (HandlerMethod) handler;
            logger.info("處理器: {}.{}", 
                method.getBeanType().getSimpleName(),
                method.getMethod().getName());
        }
        
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) {
        long duration = System.currentTimeMillis() - startTime.get();
        
        logger.info("回應: {} {} - 狀態: {}, 耗時: {}ms",
            request.getMethod(),
            request.getRequestURI(),
            response.getStatus(),
            duration);
        
        if (duration > 1000) {
            logger.warn("慢請求警告: {} {}, 耗時: {}ms",
                request.getMethod(),
                request.getRequestURI(),
                duration);
        }
        
        startTime.remove();
    }
}
```

日誌輸出:

```
2017-12-29 16:25:30 INFO  請求: GET /api/products/1
2017-12-29 16:25:30 INFO  處理器: ProductController.getProduct
2017-12-29 16:25:30 INFO  回應: GET /api/products/1 - 狀態: 200, 耗時: 45ms
```

## 小結

Spring MVC 的核心流程:
1. **DispatcherServlet** - 前端控制器,統一入口
2. **HandlerMapping** - 請求映射,找到對應的 Controller
3. **HandlerAdapter** - 執行 Controller 方法
4. **MessageConverter** - 處理請求和回應的轉換
5. **Interceptor** - 攔截器處理橫切關注點
6. **ExceptionHandler** - 統一異常處理

理解這個流程有助於:
- 除錯問題時知道在哪個環節出錯
- 自定義攔截器、轉換器等組件
- 優化請求處理效能

2017 年的學習到此告一段落,期待 2018 年繼續深入學習!
