---
layout: post
title: "Spring AOP 使用筆記"
date: 2017-08-18 13:30:00 +0800
categories: [設計模式, Java]
tags: [Spring, AOP, 面向切面程式設計]
---

上週我們學習了 Spring 的 IoC 容器（請參考 [Spring 框架入門](/posts/2017/08/11/spring-ioc-di/)），解決了物件之間的依賴關係。但在電商系統中，還有一類特殊的需求散落在各處：日誌記錄、效能監控、交易管理、權限檢查...。這就是「橫切關注點」，AOP 就是為此而生。

## 什麼是橫切關注點？

在電商系統中，我們的業務邏輯被以下功能「橫切」：

```java
@Service
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // 1. 記錄日誌
        logger.info("開始建立訂單：{}", request);
        long startTime = System.currentTimeMillis();
        
        try {
            // 2. 檢查權限
            if (!hasPermission(getCurrentUser(), "CREATE_ORDER")) {
                throw new UnauthorizedException();
            }
            
            // 3. 開始交易
            transactionManager.begin();
            
            try {
                // 真正的業務邏輯
                Order order = doCreateOrder(request);
                
                // 4. 提交交易
                transactionManager.commit();
                
                // 5. 記錄效能
                long duration = System.currentTimeMillis() - startTime;
                logger.info("訂單建立完成，耗時：{}ms", duration);
                
                return order;
                
            } catch (Exception e) {
                // 6. 回滾交易
                transactionManager.rollback();
                throw e;
            }
            
        } catch (Exception e) {
            // 7. 記錄錯誤
            logger.error("訂單建立失敗", e);
            throw e;
        }
    }
}
```

業務邏輯被大量的基礎設施代碼淹沒了！而且這些代碼在每個方法中重複出現。

## AOP 的救贖

**面向切面程式設計（Aspect-Oriented Programming, AOP）** 讓我們可以將這些橫切關注點抽離出來：

```java
@Service
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // 只專注於業務邏輯！
        Product product = productRepository.findById(request.getProductId());
        
        if (!inventoryService.checkStock(product.getId(), request.getQuantity())) {
            throw new InsufficientStockException(product.getId());
        }
        
        Order order = new Order();
        order.setProduct(product);
        order.setQuantity(request.getQuantity());
        
        paymentService.process(request.getPaymentInfo());
        inventoryService.deductStock(product.getId(), request.getQuantity());
        
        return order;
    }
}
```

日誌、效能監控、交易管理都被 AOP 自動處理了！

## AOP 核心概念

### 1. Aspect（切面）
橫切關注點的模組化，例如：日誌切面、效能監控切面。

### 2. Join Point（連接點）
程式執行的某個時刻，例如：方法執行時、例外拋出時。

### 3. Pointcut（切入點）
定義在哪些 Join Point 上執行切面邏輯。

### 4. Advice（通知）
在切入點執行的動作。

### 5. Weaving（織入）
將切面應用到目標物件的過程。

## 實戰案例：電商系統的 AOP 應用

### 場景一：日誌記錄

```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger logger = LoggerFactory.getLogger(LoggingAspect.class);
    
    // 定義切入點：所有 Service 的 public 方法
    @Pointcut("execution(public * com.example.service.*.*(..))")
    public void serviceLayer() {}
    
    // 前置通知：方法執行前
    @Before("serviceLayer()")
    public void logBefore(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        Object[] args = joinPoint.getArgs();
        
        logger.info("呼叫 {}.{}，參數：{}", className, methodName, Arrays.toString(args));
    }
    
    // 後置通知：方法執行後（無論成功或失敗）
    @After("serviceLayer()")
    public void logAfter(JoinPoint joinPoint) {
        logger.info("{}.{} 執行結束", 
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName());
    }
    
    // 返回通知：方法成功返回後
    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        logger.info("{}.{} 返回結果：{}", 
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName(),
            result);
    }
    
    // 異常通知：方法拋出例外時
    @AfterThrowing(pointcut = "serviceLayer()", throwing = "error")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable error) {
        logger.error("{}.{} 拋出例外：{}", 
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName(),
            error.getMessage());
    }
}
```

### 場景二：效能監控

```java
@Aspect
@Component
public class PerformanceAspect {
    private static final Logger logger = LoggerFactory.getLogger(PerformanceAspect.class);
    
    // 環繞通知：完全控制方法執行
    @Around("execution(* com.example.service.*.*(..))")
    public Object measureExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        long startTime = System.currentTimeMillis();
        
        try {
            // 執行目標方法
            Object result = joinPoint.proceed();
            
            long duration = System.currentTimeMillis() - startTime;
            
            // 如果執行時間超過 1 秒，記錄警告
            if (duration > 1000) {
                logger.warn("{}.{} 執行時間過長：{}ms", className, methodName, duration);
            } else {
                logger.debug("{}.{} 執行時間：{}ms", className, methodName, duration);
            }
            
            return result;
            
        } catch (Throwable e) {
            long duration = System.currentTimeMillis() - startTime;
            logger.error("{}.{} 執行失敗，耗時：{}ms", className, methodName, duration);
            throw e;
        }
    }
}
```

### 場景三：權限檢查

首先定義自定義註解：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();  // 需要的權限
}
```

然後在業務方法上使用：

```java
@Service
public class OrderService {
    
    @RequirePermission("ORDER:CREATE")
    public Order createOrder(OrderRequest request) {
        // 業務邏輯
    }
    
