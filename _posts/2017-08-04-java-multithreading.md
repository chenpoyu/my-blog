---
layout: post
title: "Java 多執行緒筆記"
date: 2017-08-04 15:20:00 +0800
categories: [設計模式, Java]
tags: [Java, 多執行緒, 並發, 執行緒安全]
---

電商系統常常需要同時處理大量請求：用戶瀏覽商品、下單、付款、查詢訂單...。如何有效利用多核心 CPU，提升系統吞吐量？這就需要理解 Java 的多執行緒機制。

## 為什麼需要多執行緒？

### 單執行緒的限制

```java
public class OrderService {
    
    // 單執行緒處理訂單
    public void processOrders(List<Order> orders) {
        for (Order order : orders) {
            validateOrder(order);        // 100ms
            checkInventory(order);       // 200ms
            processPayment(order);       // 500ms
            sendNotification(order);     // 100ms
        }
        // 處理 10 筆訂單需要 9 秒！
    }
}
```

在單執行緒環境下，所有操作都是串行的，無法充分利用 CPU 資源。

### 多執行緒的優勢

```java
public class OrderService {
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    
    // 多執行緒處理訂單
    public void processOrdersConcurrently(List<Order> orders) {
        List<Future<?>> futures = new ArrayList<>();
        
        for (Order order : orders) {
            Future<?> future = executor.submit(() -> {
                validateOrder(order);
                checkInventory(order);
                processPayment(order);
                sendNotification(order);
            });
            futures.add(future);
        }
        
        // 等待所有訂單處理完成
        for (Future<?> future : futures) {
            try {
                future.get();
            } catch (Exception e) {
                logger.error("訂單處理失敗", e);
            }
        }
        // 處理 10 筆訂單只需要約 1 秒！
    }
}
```

## 建立執行緒的方式

### 方式一：繼承 Thread

```java
public class InventoryCheckThread extends Thread {
    private Product product;
    
    public InventoryCheckThread(Product product) {
        this.product = product;
    }
    
    @Override
    public void run() {
        int stock = checkStock(product.getId());
        if (stock < 10) {
            alertLowStock(product, stock);
        }
    }
}

// 使用
InventoryCheckThread thread = new InventoryCheckThread(product);
thread.start();
```

### 方式二：實作 Runnable（推薦）

```java
public class OrderProcessTask implements Runnable {
    private Order order;
    
    public OrderProcessTask(Order order) {
        this.order = order;
    }
    
    @Override
    public void run() {
        try {
            processOrder(order);
        } catch (Exception e) {
            logger.error("處理訂單失敗：" + order.getId(), e);
        }
    }
}

// 使用
Thread thread = new Thread(new OrderProcessTask(order));
thread.start();
```

### 方式三：使用 ExecutorService（最推薦）

```java
public class OrderService {
    // 建立固定大小的執行緒池
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public void submitOrder(Order order) {
        executor.submit(() -> {
            processOrder(order);
        });
    }
    
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }
}
```

## 實戰案例：電商系統的並發處理

### 場景一：批次更新商品庫存

```java
public class InventoryService {
    private ExecutorService executor = Executors.newFixedThreadPool(20);
    
    // 批次更新庫存（串行版本）
    public void updateInventorySerial(List<Product> products) {
        long startTime = System.currentTimeMillis();
        
        for (Product product : products) {
            updateStock(product);  // 假設每次需要 100ms
        }
        
        long duration = System.currentTimeMillis() - startTime;
        logger.info("串行更新 {} 個商品，耗時 {}ms", products.size(), duration);
    }
    
    // 批次更新庫存（並行版本）
    public void updateInventoryConcurrent(List<Product> products) 
            throws InterruptedException {
        long startTime = System.currentTimeMillis();
        
        CountDownLatch latch = new CountDownLatch(products.size());
        
        for (Product product : products) {
            executor.submit(() -> {
                try {
                    updateStock(product);
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();  // 等待所有任務完成
        
        long duration = System.currentTimeMillis() - startTime;
        logger.info("並行更新 {} 個商品，耗時 {}ms", products.size(), duration);
    }
}
```

### 場景二：並行查詢多個資料源

```java
public class ProductAggregationService {
    private ExecutorService executor = Executors.newCachedThreadPool();
    
    // 從多個來源聚合商品資訊
    public ProductDetail getProductDetail(String productId) {
        CompletableFuture<Product> basicInfo = 
            CompletableFuture.supplyAsync(() -> 
                getBasicInfo(productId), executor);
        
        CompletableFuture<Integer> stock = 
            CompletableFuture.supplyAsync(() -> 
                getStock(productId), executor);
        
        CompletableFuture<List<Review>> reviews = 
            CompletableFuture.supplyAsync(() -> 
                getReviews(productId), executor);
        
        CompletableFuture<List<Product>> recommendations = 
            CompletableFuture.supplyAsync(() -> 
                getRecommendations(productId), executor);
        
        // 等待所有查詢完成並組合結果
        return CompletableFuture.allOf(basicInfo, stock, reviews, recommendations)
            .thenApply(v -> {
                ProductDetail detail = new ProductDetail();
                detail.setProduct(basicInfo.join());
                detail.setStock(stock.join());
                detail.setReviews(reviews.join());
                detail.setRecommendations(recommendations.join());
                return detail;
            })
            .join();
    }
}
```

