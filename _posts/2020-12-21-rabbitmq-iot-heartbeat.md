---
layout: post
title: "RabbitMQ 實戰：處理 IoT 裝置 Heartbeat 的經驗分享"
date: 2020-12-21 09:15:00 +0800
categories: [Message Queue, IoT]
tags: [RabbitMQ, Message Queue, IoT, AMQP, Java]
---

## 前言

最近接手了一個 IoT 專案，需要處理數千個物聯網裝置的 heartbeat（心跳）訊息。這些裝置每 30 秒會發送一次狀態資訊，包含裝置 ID、時間戳記、電池電量、訊號強度等資料。

初期評估了幾個訊息佇列方案：
- **Kafka**：適合高吞吐量的串流處理，但對我們來說太重量級
- **Redis Pub/Sub**：輕量但缺乏持久化和確認機制
- **RabbitMQ**：功能完整、支援多種協定、有完善的訊息確認機制

最後選擇了 RabbitMQ，因為它在可靠性和易用性之間取得了良好的平衡。

## RabbitMQ 核心概念

在開始之前，先理解幾個關鍵概念：

### Producer（生產者）
發送訊息的應用程式。在我們的案例中，是接收裝置 heartbeat 的 API Gateway。

### Consumer（消費者）
接收並處理訊息的應用程式。我們有多個消費者處理不同的任務：
- 更新裝置狀態到資料庫
- 檢查裝置是否離線
- 收集統計資料

### Exchange（交換器）
接收訊息並路由到佇列。有四種類型：
- **Direct**：根據 routing key 精確匹配
- **Topic**：根據 routing key 模式匹配（支援萬用字元）
- **Fanout**：廣播到所有綁定的佇列
- **Headers**：根據訊息標頭路由

### Queue（佇列）
儲存訊息，等待消費者處理。

### Binding（綁定）
Exchange 和 Queue 之間的關聯規則。

## 環境安裝

我在 EC2 上安裝 RabbitMQ：

```bash
# 更新系統
sudo yum update -y

# 安裝 Erlang（RabbitMQ 的執行環境）
sudo yum install erlang -y

# 下載並安裝 RabbitMQ
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.9.11/rabbitmq-server-3.9.11-1.el7.noarch.rpm
sudo rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
sudo yum install rabbitmq-server-3.9.11-1.el7.noarch.rpm -y

# 啟動 RabbitMQ
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server

# 啟用管理介面
sudo rabbitmq-plugins enable rabbitmq_management

# 建立管理員帳號
sudo rabbitmqctl add_user admin StrongPassword123
sudo rabbitmqctl set_user_tags admin administrator
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

管理介面可以透過 `http://ec2-ip:15672` 存取。

## 架構設計

針對 IoT heartbeat 的需求，我設計了以下架構：

```
IoT Devices 
    ↓
API Gateway (Producer)
    ↓
RabbitMQ Exchange (Topic)
    ├─→ Queue: heartbeat.process (更新裝置狀態)
    ├─→ Queue: heartbeat.monitor (離線監控)
    └─→ Queue: heartbeat.analytics (數據分析)
```

使用 **Topic Exchange** 的原因：
- 可以根據裝置類型、區域等條件路由訊息
- 彈性高，未來可以輕鬆新增新的消費者

### Routing Key 設計

```
heartbeat.{device_type}.{region}.{priority}
```

範例：
- `heartbeat.sensor.taipei.normal`
- `heartbeat.gateway.kaohsiung.high`
- `heartbeat.camera.taichung.normal`

## Producer 實作（Java）

使用 Spring Boot + Spring AMQP：

### 1. 加入依賴

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 2. 設定 RabbitMQ

```yaml
# application.yml
spring:
  rabbitmq:
    host: your-rabbitmq-host
    port: 5672
    username: admin
    password: StrongPassword123
    virtual-host: /
    connection-timeout: 15000
```

### 3. RabbitMQ 配置類

```java
@Configuration
public class RabbitMQConfig {
    
    public static final String EXCHANGE_NAME = "heartbeat.exchange";
    
    @Bean
    public TopicExchange heartbeatExchange() {
        return new TopicExchange(EXCHANGE_NAME, true, false);
    }
    
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter());
        return template;
    }
}
```

