---
layout: post
title: "生產環境實踐經驗筆記"
date: 2017-11-24 15:35:00 +0800
categories: [設計模式, Java]
tags: [生產環境, DevOps, 穩定性, 實踐經驗]
---

上週我們學習了效能優化(請參考 [效能優化](/posts/2017/11/17/performance-optimization/)),讓系統跑得更快。但生產環境不只是快,更重要的是**穩定**!本週我們來看看生產環境的實踐經驗。

## 一、環境管理

### 1. 多環境配置

```
開發環境 (dev)  → 開發人員本機
測試環境 (test) → QA 測試
預發環境 (uat)  → 模擬生產環境
生產環境 (prod) → 真實使用者
```

### application.yml

```yaml
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

---
# 開發環境
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce_dev
    username: dev
    password: dev123
  jpa:
    show-sql: true
  
logging:
  level:
    root: INFO
    com.example: DEBUG

---
# 生產環境
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-db:3306/ecommerce
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 10
  jpa:
    show-sql: false
  
logging:
  level:
    root: WARN
    com.example: INFO
```

### 2. 敏感資訊管理

```yaml
# (失敗) 錯誤:密碼寫在配置檔
spring:
  datasource:
    password: MyP@ssw0rd

# (成功) 正確:使用環境變數
spring:
  datasource:
    password: ${DB_PASSWORD}
```

Kubernetes Secret:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: cm9vdA==
  password: UEBzc3cwcmQ=
```

### 3. 配置中心 (Spring Cloud Config)

```yaml
# bootstrap.yml
spring:
  application:
    name: product-service
  cloud:
    config:
      uri: http://config-server:8888
      profile: ${SPRING_PROFILES_ACTIVE}
      fail-fast: true
      retry:
        max-attempts: 5
```

好處:
- 集中管理配置
- 動態更新配置 (無需重啟)
- 配置版本控制
- 加密敏感資訊

## 二、健康檢查

### 1. Spring Boot Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
  health:
    db:
      enabled: true
    redis:
      enabled: true
```

