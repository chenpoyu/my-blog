---
layout: post
title: "測試策略筆記"
date: 2017-11-10 10:50:00 +0800
categories: [設計模式, Java]
tags: [測試, JUnit, Mockito, 整合測試]
---

前面我們學習了從 Java 基礎到微服務部署的完整技術棧,但如何確保程式品質?完善的測試策略是關鍵!

## 測試金字塔

```
        /\
       /E2E\       少量端到端測試
      /------\
     /整合測試 \    適量整合測試
    /----------\
   /  單元測試  \   大量單元測試
  /--------------\
```

**原則**:越底層測試越多,執行越快,成本越低

## 單元測試 (Unit Test)

測試單一類別或方法,隔離外部依賴

### 1. JUnit 5 基礎

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class ProductServiceTest {
    
    private ProductService productService;
    
    @BeforeEach
    void setUp() {
        productService = new ProductService();
    }
    
    @Test
    @DisplayName("測試計算折扣價格")
    void testCalculateDiscountPrice() {
        // Given
        Product product = new Product("iPhone", new BigDecimal("10000"));
        
        // When
        BigDecimal discountPrice = productService.calculateDiscountPrice(product, 0.8);
        
        // Then
        assertEquals(new BigDecimal("8000.00"), discountPrice);
    }
    
    @Test
    @DisplayName("測試無效折扣應拋出例外")
    void testInvalidDiscount() {
        Product product = new Product("iPhone", new BigDecimal("10000"));
        
        // 斷言會拋出例外
        assertThrows(IllegalArgumentException.class, () -> {
            productService.calculateDiscountPrice(product, 1.5);
        });
    }
    
    @ParameterizedTest
    @DisplayName("測試多種折扣情境")
    @CsvSource({
        "10000, 0.9, 9000",
        "10000, 0.8, 8000",
        "10000, 0.7, 7000"
    })
    void testVariousDiscounts(BigDecimal price, double discount, BigDecimal expected) {
        Product product = new Product("iPhone", price);
        assertEquals(expected, productService.calculateDiscountPrice(product, discount));
    }
    
    @Test
    @Timeout(value = 1, unit = TimeUnit.SECONDS)
    @DisplayName("測試效能:應在 1 秒內完成")
    void testPerformance() {
        productService.searchProducts("iPhone");
    }
}
```

### 2. Mockito - 模擬外部依賴

```java
import org.mockito.*;
import static org.mockito.Mockito.*;

class OrderServiceTest {
    
    @Mock
    private ProductRepository productRepository;
    
    @Mock
    private InventoryService inventoryService;
    
    @Mock
    private PaymentService paymentService;
    
