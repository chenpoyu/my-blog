---
layout: post
title: "Spring 非同步與排程使用筆記"
date: 2017-10-13 09:25:00 +0800
categories: [設計模式, Java]
tags: [Spring, 非同步, 排程, 效能優化]
---

電商系統中有很多耗時操作:發送郵件、產生報表、處理訂單...如果都同步執行,使用者體驗會很差。Spring 提供了優雅的非同步和排程解決方案。

## 為什麼需要非同步?

```java
// 同步執行:使用者等待 3 秒
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    Order order = orderService.create(request);      // 200ms
    paymentService.process(order);                   // 500ms
    inventoryService.deductStock(order);             // 300ms
    emailService.sendConfirmation(order);            // 2000ms - 很慢!
    smsService.sendNotification(order);              // 1000ms
    return order;  // 使用者等了 4 秒!
}
```

## Spring 非同步

### 1. 啟用非同步

```java
@SpringBootApplication
@EnableAsync
public class MyShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyShopApplication.class, args);
    }
}
```

### 2. 使用 @Async

```java
@Service
public class NotificationService {
    
    // 非同步發送郵件
    @Async
    public void sendEmail(Order order) {
        // 耗時操作
        emailClient.send(order.getUser().getEmail(), 
                        "訂單確認", 
                        buildEmailContent(order));
    }
    
    // 非同步發送簡訊
    @Async
    public void sendSms(Order order) {
        smsClient.send(order.getUser().getPhone(),
                      "您的訂單已建立");
    }
    
    // 返回 Future 可以取得執行結果
    @Async
    public Future<Boolean> sendWithResult(String email, String content) {
        boolean success = emailClient.send(email, content);
        return new AsyncResult<>(success);
    }
    
    // 使用 CompletableFuture (推薦)
    @Async
    public CompletableFuture<Boolean> sendAsync(String email, String content) {
        boolean success = emailClient.send(email, content);
        return CompletableFuture.completedFuture(success);
    }
}
```

### 3. 改造訂單服務

```java
@Service
public class OrderService {
    
    @Autowired
    private NotificationService notificationService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 核心業務邏輯:同步執行
        Order order = buildOrder(request);
        orderRepository.save(order);
        paymentService.process(order);
        inventoryService.deductStock(order);
        
        // 通知類操作:非同步執行
        notificationService.sendEmail(order);
        notificationService.sendSms(order);
        
        return order;  // 使用者只等 1 秒!
    }
}
```

## 執行緒池配置

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    @Bean(name = "taskExecutor")
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        
        // 核心執行緒數
        executor.setCorePoolSize(10);
        
        // 最大執行緒數
        executor.setMaxPoolSize(50);
        
        // 佇列容量
        executor.setQueueCapacity(100);
        
        // 執行緒名稱前綴
        executor.setThreadNamePrefix("async-");
        
        // 拒絕策略:由呼叫執行緒執行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        // 等待任務完成後關閉
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        
        executor.initialize();
        return executor;
    }
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            System.err.println("非同步方法執行錯誤: " + method.getName());
            ex.printStackTrace();
        };
    }
}
```

## 實戰案例:非同步任務

### 場景一:批次匯入商品

```java
@Service
public class ProductImportService {
    
    @Async
    public CompletableFuture<ImportResult> importProducts(MultipartFile file) {
        ImportResult result = new ImportResult();
        
        try {
            List<Product> products = parseFile(file);
            
            int successCount = 0;
            int failCount = 0;
            
            for (Product product : products) {
                try {
                    productRepository.save(product);
                    successCount++;
                } catch (Exception e) {
                    failCount++;
                    result.addError(product.getName(), e.getMessage());
                }
            }
            
            result.setSuccessCount(successCount);
            result.setFailCount(failCount);
            
        } catch (Exception e) {
            result.setFailed(true);
            result.setMessage(e.getMessage());
        }
        
        return CompletableFuture.completedFuture(result);
    }
}

@RestController
public class ImportController {
    
    @Autowired
    private ProductImportService importService;
    