### 2. 自訂健康檢查

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Override
    public Health health() {
        try {
            // 檢查庫存服務是否正常
            boolean isHealthy = inventoryService.ping();
            
            if (isHealthy) {
                return Health.up()
                    .withDetail("inventory-service", "Available")
                    .build();
            } else {
                return Health.down()
                    .withDetail("inventory-service", "Unavailable")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### 3. 就緒和存活探針

```yaml
# Kubernetes
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
  failureThreshold: 3
```

Spring Boot:
```java
@Component
public class ReadinessStateHealthIndicator implements HealthIndicator {
    
    @Autowired
    private ApplicationContext applicationContext;
    
    @Override
    public Health health() {
        // 檢查是否準備好接收流量
        AvailabilityState state = AvailabilityChangeEvent
            .getLastReadinessState(applicationContext);
        
        return state == ReadinessState.ACCEPTING_TRAFFIC
            ? Health.up().build()
            : Health.down().build();
    }
}
```

## 三、優雅關閉

### 1. Spring Boot 優雅關閉

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### 2. 自訂清理邏輯

```java
@Component
public class GracefulShutdown {
    
    @Autowired
    private OrderService orderService;
    
    @PreDestroy
    public void onShutdown() {
        log.info("開始優雅關閉...");
        
        // 1. 停止接收新請求
        // (由 Spring Boot 自動處理)
        
        // 2. 等待現有請求完成
        log.info("等待現有請求完成...");
        
        // 3. 完成未處理的任務
        orderService.completeAllPendingTasks();
        
        // 4. 關閉連線
        log.info("關閉資料庫連線...");
        
        log.info("優雅關閉完成!");
    }
}
```

### 3. Kubernetes 優雅終止

```yaml
spec:
  containers:
  - name: product-service
    lifecycle:
      preStop:
        exec:
          # 等待 15 秒讓請求完成
          command: ["/bin/sh", "-c", "sleep 15"]
    terminationGracePeriodSeconds: 30
```

流程:
```
1. K8s 發送 SIGTERM 信號
2. 執行 preStop 鉤子 (sleep 15)
3. Spring Boot 開始優雅關閉
4. 等待最多 30 秒
5. 強制終止 (SIGKILL)
```

## 四、錯誤處理

### 1. 全域例外處理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        log.warn("業務例外: {}", e.getMessage());
        
        ErrorResponse error = ErrorResponse.builder()
            .code(e.getCode())
            .message(e.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        // (失敗) 不要暴露內部錯誤
        log.error("系統錯誤", e);
        
        ErrorResponse error = ErrorResponse.builder()
            .code("SYSTEM_ERROR")
            .message("系統繁忙,請稍後再試")  // 通用訊息
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
        MethodArgumentNotValidException e
    ) {
        List<String> errors = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.toList());
        
        ErrorResponse error = ErrorResponse.builder()
            .code("VALIDATION_ERROR")
            .message("參數驗證失敗")
            .details(errors)
            .timestamp(LocalDateTime.now())
            .build();
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
}
```

### 2. 降級與熔斷

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RecommendationService recommendationService;
    
    public ProductDetailVO getProductDetail(Long id) {
        // 主要資訊:必須成功
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        
        ProductDetailVO vo = new ProductDetailVO(product);
        
        // 推薦商品:可降級
        try {
            List<Product> recommendations = recommendationService.getRecommendations(id);
            vo.setRecommendations(recommendations);
        } catch (Exception e) {
            log.warn("推薦服務失敗,使用降級方案", e);
            vo.setRecommendations(Collections.emptyList());  // 降級
        }
        
        return vo;
    }
}
```

### 3. 重試機制

```java
@Service
public class PaymentService {
    
    @Retryable(
        value = {PaymentGatewayException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public boolean processPayment(Order order) {
        // 呼叫第三方支付 API
        return paymentGateway.charge(order);
    }
    
    @Recover
    public boolean recoverPayment(PaymentGatewayException e, Order order) {
        log.error("支付失敗,已重試 3 次: {}", order.getId(), e);
        // 記錄失敗,通知人工處理
        alertService.sendAlert("支付失敗", order);
        return false;
    }
}
```

## 五、限流與防護

### 1. 令牌桶限流

```java
@Component
public class RateLimiter {
    
    // 每秒允許 100 個請求
    private final com.google.common.util.concurrent.RateLimiter rateLimiter = 
        com.google.common.util.concurrent.RateLimiter.create(100.0);
    
    public boolean tryAcquire() {
        return rateLimiter.tryAcquire(100, TimeUnit.MILLISECONDS);
    }
}

@RestController
public class ProductController {
    
    @Autowired
    private RateLimiter rateLimiter;
    
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        if (!rateLimiter.tryAcquire()) {
            throw new TooManyRequestsException("請求過於頻繁,請稍後再試");
        }
        
        return productService.getProduct(id);
    }
}
```

### 2. Redis 分散式限流

```java
@Component
public class RedisRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean tryAcquire(String key, int limit, int windowSeconds) {
        String redisKey = "rate_limit:" + key;
        Long currentTime = System.currentTimeMillis();
        
        // Lua 腳本保證原子性
        String luaScript = 
            "local key = KEYS[1] " +
            "local limit = tonumber(ARGV[1]) " +
            "local window = tonumber(ARGV[2]) " +
            "local current = tonumber(ARGV[3]) " +
            
            "redis.call('zremrangebyscore', key, 0, current - window * 1000) " +
            "local count = redis.call('zcard', key) " +
            
            "if count < limit then " +
            "    redis.call('zadd', key, current, current) " +
            "    redis.call('expire', key, window) " +
            "    return 1 " +
            "else " +
            "    return 0 " +
            "end";
        
        RedisScript<Long> script = RedisScript.of(luaScript, Long.class);
        Long result = redisTemplate.execute(
            script,
            Collections.singletonList(redisKey),
            String.valueOf(limit),
            String.valueOf(windowSeconds),
            String.valueOf(currentTime)
        );
        
        return result != null && result == 1;
    }
}
```

### 3. 防重複提交

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PreventDuplicateSubmit {
    int timeout() default 5;  // 秒
}

@Aspect
@Component
public class PreventDuplicateSubmitAspect {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Around("@annotation(preventDuplicate)")
    public Object around(ProceedingJoinPoint joinPoint, PreventDuplicateSubmit preventDuplicate) 
        throws Throwable {
        
        // 獲取請求唯一標識
        ServletRequestAttributes attributes = 
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        String userId = getCurrentUserId();
        String uri = request.getRequestURI();
        String key = "duplicate:" + userId + ":" + uri;
        
        // 嘗試設定 Redis key
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", preventDuplicate.timeout(), TimeUnit.SECONDS);
        
        if (Boolean.FALSE.equals(success)) {
            throw new DuplicateSubmitException("請勿重複提交");
        }
        
        try {
            return joinPoint.proceed();
        } catch (Exception e) {
            // 失敗時刪除 key,允許重試
            redisTemplate.delete(key);
            throw e;
        }
    }
}

// 使用
@PostMapping("/orders")
@PreventDuplicateSubmit(timeout = 10)
public Order createOrder(@RequestBody OrderDTO dto) {
    return orderService.createOrder(dto);
}
```

## 六、監控告警

### 1. Prometheus Metrics

```java
@Component
public class BusinessMetrics {
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    private final Gauge activeUsers;
    
    public BusinessMetrics(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("type", "business")
            .register(registry);
        
        this.orderTimer = Timer.builder("order.creation.time")
            .description("Order creation time")
            .register(registry);
        
        this.activeUsers = Gauge.builder("users.active", this::getActiveUserCount)
            .description("Active users count")
            .register(registry);
    }
    
    public void recordOrderCreated() {
        orderCounter.increment();
    }
    
    public void recordOrderCreationTime(Runnable task) {
        orderTimer.record(task);
    }
    
    private int getActiveUserCount() {
        // 從 Redis 或資料庫查詢
        return userService.getActiveUserCount();
    }
}
```

### 2. 告警規則 (Prometheus)

```yaml
groups:
- name: application
  rules:
  # CPU 使用率 > 80%
  - alert: HighCpuUsage
    expr: process_cpu_usage > 0.8
    for: 5m
    annotations:
      summary: "CPU 使用率過高"
      description: "{{ $labels.instance }} CPU 使用率 {{ $value }}"
  
  # 記憶體使用率 > 85%
  - alert: HighMemoryUsage
    expr: jvm_memory_used_bytes / jvm_memory_max_bytes > 0.85
    for: 5m
    annotations:
      summary: "記憶體使用率過高"
  
  # 錯誤率 > 1%
  - alert: HighErrorRate
    expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) / rate(http_server_requests_seconds_count[5m]) > 0.01
    for: 2m
    annotations:
      summary: "錯誤率過高"
  
  # 響應時間 > 1 秒
  - alert: SlowResponse
    expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 1
    for: 5m
    annotations:
      summary: "響應時間過慢"
```

### 3. 告警通知 (AlertManager)

```yaml
route:
  receiver: 'team-ops'
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  
  routes:
  - match:
      severity: critical
    receiver: 'team-ops-critical'
    
receivers:
- name: 'team-ops'
  email_configs:
  - to: 'ops@example.com'
  
- name: 'team-ops-critical'
  email_configs:
  - to: 'ops@example.com'
  webhook_configs:
  - url: 'https://hooks.slack.com/services/xxx'
```

## 七、備份與災難恢復

### 1. 資料庫備份

```bash
#!/bin/bash
# 每日備份腳本

DATE=$(date +%Y%m%d)
BACKUP_DIR="/backup/mysql"

# 全量備份
mysqldump -h prod-db -u backup -p$DB_PASSWORD \
  --single-transaction \
  --routines \
  --triggers \
  --all-databases \
  | gzip > $BACKUP_DIR/full_backup_$DATE.sql.gz

# 保留最近 7 天
find $BACKUP_DIR -name "full_backup_*.sql.gz" -mtime +7 -delete

# 上傳到 S3
aws s3 cp $BACKUP_DIR/full_backup_$DATE.sql.gz \
  s3://backups/mysql/
```

### 2. Redis 持久化

```yaml
# redis.conf
save 900 1      # 15 分鐘內至少 1 次修改
save 300 10     # 5 分鐘內至少 10 次修改
save 60 10000   # 1 分鐘內至少 10000 次修改

appendonly yes
appendfsync everysec
```

### 3. 災難恢復演練

```
每季度演練:
1. 模擬資料庫故障
2. 從備份恢復
3. 驗證資料完整性
4. 測量恢復時間 (RTO)
5. 檢查資料遺失 (RPO)
```

## 八、版本管理

### 1. 語意化版本

```
主版本.次版本.修訂號
例如: 1.2.3

主版本:不相容的 API 變更
次版本:向下相容的功能新增
修訂號:向下相容的 Bug 修復
```

### 2. 滾動更新策略

```yaml
# Kubernetes Deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # 最多多 2 個 Pod
      maxUnavailable: 1  # 最多停 1 個 Pod
```

流程:
```
1. 保持 9 個舊版本運行
2. 啟動 2 個新版本
3. 新版本就緒後,停止 1 個舊版本
4. 重複直到全部更新完成
```

### 3. 金絲雀發布

```yaml
# 舊版本 - 90%
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-v1
spec:
  replicas: 9

---
# 新版本 - 10%
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-v2
spec:
  replicas: 1
```

觀察指標:
- 錯誤率
- 響應時間
- CPU/記憶體使用
- 業務指標 (訂單數、轉換率)

沒問題後逐步增加新版本比例。

## 小結

生產環境實踐經驗:
- **環境隔離**:dev/test/uat/prod
- **健康檢查**:隨時知道系統狀態
- **優雅關閉**:不中斷使用者請求
- **錯誤處理**:降級、重試、熔斷
- **限流防護**:保護系統不被打垮
- **監控告警**:及時發現問題
- **備份恢復**:資料安全第一
- **灰度發布**:降低發布風險

穩定性 > 效能 > 功能

下週我們將學習 **監控與日誌**,看看如何快速定位問題!