    @InjectMocks
    private OrderService orderService;
    
    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }
    
    @Test
    @DisplayName("測試建立訂單成功")
    void testCreateOrderSuccess() {
        // Given
        Long productId = 1L;
        int quantity = 2;
        
        Product product = new Product(productId, "iPhone", new BigDecimal("10000"));
        when(productRepository.findById(productId)).thenReturn(Optional.of(product));
        when(inventoryService.checkStock(productId, quantity)).thenReturn(true);
        when(inventoryService.decreaseStock(productId, quantity)).thenReturn(true);
        when(paymentService.pay(any())).thenReturn(true);
        
        // When
        Order order = orderService.createOrder(productId, quantity);
        
        // Then
        assertNotNull(order);
        assertEquals(new BigDecimal("20000"), order.getTotalAmount());
        
        // 驗證方法被呼叫
        verify(productRepository).findById(productId);
        verify(inventoryService).checkStock(productId, quantity);
        verify(inventoryService).decreaseStock(productId, quantity);
        verify(paymentService).pay(any());
    }
    
    @Test
    @DisplayName("測試庫存不足應拋出例外")
    void testInsufficientStock() {
        // Given
        Long productId = 1L;
        int quantity = 100;
        
        Product product = new Product(productId, "iPhone", new BigDecimal("10000"));
        when(productRepository.findById(productId)).thenReturn(Optional.of(product));
        when(inventoryService.checkStock(productId, quantity)).thenReturn(false);
        
        // When & Then
        assertThrows(InsufficientStockException.class, () -> {
            orderService.createOrder(productId, quantity);
        });
        
        // 驗證不應該呼叫付款
        verify(paymentService, never()).pay(any());
    }
    
    @Test
    @DisplayName("測試付款失敗應回滾庫存")
    void testPaymentFailureRollback() {
        // Given
        Long productId = 1L;
        int quantity = 2;
        
        Product product = new Product(productId, "iPhone", new BigDecimal("10000"));
        when(productRepository.findById(productId)).thenReturn(Optional.of(product));
        when(inventoryService.checkStock(productId, quantity)).thenReturn(true);
        when(inventoryService.decreaseStock(productId, quantity)).thenReturn(true);
        when(paymentService.pay(any())).thenReturn(false);
        
        // When & Then
        assertThrows(PaymentFailedException.class, () -> {
            orderService.createOrder(productId, quantity);
        });
        
        // 驗證應該回滾庫存
        verify(inventoryService).increaseStock(productId, quantity);
    }
}
```

### 3. ArgumentCaptor - 捕獲參數

```java
@Test
@DisplayName("測試發送訂單通知")
void testSendOrderNotification() {
    // Given
    Order order = new Order(1L, new BigDecimal("20000"));
    ArgumentCaptor<EmailMessage> emailCaptor = ArgumentCaptor.forClass(EmailMessage.class);
    
    // When
    orderService.sendOrderNotification(order);
    
    // Then
    verify(emailService).send(emailCaptor.capture());
    
    EmailMessage sentEmail = emailCaptor.getValue();
    assertEquals("order@example.com", sentEmail.getTo());
    assertTrue(sentEmail.getContent().contains("20000"));
}
```

## 整合測試 (Integration Test)

測試多個元件協作,包含真實的資料庫、快取等

### 1. Spring Boot Test

```java
@SpringBootTest
@ActiveProfiles("test")
@Transactional  // 每個測試方法結束後回滾
class ProductServiceIntegrationTest {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    @DisplayName("整合測試:建立商品")
    void testCreateProduct() {
        // Given
        ProductDTO dto = new ProductDTO();
        dto.setName("iPhone 14");
        dto.setPrice(new BigDecimal("30000"));
        dto.setCategoryId(1L);
        
        // When
        Product product = productService.createProduct(dto);
        
        // Then
        assertNotNull(product.getId());
        
        // 從資料庫查詢驗證
        Optional<Product> saved = productRepository.findById(product.getId());
        assertTrue(saved.isPresent());
        assertEquals("iPhone 14", saved.get().getName());
    }
}
```

### 2. TestContainers - 真實資料庫測試

```java
@SpringBootTest
@ActiveProfiles("test")
@Testcontainers
class OrderServiceIntegrationTest {
    
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:5.7")
        .withDatabaseName("test")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
    }
    
    @Autowired
    private OrderService orderService;
    
    @Test
    @DisplayName("整合測試:建立訂單包含快取")
    void testCreateOrderWithCache() {
        // Given
        Long productId = 1L;
        int quantity = 2;
        
        // When
        Order order1 = orderService.createOrder(productId, quantity);
        Order order2 = orderService.getOrder(order1.getId());  // 應該從快取取得
        
        // Then
        assertEquals(order1.getId(), order2.getId());
    }
}
```

### 3. MockMvc - 測試 REST API

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private ProductService productService;
    
    @Test
    @DisplayName("測試取得商品列表")
    void testGetProducts() throws Exception {
        // Given
        List<Product> products = Arrays.asList(
            new Product(1L, "iPhone", new BigDecimal("30000")),
            new Product(2L, "iPad", new BigDecimal("20000"))
        );
        when(productService.getAllProducts()).thenReturn(products);
        
        // When & Then
        mockMvc.perform(get("/api/products")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$", hasSize(2)))
            .andExpect(jsonPath("$[0].name").value("iPhone"))
            .andExpect(jsonPath("$[0].price").value(30000))
            .andExpect(jsonPath("$[1].name").value("iPad"));
    }
    
    @Test
    @DisplayName("測試建立商品")
    void testCreateProduct() throws Exception {
        // Given
        ProductDTO dto = new ProductDTO();
        dto.setName("MacBook");
        dto.setPrice(new BigDecimal("50000"));
        
        Product product = new Product(1L, "MacBook", new BigDecimal("50000"));
        when(productService.createProduct(any())).thenReturn(product);
        
        // When & Then
        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"MacBook\",\"price\":50000}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("MacBook"));
    }
    
    @Test
    @DisplayName("測試參數驗證")
    void testValidation() throws Exception {
        // When & Then
        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"\",\"price\":-1}"))  // 無效資料
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors", hasSize(2)));
    }
}
```

## 端到端測試 (E2E Test)