    @PostMapping("/import")
    public ResponseEntity<String> importProducts(@RequestParam MultipartFile file) {
        
        // 非同步處理,立即返回
        CompletableFuture<ImportResult> future = importService.importProducts(file);
        
        // 可以將 future 存起來,讓使用者後續查詢進度
        String taskId = UUID.randomUUID().toString();
        taskManager.register(taskId, future);
        
        return ResponseEntity.ok("匯入任務已啟動,任務ID: " + taskId);
    }
    
    @GetMapping("/import/{taskId}")
    public ResponseEntity<ImportResult> getImportStatus(@PathVariable String taskId) {
        CompletableFuture<ImportResult> future = taskManager.get(taskId);
        
        if (future == null) {
            return ResponseEntity.notFound().build();
        }
        
        if (!future.isDone()) {
            return ResponseEntity.ok(ImportResult.running());
        }
        
        try {
            ImportResult result = future.get();
            return ResponseEntity.ok(result);
        } catch (Exception e) {
            return ResponseEntity.ok(ImportResult.failed(e.getMessage()));
        }
    }
}
```

### 場景二:並行查詢多個服務

```java
@Service
public class ProductDetailService {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ReviewService reviewService;
    
    @Autowired
    private RecommendationService recommendationService;
    
    public ProductDetailDTO getProductDetail(Long productId) {
        
        // 並行查詢多個資料源
        CompletableFuture<Product> productFuture = 
            CompletableFuture.supplyAsync(() -> 
                productService.findById(productId));
        
        CompletableFuture<List<Review>> reviewsFuture = 
            CompletableFuture.supplyAsync(() -> 
                reviewService.findByProductId(productId));
        
        CompletableFuture<List<Product>> recommendationsFuture = 
            CompletableFuture.supplyAsync(() -> 
                recommendationService.getRecommendations(productId));
        
        // 等待所有完成
        CompletableFuture.allOf(productFuture, reviewsFuture, recommendationsFuture)
            .join();
        
        // 組合結果
        ProductDetailDTO detail = new ProductDetailDTO();
        detail.setProduct(productFuture.join());
        detail.setReviews(reviewsFuture.join());
        detail.setRecommendations(recommendationsFuture.join());
        
        return detail;
    }
}
```

## 排程任務

### 1. 啟用排程

```java
@SpringBootApplication
@EnableScheduling
public class MyShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyShopApplication.class, args);
    }
}
```

### 2. 使用 @Scheduled

```java
@Component
public class ScheduledTasks {
    
    // 固定延遲:上次執行完成後 5 秒再執行
    @Scheduled(fixedDelay = 5000)
    public void processOrders() {
        System.out.println("處理待處理訂單...");
    }
    
    // 固定頻率:每 10 秒執行一次
    @Scheduled(fixedRate = 10000)
    public void syncInventory() {
        System.out.println("同步庫存...");
    }
    
    // 初始延遲:啟動後 1 分鐘才開始執行
    @Scheduled(initialDelay = 60000, fixedRate = 300000)
    public void generateReport() {
        System.out.println("產生報表...");
    }
    
    // Cron 表達式:每天凌晨 2 點執行
    @Scheduled(cron = "0 0 2 * * ?")
    public void cleanupExpiredOrders() {
        System.out.println("清理過期訂單...");
    }
    
    // 每小時的第 30 分執行
    @Scheduled(cron = "0 30 * * * ?")
    public void updateProductRanking() {
        System.out.println("更新商品排行...");
    }
    
    // 每週一早上 9 點執行
    @Scheduled(cron = "0 0 9 ? * MON")
    public void weeklyReport() {
        System.out.println("產生週報...");
    }
}
```

### Cron 表達式說明

```
秒 分 時 日 月 星期
0  0  2  *  *  ?      每天凌晨 2 點
0  */5 * * * ?        每 5 分鐘
0  0  */2 * * ?       每 2 小時
0  0  12 * * ?        每天中午 12 點
0  0  9-17 * * ?      每天 9-17 點整點
0  0  0  1  * ?       每月 1 號凌晨
0  0  9  ? * MON-FRI  週一到週五早上 9 點
```

## 實戰案例:排程任務

### 場景一:定時清理過期訂單

```java
@Component
public class OrderCleanupTask {
    