### 4. Heartbeat 資料模型

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class HeartbeatMessage {
    private String deviceId;
    private String deviceType;  // sensor, gateway, camera
    private String region;      // taipei, kaohsiung, taichung
    private LocalDateTime timestamp;
    private Integer batteryLevel;  // 0-100
    private Integer signalStrength; // dBm
    private String status;      // online, warning, critical
    private Map<String, Object> metadata;
}
```

### 5. Producer 服務

```java
@Service
@Slf4j
public class HeartbeatProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendHeartbeat(HeartbeatMessage heartbeat) {
        String routingKey = String.format("heartbeat.%s.%s.%s",
            heartbeat.getDeviceType(),
            heartbeat.getRegion(),
            getPriority(heartbeat)
        );
        
        try {
            rabbitTemplate.convertAndSend(
                RabbitMQConfig.EXCHANGE_NAME,
                routingKey,
                heartbeat
            );
            log.debug("Heartbeat sent: deviceId={}, routingKey={}", 
                heartbeat.getDeviceId(), routingKey);
        } catch (AmqpException e) {
            log.error("Failed to send heartbeat: {}", heartbeat.getDeviceId(), e);
            // 這裡可以實作重試邏輯或將訊息存到本地佇列
        }
    }
    
    private String getPriority(HeartbeatMessage heartbeat) {
        if ("critical".equals(heartbeat.getStatus()) || 
            heartbeat.getBatteryLevel() < 20) {
            return "high";
        }
        return "normal";
    }
}
```

### 6. REST API 端點

```java
@RestController
@RequestMapping("/api/iot")
@Slf4j
public class IoTController {
    
    @Autowired
    private HeartbeatProducer heartbeatProducer;
    
    @PostMapping("/heartbeat")
    public ResponseEntity<Map<String, String>> receiveHeartbeat(
            @RequestBody HeartbeatMessage heartbeat) {
        
        log.info("Received heartbeat from device: {}", heartbeat.getDeviceId());
        
        // 驗證資料
        if (heartbeat.getDeviceId() == null || heartbeat.getTimestamp() == null) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "Missing required fields"));
        }
        
        // 發送到 RabbitMQ
        heartbeatProducer.sendHeartbeat(heartbeat);
        
        return ResponseEntity.ok(Map.of("status", "accepted"));
    }
}
```

## Consumer 實作（Java）

### 1. 佇列配置

```java
@Configuration
public class ConsumerConfig {
    
    @Bean
    public Queue processQueue() {
        return QueueBuilder.durable("heartbeat.process")
            .withArgument("x-message-ttl", 3600000)  // 1 小時過期
            .build();
    }
    
    @Bean
    public Queue monitorQueue() {
        return QueueBuilder.durable("heartbeat.monitor")
            .withArgument("x-message-ttl", 3600000)
            .build();
    }
    
    @Bean
    public Binding processBinding(Queue processQueue, TopicExchange heartbeatExchange) {
        return BindingBuilder
            .bind(processQueue)
            .to(heartbeatExchange)
            .with("heartbeat.#");  // 接收所有 heartbeat 訊息
    }
    
    @Bean
    public Binding monitorBinding(Queue monitorQueue, TopicExchange heartbeatExchange) {
        return BindingBuilder
            .bind(monitorQueue)
            .to(heartbeatExchange)
            .with("heartbeat.*.*.high");  // 只接收高優先級訊息
    }
}
```

### 2. Consumer 服務

```java
@Service
@Slf4j
public class HeartbeatConsumer {
    
    @Autowired
    private DeviceService deviceService;
    
    @RabbitListener(queues = "heartbeat.process", concurrency = "3-5")
    public void processHeartbeat(HeartbeatMessage heartbeat, 
                                 @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey) {
        
        log.info("Processing heartbeat: deviceId={}, routingKey={}", 
            heartbeat.getDeviceId(), routingKey);
        
        try {
            // 更新裝置狀態到資料庫
            deviceService.updateDeviceStatus(
                heartbeat.getDeviceId(),
                heartbeat.getTimestamp(),
                heartbeat.getBatteryLevel(),
                heartbeat.getSignalStrength(),
                heartbeat.getStatus()
            );
            
            log.debug("Device status updated: {}", heartbeat.getDeviceId());
            
        } catch (Exception e) {
            log.error("Failed to process heartbeat: {}", heartbeat.getDeviceId(), e);
            throw new AmqpRejectAndDontRequeueException("Processing failed", e);
        }
    }
    