### RestAssured - API 測試

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductE2ETest {
    
    @LocalServerPort
    private int port;
    
    @BeforeEach
    void setUp() {
        RestAssured.port = port;
    }
    
    @Test
    @DisplayName("端到端測試:完整購物流程")
    void testCompleteShoppingFlow() {
        // 1. 註冊使用者
        String token = given()
            .contentType(ContentType.JSON)
            .body("{\"username\":\"test\",\"password\":\"123456\"}")
        .when()
            .post("/api/auth/register")
        .then()
            .statusCode(200)
            .extract().path("token");
        
        // 2. 瀏覽商品
        Long productId = given()
            .header("Authorization", "Bearer " + token)
        .when()
            .get("/api/products")
        .then()
            .statusCode(200)
            .body("size()", greaterThan(0))
            .extract().path("[0].id");
        
        // 3. 加入購物車
        given()
            .header("Authorization", "Bearer " + token)
            .contentType(ContentType.JSON)
            .body("{\"productId\":" + productId + ",\"quantity\":2}")
        .when()
            .post("/api/cart/items")
        .then()
            .statusCode(200);
        
        // 4. 結帳
        Long orderId = given()
            .header("Authorization", "Bearer " + token)
            .contentType(ContentType.JSON)
        .when()
            .post("/api/orders/checkout")
        .then()
            .statusCode(201)
            .body("status", equalTo("PENDING"))
            .extract().path("id");
        
        // 5. 查詢訂單
        given()
            .header("Authorization", "Bearer " + token)
        .when()
            .get("/api/orders/" + orderId)
        .then()
            .statusCode(200)
            .body("id", equalTo(orderId));
    }
}
```

## 測試覆蓋率

### pom.xml

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```bash
# 執行測試並產生覆蓋率報告
mvn clean test

# 查看報告
open target/site/jacoco/index.html
```

## 測試資料準備

### 1. @Sql - 初始化資料

```java
@SpringBootTest
@Sql("/test-data.sql")
class ProductRepositoryTest {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    void testFindByCategory() {
        List<Product> products = productRepository.findByCategory("手機");
        assertEquals(5, products.size());
    }
}
```

### 2. DbUnit - 資料驗證

```java
@SpringBootTest
@TestExecutionListeners({
    DbUnitTestExecutionListener.class
})
class OrderRepositoryTest {
    
    @Test
    @DatabaseSetup("/datasets/orders.xml")
    @ExpectedDatabase("/datasets/expected-orders.xml")
    void testUpdateOrderStatus() {
        orderRepository.updateStatus(1L, OrderStatus.COMPLETED);
    }
}
```

### 3. Faker - 產生測試資料

```java
class TestDataFactory {
    
    private static final Faker faker = new Faker(new Locale("zh-TW"));
    
    public static Product createRandomProduct() {
        return Product.builder()
            .name(faker.commerce().productName())
            .price(new BigDecimal(faker.number().numberBetween(1000, 50000)))
            .description(faker.lorem().paragraph())
            .build();
    }
    
    public static User createRandomUser() {
        return User.builder()
            .username(faker.name().username())
            .email(faker.internet().emailAddress())
            .phone(faker.phoneNumber().cellPhone())
            .build();
    }
}
```

## 效能測試

### JMH - Micro Benchmark

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Thread)
public class ProductServiceBenchmark {
    
    private ProductService productService;
    private List<Product> products;
    
    @Setup
    public void setup() {
        productService = new ProductService();
        products = createTestProducts(10000);
    }
    
    @Benchmark
    public List<Product> testFilterProducts() {
        return products.stream()
            .filter(p -> p.getPrice().compareTo(new BigDecimal("10000")) > 0)
            .collect(Collectors.toList());
    }
}
```

## 測試實踐經驗

### 1. 命名規範

```
測試方法名稱:test + 測試場景 + 預期結果
例如:testCreateOrder_InsufficientStock_ThrowsException
```

### 2. AAA 模式

```java
@Test
void testExample() {
    // Arrange (Given) - 準備測試資料
    Product product = new Product("iPhone", new BigDecimal("10000"));
    
    // Act (When) - 執行測試動作
    BigDecimal price = productService.calculateDiscountPrice(product, 0.8);
    
    // Assert (Then) - 驗證結果
    assertEquals(new BigDecimal("8000"), price);
}
```

### 3. 測試獨立性

```java
// (失敗) 錯誤:測試間有依賴
@Test
void test1() {
    product = new Product("iPhone", new BigDecimal("10000"));
}

@Test
void test2() {
    // 依賴 test1 的 product
    assertNotNull(product);
}

// (成功) 正確:每個測試獨立
@BeforeEach
void setUp() {
    product = new Product("iPhone", new BigDecimal("10000"));
}
```

### 4. 不要測試框架

```java
// (失敗) 錯誤:測試 Spring 框架
@Test
void testAutowired() {
    assertNotNull(productRepository);
}

// (成功) 正確:測試業務邏輯
@Test
void testCreateProduct() {
    Product product = productService.createProduct(dto);
    assertNotNull(product.getId());
}
```

## 小結

完善的測試策略:
- **單元測試**:快速驗證邏輯,使用 Mockito 隔離依賴
- **整合測試**:驗證元件協作,使用 TestContainers 真實環境
- **端到端測試**:驗證完整流程,使用 RestAssured
- **測試覆蓋率**:至少 80% 行覆蓋率
- **持續整合**:每次提交都執行測試

測試不是負擔,是品質的保證!

下週我們將學習 **效能優化**,讓系統跑得更快!