    @Autowired
    private OrderRepository orderRepository;
    
    // 每天凌晨 3 點執行
    @Scheduled(cron = "0 0 3 * * ?")
    public void cleanupExpiredOrders() {
        LocalDateTime expireTime = LocalDateTime.now().minusDays(30);
        
        List<Order> expiredOrders = orderRepository
            .findByStatusAndCreatedAtBefore(OrderStatus.CANCELLED, expireTime);
        
        int count = 0;
        for (Order order : expiredOrders) {
            orderRepository.delete(order);
            count++;
        }
        
        logger.info("清理了 {} 筆過期訂單", count);
    }
}
```

### 場景二:定時同步庫存

```java
@Component
public class InventorySyncTask {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private RedisTemplate<String, Integer> redisTemplate;
    
    // 每 5 分鐘執行一次
    @Scheduled(fixedRate = 300000)
    public void syncInventory() {
        Set<String> keys = redisTemplate.keys("inventory:*");
        
        if (keys == null || keys.isEmpty()) {
            return;
        }
        
        for (String key : keys) {
            Long productId = Long.parseLong(key.split(":")[1]);
            Integer redisStock = redisTemplate.opsForValue().get(key);
            
            if (redisStock != null) {
                // 將 Redis 庫存同步到資料庫
                inventoryService.updateStock(productId, redisStock);
            }
        }
        
        logger.info("同步了 {} 個商品的庫存", keys.size());
    }
}
```

### 場景三:定時產生報表

```java
@Component
public class ReportGenerationTask {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private ReportService reportService;
    
    // 每天早上 8 點產生昨日報表
    @Scheduled(cron = "0 0 8 * * ?")
    public void generateDailyReport() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        
        DailyReport report = new DailyReport();
        report.setDate(yesterday);
        
        // 計算昨日訂單數
        long orderCount = orderRepository
            .countByCreatedAtBetween(
                yesterday.atStartOfDay(),
                yesterday.plusDays(1).atStartOfDay()
            );
        report.setOrderCount(orderCount);
        
        // 計算昨日營業額
        Double revenue = orderRepository
            .sumTotalAmountByCreatedAtBetween(
                yesterday.atStartOfDay(),
                yesterday.plusDays(1).atStartOfDay()
            );
        report.setRevenue(revenue);
        
        reportService.save(report);
        
        // 發送報表郵件給管理員
        emailService.sendDailyReport(report);
        
        logger.info("已產生 {} 的日報", yesterday);
    }
}
```

## 排程任務配置

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {
    
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        // 配置執行緒池
        taskRegistrar.setScheduler(taskScheduler());
    }
    
    @Bean
    public Executor taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(60);
        scheduler.initialize();
        return scheduler;
    }
}
```

## 動態排程

```java
@Service
public class DynamicScheduleService {
    
    @Autowired
    private TaskScheduler taskScheduler;
    
    private Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();
    
    // 動態新增排程任務
    public void scheduleTask(String taskId, Runnable task, String cron) {
        ScheduledFuture<?> future = taskScheduler.schedule(
            task,
            new CronTrigger(cron)
        );
        
        scheduledTasks.put(taskId, future);
    }
    
    // 取消排程任務
    public void cancelTask(String taskId) {
        ScheduledFuture<?> future = scheduledTasks.get(taskId);
        if (future != null) {
            future.cancel(false);
            scheduledTasks.remove(taskId);
        }
    }
}
```

## 實踐經驗

1. **合理設定執行緒池大小**:避免執行緒過多或過少
2. **處理例外**:非同步方法要妥善處理例外
3. **避免阻塞**:非同步方法內不要有阻塞操作
4. **監控任務狀態**:記錄任務執行時間和結果
5. **冪等性**:排程任務要設計成冪等的

## 小結

Spring 非同步和排程提供了:
- **@Async**:輕鬆實現非同步執行
- **@Scheduled**:簡單的排程任務
- **執行緒池管理**:靈活的執行緒池配置
- **CompletableFuture**:非同步編程

掌握非同步和排程,你的應用會更有效率!下週我們將學習 **Spring 微服務架構**,看看如何將單體應用拆分成微服務。
