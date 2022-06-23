---
layout: post
title: "API 文件撰寫筆記"
date: 2017-12-15 14:30:00 +0800
categories: [設計模式, Java]
tags: [API, 文件, Swagger, 開發]
---

開發RESTful API後,如何讓前端或其他團隊快速理解使用方式?手寫文件太麻煩且容易過時,這週來看看如何自動產生API文件。

## 為什麼需要API文件

電商系統有多個微服務,每個服務都提供API:
- 商品服務: `/api/products`
- 訂單服務: `/api/orders`
- 用戶服務: `/api/users`

前端工程師問:「這個API要傳什麼參數?回傳格式是什麼?」

手寫文件的問題:
- 程式改了,忘記更新文件
- 文件和實際代碼不一致
- 維護成本高

## Swagger 簡介

Swagger (現在稱為 OpenAPI) 可以從代碼自動產生API文件。

### 1. 加入依賴

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.7.0</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

### 2. Swagger 配置

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.controller"))
                .paths(PathSelectors.any())
                .build()
                .apiInfo(apiInfo());
    }
    
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("電商系統 API 文件")
                .description("商品、訂單相關API")
                .version("1.0")
                .contact(new Contact("開發團隊", "", "dev@example.com"))
                .build();
    }
}
```

### 3. Controller 註解

```java
@RestController
@RequestMapping("/api/products")
@Api(tags = "商品管理")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @ApiOperation("取得商品列表")
    @ApiResponses({
        @ApiResponse(code = 200, message = "成功"),
        @ApiResponse(code = 500, message = "系統錯誤")
    })
    @GetMapping
    public List<Product> getProducts(
        @ApiParam(value = "分類ID", required = false) 
        @RequestParam(required = false) Long categoryId,
        
        @ApiParam(value = "頁碼", defaultValue = "0") 
        @RequestParam(defaultValue = "0") int page,
        
        @ApiParam(value = "每頁筆數", defaultValue = "20") 
        @RequestParam(defaultValue = "20") int size
    ) {
        return productService.getProducts(categoryId, page, size);
    }
    
    @ApiOperation("取得商品詳情")
    @GetMapping("/{id}")
    public Product getProduct(
        @ApiParam(value = "商品ID", required = true) 
        @PathVariable Long id
    ) {
        return productService.getProduct(id);
    }
    
    @ApiOperation("新增商品")
    @PostMapping
    public Product createProduct(
        @ApiParam(value = "商品資訊", required = true) 
        @RequestBody @Valid ProductDTO dto
    ) {
        return productService.createProduct(dto);
    }
    
    @ApiOperation("更新商品")
    @PutMapping("/{id}")
    public Product updateProduct(
        @ApiParam(value = "商品ID", required = true) 
        @PathVariable Long id,
        
        @ApiParam(value = "商品資訊", required = true) 
        @RequestBody @Valid ProductDTO dto
    ) {
        return productService.updateProduct(id, dto);
    }
    
    @ApiOperation("刪除商品")
    @DeleteMapping("/{id}")
    public void deleteProduct(
        @ApiParam(value = "商品ID", required = true) 
        @PathVariable Long id
    ) {
        productService.deleteProduct(id);
    }
}
```

### 4. Model 註解

```java
@ApiModel("商品資訊")
public class ProductDTO {
    
    @ApiModelProperty(value = "商品名稱", required = true, example = "iPhone 7")
    @NotBlank(message = "商品名稱不能為空")
    private String name;
    
    @ApiModelProperty(value = "商品價格", required = true, example = "25000")
    @NotNull(message = "價格不能為空")
    @Min(value = 0, message = "價格必須大於0")
    private BigDecimal price;
    
    @ApiModelProperty(value = "商品描述", example = "Apple iPhone 7 128GB")
    private String description;
    
    @ApiModelProperty(value = "分類ID", required = true, example = "1")
    @NotNull(message = "分類不能為空")
    private Long categoryId;
    
    @ApiModelProperty(value = "庫存數量", example = "100")
    private Integer stock;
    
    // getters and setters
}
```

## 訪問 Swagger UI

啟動應用後,訪問: `http://localhost:8080/swagger-ui.html`

可以看到:
- 所有API端點
- 參數說明
- 回傳格式
- 可以直接測試API

## 實際範例

### 訂單 API 文件

```java
@RestController
@RequestMapping("/api/orders")
@Api(tags = "訂單管理")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @ApiOperation("建立訂單")
    @ApiImplicitParams({
        @ApiImplicitParam(name = "Authorization", value = "JWT Token", 
                          required = true, dataType = "string", paramType = "header")
    })
    @PostMapping
    public Order createOrder(@RequestBody @Valid OrderDTO dto) {
        return orderService.createOrder(dto);
    }
    
    @ApiOperation("查詢我的訂單")
    @GetMapping("/my")
    public List<Order> getMyOrders(
        @ApiParam("訂單狀態") @RequestParam(required = false) OrderStatus status,
        @ApiParam("開始日期") @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate startDate,
        @ApiParam("結束日期") @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate endDate
    ) {
        return orderService.getMyOrders(status, startDate, endDate);
    }
    
    @ApiOperation("取消訂單")
    @PutMapping("/{orderId}/cancel")
    public void cancelOrder(@ApiParam("訂單ID") @PathVariable Long orderId) {
        orderService.cancelOrder(orderId);
    }
}
```

### 訂單DTO

