---
layout: post
title: "監控與日誌筆記"
date: 2017-12-01 11:25:00 +0800
categories: [設計模式, Java]
tags: [監控, 日誌, ELK, Prometheus, Grafana]
---

上週我們學習了生產環境實踐經驗(請參考 [生產環境實踐經驗](/posts/2017/11/24/production-best-practices/)),本週來看看如何透過監控和日誌快速定位問題!

## 為什麼需要監控與日誌?

```
凌晨 3 點,電話響起...
客戶:「系統很慢,訂單都下不了!」

你:
1. 哪個服務慢? → 需要監控
2. 什麼時候開始的? → 需要日誌
3. 哪裡出錯了? → 需要追蹤

沒有監控和日誌 = 盲人摸象!
```

## 一、日誌實踐經驗

### 1. 結構化日誌

```java
// (失敗) 錯誤:字串拼接
log.info("User " + userId + " created order " + orderId + 
         " with amount " + amount);

// (成功) 正確:參數化日誌
log.info("User {} created order {} with amount {}", userId, orderId, amount);

// (成功) 更好:結構化日誌 (JSON)
MDC.put("userId", userId.toString());
MDC.put("orderId", orderId.toString());
MDC.put("amount", amount.toString());
log.info("Order created");
MDC.clear();
```

輸出:
```json
{
  "timestamp": "2023-12-01T10:00:00.000Z",
  "level": "INFO",
  "message": "Order created",
  "userId": "12345",
  "orderId": "98765",
  "amount": "1000",
  "traceId": "abc123",
  "spanId": "def456"
}
```

### 2. 日誌級別

```java
@Service
public class OrderService {
    
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    public Order createOrder(OrderDTO dto) {
        // DEBUG:詳細除錯資訊 (生產環境關閉)
        log.debug("Creating order with details: {}", dto);
        
        // INFO:重要業務流程
        log.info("Order creation started for user {}", dto.getUserId());
        
        try {
            // 業務邏輯
            Order order = processOrder(dto);
            
            // INFO:成功資訊
            log.info("Order {} created successfully, amount: {}", 
                order.getId(), order.getTotalAmount());
            
            return order;
            
        } catch (BusinessException e) {
            // WARN:業務例外 (預期內的錯誤)
            log.warn("Order creation failed: {}", e.getMessage());
            throw e;
            
        } catch (Exception e) {
            // ERROR:系統錯誤 (非預期)
            log.error("Unexpected error creating order", e);
            throw new SystemException("System error", e);
        }
    }
}
```

級別選擇:
- **TRACE**:最詳細,通常不用
- **DEBUG**:開發除錯,生產關閉
- **INFO**:重要業務流程
- **WARN**:警告,不影響功能
- **ERROR**:錯誤,需要處理

### 3. 追蹤 ID (Trace ID)

```java
@Component
public class TraceIdFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        // 從請求頭取得或產生新的 Trace ID
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }
        
        // 放入 MDC
        MDC.put("traceId", traceId);
        
        // 加入響應頭
        response.addHeader("X-Trace-Id", traceId);
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

### 4. Logback 配置

```xml
<!-- logback-spring.xml -->
<configuration>
    <!-- 控制台輸出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- JSON 格式 -->
            <includeMdc>true</includeMdc>
            <includeContext>false</includeContext>
        </encoder>
    </appender>
    
    <!-- 檔案輸出 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每日輪轉 -->
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <!-- 保留 30 天 -->
            <maxHistory>30</maxHistory>
            <!-- 單檔最大 100MB -->
            <timeBasedFileNamingAndTriggeringPolicy 
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdc>true</includeMdc>
        </encoder>
    </appender>
    
    <!-- 非同步輸出 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="FILE"/>
    </appender>
    
    <!-- 根日誌 -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC_FILE"/>
    </root>
    
    <!-- 自訂日誌級別 -->
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

### 5. 業務日誌 vs. 技術日誌

```java
@Service
public class OrderService {
    
    // 技術日誌:應用程式行為
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    // 業務日誌:業務操作記錄
    private static final Logger auditLog = LoggerFactory.getLogger("AUDIT");
    
    @Autowired
    private AuditLogRepository auditLogRepository;
    
    public Order createOrder(OrderDTO dto) {
        log.info("Creating order for user {}", dto.getUserId());
        
        Order order = processOrder(dto);
        
        // 業務審計日誌:寫入資料庫
        AuditLog audit = AuditLog.builder()
            .userId(dto.getUserId())
            .action("CREATE_ORDER")
            .orderId(order.getId())
            .amount(order.getTotalAmount())
            .timestamp(LocalDateTime.now())
            .ipAddress(getClientIp())
            .build();
        auditLogRepository.save(audit);
        
        // 業務日誌:寫入檔案 (備份)
        auditLog.info("USER_ACTION|userId={}|action=CREATE_ORDER|orderId={}|amount={}",
            dto.getUserId(), order.getId(), order.getTotalAmount());
        
        return order;
    }
}
```