### 場景三：限流與執行緒池

```java
public class OrderSubmissionService {
    // 使用有界佇列防止記憶體溢出
    private ThreadPoolExecutor executor = new ThreadPoolExecutor(
        10,                          // 核心執行緒數
        50,                          // 最大執行緒數
        60L, TimeUnit.SECONDS,       // 閒置執行緒存活時間
        new ArrayBlockingQueue<>(1000),  // 有界佇列
        new ThreadPoolExecutor.CallerRunsPolicy()  // 拒絕策略：由呼叫者執行
    );
    
    public boolean submitOrder(Order order) {
        try {
            executor.execute(() -> {
                try {
                    processOrder(order);
                } catch (Exception e) {
                    logger.error("訂單處理失敗：" + order.getId(), e);
                }
            });
            return true;
        } catch (RejectedExecutionException e) {
            logger.warn("系統繁忙，請稍後再試");
            return false;
        }
    }
    
    // 取得執行緒池狀態
    public PoolStatus getPoolStatus() {
        PoolStatus status = new PoolStatus();
        status.setActiveThreads(executor.getActiveCount());
        status.setPoolSize(executor.getPoolSize());
        status.setQueueSize(executor.getQueue().size());
        status.setCompletedTasks(executor.getCompletedTaskCount());
        return status;
    }
}
```

## 執行緒同步問題

### 問題：競態條件 (Race Condition)

```java
public class InventoryService {
    private Map<String, Integer> stock = new HashMap<>();
    
    //  不安全：多執行緒同時扣庫存會有問題
    public boolean deductStock(String productId, int quantity) {
        Integer current = stock.get(productId);
        if (current >= quantity) {
            stock.put(productId, current - quantity);
            return true;
        }
        return false;
    }
}
```

當兩個執行緒同時讀取庫存為 10，都認為可以扣除 8，結果庫存變成 -6！

### 解決方案一：synchronized

```java
public class InventoryService {
    private Map<String, Integer> stock = new HashMap<>();
    
    //  使用 synchronized 保證執行緒安全
    public synchronized boolean deductStock(String productId, int quantity) {
        Integer current = stock.get(productId);
        if (current >= quantity) {
            stock.put(productId, current - quantity);
            return true;
        }
        return false;
    }
}
```

### 解決方案二：使用並發集合

```java
public class InventoryService {
    private ConcurrentHashMap<String, AtomicInteger> stock = new ConcurrentHashMap<>();
    
    //  使用 AtomicInteger 保證原子性操作
    public boolean deductStock(String productId, int quantity) {
        AtomicInteger current = stock.get(productId);
        if (current == null) {
            return false;
        }
        
        // 使用 CAS (Compare And Swap) 操作
        while (true) {
            int currentValue = current.get();
            if (currentValue < quantity) {
                return false;
            }
            if (current.compareAndSet(currentValue, currentValue - quantity)) {
                return true;
            }
        }
    }
}
```

### 解決方案三：ReentrantLock

```java
public class InventoryService {
    private Map<String, Integer> stock = new HashMap<>();
    private ReentrantLock lock = new ReentrantLock();
    
    public boolean deductStock(String productId, int quantity) {
        lock.lock();
        try {
            Integer current = stock.get(productId);
            if (current >= quantity) {
                stock.put(productId, current - quantity);
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
}
```

## 常見的執行緒池類型

```java
// 1. 固定大小執行緒池：適合負載已知的場景
ExecutorService fixed = Executors.newFixedThreadPool(10);

// 2. 快取執行緒池：適合大量短期任務
ExecutorService cached = Executors.newCachedThreadPool();

// 3. 單執行緒執行器：適合需要順序執行的任務
ExecutorService single = Executors.newSingleThreadExecutor();

// 4. 排程執行緒池：適合定時任務
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
scheduled.scheduleAtFixedRate(() -> {
    syncInventory();
}, 0, 1, TimeUnit.HOURS);
```

## 多執行緒的實踐經驗

1. **優先使用執行緒池**：避免頻繁建立銷毀執行緒
2. **設定有界佇列**：防止記憶體溢出
3. **合理設定執行緒數**：
   - CPU 密集型：執行緒數 = CPU 核心數 + 1
   - I/O 密集型：執行緒數 = CPU 核心數 × 2
4. **避免過度同步**：只鎖必要的臨界區
5. **使用並發集合**：`ConcurrentHashMap`、`CopyOnWriteArrayList`
6. **處理中斷**：正確處理 `InterruptedException`
7. **命名執行緒**：方便除錯

```java
ThreadFactory threadFactory = new ThreadFactory() {
    private AtomicInteger counter = new AtomicInteger(0);
    
    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r);
        thread.setName("OrderProcessor-" + counter.incrementAndGet());
        return thread;
    }
};

ExecutorService executor = Executors.newFixedThreadPool(10, threadFactory);
```

## 小結

多執行緒是提升系統效能的利器，但也帶來了複雜度：
- **並行執行**提升吞吐量
- **執行緒安全**需要同步機制
- **執行緒池**管理執行緒生命週期
- **並發工具**簡化開發

掌握多執行緒，是開發高效能電商系統的基礎。下週我們將正式進入 **Spring 框架**，看看 IoC 容器如何管理物件的生命週期。