    @RabbitListener(queues = "heartbeat.monitor", concurrency = "2-3")
    public void monitorHeartbeat(HeartbeatMessage heartbeat) {
        
        log.warn("High priority heartbeat received: deviceId={}, status={}", 
            heartbeat.getDeviceId(), heartbeat.getStatus());
        
        try {
            // 檢查裝置狀態
            if ("critical".equals(heartbeat.getStatus())) {
                // 發送告警通知
                alertService.sendCriticalAlert(heartbeat);
            }
            
            if (heartbeat.getBatteryLevel() < 20) {
                // 發送低電量通知
                alertService.sendLowBatteryAlert(heartbeat);
            }
            
        } catch (Exception e) {
            log.error("Failed to monitor heartbeat: {}", heartbeat.getDeviceId(), e);
        }
    }
}
```

### 3. 錯誤處理與重試

```java
@Configuration
public class RabbitMQErrorConfig {
    
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        
        SimpleRabbitListenerContainerFactory factory = 
            new SimpleRabbitListenerContainerFactory();
        
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(new Jackson2JsonMessageConverter());
        
        // 設定重試機制
        factory.setAdviceChain(retryInterceptor());
        
        // 設定並發消費者數量
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);
        
        // 設定預取數量（每次拉取多少訊息）
        factory.setPrefetchCount(10);
        
        return factory;
    }
    
    @Bean
    public RetryOperationsInterceptor retryInterceptor() {
        return RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000)  // 初始延遲 1s，乘數 2.0，最大 10s
            .recoverer(new RejectAndDontRequeueRecoverer())
            .build();
    }
}
```

## 監控與管理

### 1. 使用 Management Plugin

透過瀏覽器存取 `http://rabbitmq-host:15672`，可以看到：
- 連線數量
- 佇列長度
- 訊息發送/接收速率
- 消費者數量

### 2. 程式化監控

使用 RabbitMQ HTTP API：

```java
@Service
@Slf4j
public class RabbitMQMonitorService {
    
    @Value("${spring.rabbitmq.host}")
    private String rabbitHost;
    
    @Value("${spring.rabbitmq.username}")
    private String username;
    
    @Value("${spring.rabbitmq.password}")
    private String password;
    
    public QueueInfo getQueueInfo(String queueName) {
        RestTemplate restTemplate = new RestTemplate();
        
        String url = String.format("http://%s:15672/api/queues/%%2f/%s", 
            rabbitHost, queueName);
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBasicAuth(username, password);
        
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        try {
            ResponseEntity<QueueInfo> response = restTemplate.exchange(
                url, HttpMethod.GET, entity, QueueInfo.class);
            
            return response.getBody();
            
        } catch (Exception e) {
            log.error("Failed to get queue info: {}", queueName, e);
            return null;
        }
    }
    
    @Data
    public static class QueueInfo {
        private String name;
        private Integer messages;
        private Integer messagesReady;
        private Integer messagesUnacknowledged;
        private Integer consumers;
    }
}
```

### 3. 定期檢查佇列積壓

```java
@Component
@Slf4j
public class QueueHealthChecker {
    
    @Autowired
    private RabbitMQMonitorService monitorService;
    
    @Scheduled(fixedRate = 60000)  // 每分鐘檢查一次
    public void checkQueueHealth() {
        String[] queues = {"heartbeat.process", "heartbeat.monitor"};
        
        for (String queueName : queues) {
            RabbitMQMonitorService.QueueInfo info = 
                monitorService.getQueueInfo(queueName);
            
            if (info != null && info.getMessages() > 10000) {
                log.warn("Queue {} has {} messages pending", 
                    queueName, info.getMessages());
                // 可以發送告警通知
            }
        }
    }
}
```

## 效能調校

### 1. 調整 Prefetch Count

預取數量影響吞吐量：
- **過小**：頻繁的網路往返，效能差
- **過大**：訊息分配不均，某些消費者閒置

根據測試，我設定為 10-20 之間。

```java
factory.setPrefetchCount(10);
```

### 2. 調整並發消費者

```java
@RabbitListener(
    queues = "heartbeat.process",
    concurrency = "5-10"  // 最少 5 個，最多 10 個消費者
)
```

### 3. 批次確認

```java
factory.setAcknowledgeMode(AcknowledgeMode.AUTO);
// 或手動批次確認
```

### 4. 訊息持久化 vs 效能