```java
@ApiModel("建立訂單請求")
public class OrderDTO {
    
    @ApiModelProperty(value = "商品列表", required = true)
    @NotEmpty(message = "訂單不能為空")
    @Valid
    private List<OrderItemDTO> items;
    
    @ApiModelProperty(value = "收貨地址ID", required = true)
    @NotNull(message = "收貨地址不能為空")
    private Long addressId;
    
    @ApiModelProperty(value = "備註")
    private String remark;
    
    // getters and setters
}

@ApiModel("訂單項目")
public class OrderItemDTO {
    
    @ApiModelProperty(value = "商品ID", required = true, example = "1")
    @NotNull(message = "商品ID不能為空")
    private Long productId;
    
    @ApiModelProperty(value = "數量", required = true, example = "2")
    @NotNull(message = "數量不能為空")
    @Min(value = 1, message = "數量至少為1")
    private Integer quantity;
    
    // getters and setters
}
```

## 分組管理

大型專案可能有很多API,可以分組顯示:

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    
    @Bean
    public Docket productApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("商品相關")
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.product"))
                .build();
    }
    
    @Bean
    public Docket orderApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("訂單相關")
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.order"))
                .build();
    }
    
    @Bean
    public Docket userApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("用戶相關")
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.user"))
                .build();
    }
}
```

## 隱藏內部API

有些API不想暴露在文件中:

```java
@ApiIgnore  // 整個Controller不顯示
@RestController
public class InternalController {
    // ...
}

// 或針對單一方法
@ApiOperation(value = "內部使用", hidden = true)
@GetMapping("/internal")
public void internalMethod() {
    // ...
}
```

## 自訂回應範例

```java
@ApiOperation("取得商品詳情")
@ApiResponses({
    @ApiResponse(code = 200, message = "成功", 
        examples = @Example(value = @ExampleProperty(
            mediaType = "application/json",
            value = "{\"id\":1,\"name\":\"iPhone 7\",\"price\":25000}"
        ))
    ),
    @ApiResponse(code = 404, message = "商品不存在")
})
@GetMapping("/{id}")
public Product getProduct(@PathVariable Long id) {
    return productService.getProduct(id);
}
```

## 加入認證

如果API需要登入,在Swagger UI中配置JWT:

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any())
                .build()
                .securitySchemes(Arrays.asList(apiKey()))
                .securityContexts(Arrays.asList(securityContext()));
    }
    
    private ApiKey apiKey() {
        return new ApiKey("JWT", "Authorization", "header");
    }
    
    private SecurityContext securityContext() {
        return SecurityContext.builder()
                .securityReferences(defaultAuth())
                .forPaths(PathSelectors.regex("/api/.*"))
                .build();
    }
    
    private List<SecurityReference> defaultAuth() {
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        return Arrays.asList(new SecurityReference("JWT", authorizationScopes));
    }
}
```

## 匯出文件

Swagger UI適合開發時使用,但如果要給外部合作夥伴,可以匯出靜態文件。

訪問JSON格式的API定義:
```
http://localhost:8080/v2/api-docs
```

可以用工具轉成其他格式:
- PDF
- Word
- Markdown

## 實用技巧

### 1. 列舉類型說明

```java
@ApiModel
public enum OrderStatus {
    @ApiModelProperty("待付款")
    PENDING,
    
    @ApiModelProperty("已付款")
    PAID,
    
    @ApiModelProperty("已出貨")
    SHIPPED,
    
    @ApiModelProperty("已完成")
    COMPLETED,
    
    @ApiModelProperty("已取消")
    CANCELLED
}
```

### 2. 分頁回應

```java
@ApiModel("分頁回應")
public class PageResult<T> {
    
    @ApiModelProperty("資料列表")
    private List<T> content;
    
    @ApiModelProperty("總筆數")
    private long totalElements;
    
    @ApiModelProperty("總頁數")
    private int totalPages;
    
    @ApiModelProperty("當前頁碼")
    private int currentPage;
    
    @ApiModelProperty("每頁筆數")
    private int pageSize;
    
    // getters and setters
}
```

### 3. 統一回應格式

```java
@ApiModel("API回應")
public class ApiResponse<T> {
    
    @ApiModelProperty("狀態碼")
    private int code;
    
    @ApiModelProperty("訊息")
    private String message;
    
    @ApiModelProperty("資料")
    private T data;
    
    @ApiModelProperty("時間戳")
    private Long timestamp;
    
    // getters and setters
}
```

## 其他選擇

除了Swagger,還有其他API文件工具:

### Spring REST Docs

優點:
- 測試驅動,文件不會過時
- 產生更專業的文件

缺點:
- 需要寫測試
- 不能在線測試API

### API Blueprint

用Markdown格式撰寫API文件,適合API優先設計。

### Postman

可以從Postman Collection匯出文件,適合已經在用Postman的團隊。

## 實踐建議

1. **重要API加詳細說明** - 不是每個欄位都要寫,重點說明即可
2. **提供範例** - example屬性讓人一看就懂
3. **保持更新** - 改API記得更新註解
4. **隱藏內部API** - 用@ApiIgnore隱藏不對外的API
5. **版本管理** - API有大改動時增加版本號

## 小結

Swagger讓API文件變簡單:
- 從代碼自動產生,減少維護成本
- 提供UI介面可以直接測試
- 支援JWT等認證機制
- 可以分組管理大型專案

有了API文件,前後端協作更順暢,新人也能快速上手。

下週我們將學習 **Maven 進階使用**,了解如何管理專案依賴和建構流程!