## 二、ELK Stack 日誌收集

### 1. 架構

```
應用程式 → Filebeat → Logstash → Elasticsearch → Kibana
```

### 2. Filebeat 配置

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/app/*.log
  json.keys_under_root: true
  json.add_error_key: true
  
  fields:
    app: product-service
    env: production
    
output.logstash:
  hosts: ["logstash:5044"]
```

### 3. Logstash 配置

```ruby
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # 解析 JSON
  json {
    source => "message"
  }
  
  # 提取時間戳
  date {
    match => ["timestamp", "ISO8601"]
  }
  
  # 過濾敏感資訊
  mutate {
    remove_field => ["password", "token"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
}
```

### 4. Kibana 查詢

```
# 查詢所有錯誤日誌
level:ERROR

# 查詢特定使用者的日誌
userId:12345

# 查詢特定時間範圍
timestamp:[2023-12-01 TO 2023-12-02]

# 複雜查詢
level:ERROR AND userId:12345 AND message:*order*

# 追蹤完整請求鏈
traceId:abc123
```

## 三、分散式追蹤 (Distributed Tracing)

### 1. Spring Cloud Sleuth + Zipkin

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
  sleuth:
    sampler:
      probability: 1.0  # 100% 取樣 (生產環境建議 0.1)
  zipkin:
    base-url: http://zipkin:9411
```

### 2. 自動追蹤

```java
@Service
public class OrderService {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    // Sleuth 自動產生 Trace 和 Span
    public Order createOrder(OrderDTO dto) {
        // Span 1: Get product info
        Product product = productService.getProduct(dto.getProductId());
        
        // Span 2: Check inventory
        boolean hasStock = inventoryService.checkStock(
            dto.getProductId(), 
            dto.getQuantity()
        );
        
        // Span 3: Process payment
        boolean paid = paymentService.pay(dto);
        
        // 所有 Span 在 Zipkin 可視化顯示
        return createOrderEntity(dto, product);
    }
}
```

### 3. 自訂 Span

```java
@Service
public class OrderService {
    
    @Autowired
    private Tracer tracer;
    
    public void processOrder(Order order) {
        // 建立自訂 Span
        Span span = tracer.nextSpan().name("process-order").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // 添加標籤
            span.tag("orderId", order.getId().toString());
            span.tag("amount", order.getTotalAmount().toString());
            
            // 業務邏輯
            doProcessOrder(order);
            
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 4. Zipkin UI

```
Trace 視圖:
[Gateway] ─────────────────────────── 500ms
  └─ [Order Service] ──────────────── 450ms
       ├─ [Product Service] ───────── 100ms
       ├─ [Inventory Service] ──────  50ms
       └─ [Payment Service] ───────── 300ms  ← 慢!
```

一眼看出是支付服務慢!

## 四、Prometheus + Grafana 監控

### 1. Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: 
        - 'product-service:8080'
        - 'order-service:8080'
```

### 2. Spring Boot Metrics

```java
@Component
public class BusinessMetrics {
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    private final Gauge orderGauge;
    private final DistributionSummary orderAmountSummary;
    
    public BusinessMetrics(MeterRegistry registry) {
        // 計數器:累計訂單數
        this.orderCounter = Counter.builder("orders.created.total")
            .description("Total orders created")
            .tag("status", "success")
            .register(registry);
        
        // 計時器:訂單處理時間
        this.orderTimer = Timer.builder("order.processing.time")
            .description("Order processing time")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        // 儀表:當前待處理訂單
        this.orderGauge = Gauge.builder("orders.pending", 
            this, BusinessMetrics::getPendingOrderCount)
            .description("Pending orders count")
            .register(registry);
        
        // 分佈摘要:訂單金額分佈
        this.orderAmountSummary = DistributionSummary.builder("order.amount")
            .description("Order amount distribution")
            .baseUnit("dollars")
            .register(registry);
    }
    
    public void recordOrderCreated(Order order) {
        orderCounter.increment();
        orderAmountSummary.record(order.getTotalAmount().doubleValue());
    }
    
    public void recordOrderProcessingTime(Runnable task) {
        orderTimer.record(task);
    }
    
    private double getPendingOrderCount() {
        // 查詢資料庫
        return orderRepository.countByStatus(OrderStatus.PENDING);
    }
}
```

### 3. Prometheus 查詢 (PromQL)

```promql
# QPS (每秒請求數)
rate(http_server_requests_seconds_count[5m])

# 錯誤率
rate(http_server_requests_seconds_count{status=~"5.."}[5m]) 
/ 
rate(http_server_requests_seconds_count[5m])

# P95 響應時間
histogram_quantile(0.95, 
  rate(http_server_requests_seconds_bucket[5m])
)

# CPU 使用率
process_cpu_usage

# 記憶體使用率
jvm_memory_used_bytes / jvm_memory_max_bytes

# 每秒訂單數
rate(orders_created_total[1m])

# 平均訂單金額
rate(order_amount_sum[5m]) / rate(order_amount_count[5m])
```

### 4. Grafana Dashboard

```json
{
  "dashboard": {
    "title": "E-commerce Monitoring",
    "panels": [
      {
        "title": "QPS",
        "targets": [{
          "expr": "rate(http_server_requests_seconds_count[5m])"
        }]
      },
      {
        "title": "Error Rate",
        "targets": [{
          "expr": "rate(http_server_requests_seconds_count{status=~\"5..\"}[5m]) / rate(http_server_requests_seconds_count[5m])"
        }],
        "alert": {
          "conditions": [{
            "evaluator": {"params": [0.01], "type": "gt"},
            "query": {"params": ["A", "5m", "now"]}
          }]
        }
      },
      {
        "title": "Response Time (P95)",
        "targets": [{
          "expr": "histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))"
        }]
      },
      {
        "title": "Orders per Second",
        "targets": [{
          "expr": "rate(orders_created_total[1m])"
        }]
      }
    ]
  }
}
```

## 五、應用性能監控 (APM)

### 1. Pinpoint

```
優點:
- 無侵入 (Java Agent)
- 自動追蹤方法呼叫
- 實時應用拓撲圖
- SQL 執行分析

啟動:
java -javaagent:pinpoint-bootstrap.jar \
     -Dpinpoint.agentId=product-service-01 \
     -Dpinpoint.applicationName=product-service \
     -jar app.jar
```

### 2. SkyWalking

```yaml
# agent/config/agent.config
agent.service_name=product-service
agent.namespace=production
collector.backend_service=skywalking-oap:11800
```

功能:
- 服務拓撲圖
- 追蹤分析
- 服務指標
- JVM 監控
- 日誌關聯

## 六、告警策略

### 1. 告警級別

```
P0 (緊急):服務完全不可用
  → 立即打電話給 On-call

P1 (嚴重):核心功能受影響
  → 30 分鐘內響應

P2 (警告):非核心功能問題
  → 2 小時內響應

P3 (資訊):潛在問題
  → 工作時間處理
```

### 2. 告警規則

```yaml
# Prometheus Alert Rules
groups:
- name: critical
  rules:
  # P0:服務掛了
  - alert: ServiceDown
    expr: up == 0
    for: 1m
    labels:
      severity: P0
    annotations:
      summary: "服務不可用: {{ $labels.instance }}"
  
  # P1:錯誤率過高
  - alert: HighErrorRate
    expr: |
      rate(http_server_requests_seconds_count{status=~"5.."}[5m]) 
      / 
      rate(http_server_requests_seconds_count[5m]) > 0.05
    for: 5m
    labels:
      severity: P1
    annotations:
      summary: "錯誤率過高 (>5%)"
  
  # P2:響應慢
  - alert: SlowResponse
    expr: |
      histogram_quantile(0.95, 
        rate(http_server_requests_seconds_bucket[5m])
      ) > 2
    for: 10m
    labels:
      severity: P2
    annotations:
      summary: "響應時間過慢 (P95 > 2s)"
```

### 3. 告警抑制

```yaml
# 避免告警風暴
inhibit_rules:
  # 服務掛了就不用再告警慢了
  - source_match:
      alertname: ServiceDown
    target_match:
      alertname: SlowResponse
    equal: ['instance']
```

## 七、監控大盤

### 1. RED 方法

```
Rate (請求速率):每秒請求數
Errors (錯誤率):失敗請求比例
Duration (響應時間):請求處理時間
```

### 2. USE 方法 (資源監控)

```
Utilization (使用率):CPU、記憶體使用率
Saturation (飽和度):等待佇列長度
Errors (錯誤):硬體錯誤
```

### 3. 業務監控

```
- 每分鐘訂單數
- 轉換率 (訂單/訪問)
- 平均訂單金額
- 支付成功率
- 庫存預警
```

## 小結

監控與日誌實踐經驗:
- **結構化日誌**:JSON 格式,易於搜尋
- **Trace ID**:追蹤完整請求鏈
- **ELK Stack**:集中日誌管理
- **分散式追蹤**:定位微服務瓶頸
- **Prometheus + Grafana**:實時監控
- **分級告警**:避免告警疲勞
- **業務監控**:關注核心指標

好的監控 = 快速定位問題 = 減少故障時間!

下週我們將進行**系列總結**,回顧這半年的學習旅程!