權衡考量：
- **持久化**：訊息不會遺失，但效能較差
- **非持久化**：效能好，但重啟後訊息會遺失

對於 heartbeat 這種可以容忍少量遺失的資料，我選擇非持久化：

```java
@Bean
public TopicExchange heartbeatExchange() {
    return new TopicExchange(EXCHANGE_NAME, false, false);  // durable = false
}
```

## 處理裝置離線偵測

除了處理 heartbeat，還需要偵測裝置是否離線：

```java
@Service
@Slf4j
public class DeviceOfflineDetector {
    
    @Autowired
    private DeviceRepository deviceRepository;
    
    @Autowired
    private AlertService alertService;
    
    // 每 5 分鐘檢查一次
    @Scheduled(fixedRate = 300000)
    public void detectOfflineDevices() {
        LocalDateTime threshold = LocalDateTime.now().minusMinutes(2);
        
        List<Device> offlineDevices = deviceRepository
            .findByLastHeartbeatBefore(threshold);
        
        log.info("Found {} offline devices", offlineDevices.size());
        
        for (Device device : offlineDevices) {
            if (!"offline".equals(device.getStatus())) {
                log.warn("Device {} went offline", device.getDeviceId());
                
                // 更新狀態
                device.setStatus("offline");
                deviceRepository.save(device);
                
                // 發送告警
                alertService.sendOfflineAlert(device);
            }
        }
    }
}
```

## 遇到的問題與解決

### 問題 1：訊息積壓

**現象**：佇列長度快速增長，消費速度跟不上

**原因**：
1. 消費者處理速度太慢（資料庫寫入瓶頸）
2. 消費者數量不足

**解決**：
1. 增加消費者數量
2. 優化資料庫寫入（批次寫入）
3. 使用連線池

### 問題 2：重複消費

**現象**：同一個 heartbeat 被處理多次

**原因**：消費者處理完但來不及確認，RabbitMQ 將訊息重新分配

**解決**：
1. 增加 acknowledgement timeout
2. 使用冪等性設計（upsert 而非 insert）
3. 使用唯一鍵防止重複

```java
// 使用 ON DUPLICATE KEY UPDATE
deviceService.upsertDeviceStatus(heartbeat);
```

### 問題 3：記憶體不足

**現象**：RabbitMQ 記憶體使用率過高，甚至 OOM

**原因**：佇列積壓太多訊息

**解決**：
1. 設定訊息 TTL（過期自動刪除）
2. 設定佇列最大長度
3. 增加記憶體限制

```bash
# 設定記憶體高水位線
sudo rabbitmqctl set_vm_memory_high_watermark 0.6
```

## 實際效能數據

經過調校後的系統表現：

- **訊息吞吐量**：每秒 5,000 條訊息
- **平均延遲**：50-100ms（從發送到消費）
- **CPU 使用率**：30-40%
- **記憶體使用**：2-3 GB
- **裝置數量**：3,000 台
- **Heartbeat 間隔**：30 秒

## 下一步優化

1. **引入 Dead Letter Exchange**：處理失敗的訊息
2. **實作訊息追蹤**：記錄訊息處理歷程
3. **整合 Prometheus + Grafana**：更完整的監控
4. **評估 Cluster 部署**：提高可用性

## 總結

RabbitMQ 對於處理 IoT heartbeat 這類場景非常適合：
- 支援多種路由模式
- 訊息確認機制可靠
- 管理介面直觀
- 社群活躍、文件完整

相比直接呼叫 API 更新資料庫，使用訊息佇列的好處：
- **解耦**：API 和資料庫處理分離
- **緩衝**：應付流量尖峰
- **擴展性**：輕鬆增加消費者
- **可靠性**：訊息不會因暫時性錯誤而遺失

對於 Java 背景的工程師，Spring AMQP 提供了很好的抽象，大幅降低了使用門檻。如果你也在處理類似的場景，強烈推薦試試 RabbitMQ。

下一篇計畫分享 AWS Step Functions，看看如何用它來編排複雜的工作流程。

## 參考資料

- [RabbitMQ 官方文件](https://www.rabbitmq.com/documentation.html)
- [Spring AMQP 參考文件](https://docs.spring.io/spring-amqp/reference/)
- [RabbitMQ Best Practices](https://www.cloudamqp.com/blog/part1-rabbitmq-best-practice.html)