    @RequirePermission("ORDER:DELETE")
    public void cancelOrder(String orderId) {
        // 業務邏輯
    }
}
```

建立權限檢查切面：

```java
@Aspect
@Component
public class PermissionAspect {
    
    @Autowired
    private SecurityService securityService;
    
    // 切入所有帶有 @RequirePermission 註解的方法
    @Before("@annotation(requirePermission)")
    public void checkPermission(JoinPoint joinPoint, RequirePermission requirePermission) {
        String permission = requirePermission.value();
        User currentUser = securityService.getCurrentUser();
        
        if (!securityService.hasPermission(currentUser, permission)) {
            throw new UnauthorizedException(
                String.format("使用者 %s 沒有 %s 權限", currentUser.getName(), permission)
            );
        }
        
        logger.info("使用者 {} 通過權限檢查：{}", currentUser.getName(), permission);
    }
}
```

### 場景四：交易管理

雖然 Spring 提供了 `@Transactional` 註解，但我們也可以自己實作交易切面：

```java
@Aspect
@Component
public class TransactionAspect {
    
    @Autowired
    private TransactionManager transactionManager;
    
    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object manageTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        Transaction transaction = transactionManager.begin();
        
        try {
            // 執行業務方法
            Object result = joinPoint.proceed();
            
            // 提交交易
            transactionManager.commit(transaction);
            
            logger.info("交易提交成功：{}", transaction.getId());
            return result;
            
        } catch (Exception e) {
            // 回滾交易
            transactionManager.rollback(transaction);
            logger.error("交易回滾：{}", transaction.getId(), e);
            throw e;
        }
    }
}
```

### 場景五：快取管理

```java
@Aspect
@Component
public class CacheAspect {
    
    @Autowired
    private CacheManager cacheManager;
    
    // 自定義快取註解
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Cacheable {
        String key();
        int expireSeconds() default 300;
    }
    
    @Around("@annotation(cacheable)")
    public Object cacheResult(ProceedingJoinPoint joinPoint, Cacheable cacheable) throws Throwable {
        // 產生快取鍵
        String cacheKey = generateCacheKey(cacheable.key(), joinPoint.getArgs());
        
        // 嘗試從快取取得
        Object cached = cacheManager.get(cacheKey);
        if (cached != null) {
            logger.debug("快取命中：{}", cacheKey);
            return cached;
        }
        
        // 執行方法
        Object result = joinPoint.proceed();
        
        // 儲存到快取
        cacheManager.put(cacheKey, result, cacheable.expireSeconds());
        logger.debug("快取儲存：{}", cacheKey);
        
        return result;
    }
    
    private String generateCacheKey(String prefix, Object[] args) {
        return prefix + ":" + Arrays.toString(args).hashCode();
    }
}

// 使用範例
@Service
public class ProductService {
    
    @Cacheable(key = "product", expireSeconds = 600)
    public Product getProduct(String productId) {
        // 實際查詢資料庫
        return productRepository.findById(productId);
    }
}
```

## Pointcut 表達式詳解

```java
// 1. 匹配特定方法
@Pointcut("execution(public * com.example.service.OrderService.createOrder(..))")

// 2. 匹配特定類別的所有方法
@Pointcut("execution(* com.example.service.OrderService.*(..))")

// 3. 匹配特定套件下所有類別的所有方法
@Pointcut("execution(* com.example.service.*.*(..))")

// 4. 匹配特定套件及其子套件
@Pointcut("execution(* com.example.service..*.*(..))")

// 5. 匹配特定返回類型
@Pointcut("execution(com.example.model.Order *(..))")

// 6. 匹配特定參數
@Pointcut("execution(* *(String, int))")

// 7. 匹配帶有特定註解的方法
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")

// 8. 匹配帶有特定註解的類別中的所有方法
@Pointcut("@within(org.springframework.stereotype.Service)")

// 9. 組合多個條件
@Pointcut("execution(* com.example.service.*.*(..)) && @annotation(cacheable)")
```

## Advice 類型總結

| 類型 | 註解 | 執行時機 | 適用場景 |
|------|------|----------|----------|
| Before | @Before | 方法執行前 | 權限檢查、參數驗證 |
| After | @After | 方法執行後（無論成功或失敗） | 資源清理 |
| AfterReturning | @AfterReturning | 方法成功返回後 | 結果處理、快取 |
| AfterThrowing | @AfterThrowing | 方法拋出例外時 | 錯誤處理、回滾 |
| Around | @Around | 完全控制方法執行 | 效能監控、交易管理 |

## AOP 的實踐經驗

1. **保持切面單一職責**：一個切面只處理一個關注點
2. **避免過度使用**：不是所有代碼都適合用 AOP
3. **注意執行順序**：使用 `@Order` 控制多個切面的執行順序
4. **效能考量**：AOP 會產生代理物件，有額外開銷
5. **測試友善**：切面邏輯也要寫單元測試

```java
@Aspect
@Component
@Order(1)  // 數字越小優先級越高
public class LoggingAspect {
    // ...
}

@Aspect
@Component
@Order(2)
public class PerformanceAspect {
    // ...
}
```

## 小結

Spring AOP 是處理橫切關注點的利器：
- **分離關注點**：業務邏輯與基礎設施代碼分離
- **提高重用性**：切面邏輯可以應用到多個類別
- **降低侵入性**：不需要修改原有代碼
- **提升可維護性**：集中管理橫切關注點

掌握 AOP，你的代碼會變得更簡潔、更優雅。下週我們將學習 **Spring MVC**，看看如何建構 RESTful API。
