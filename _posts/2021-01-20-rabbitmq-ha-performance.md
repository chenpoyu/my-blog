---
layout: post
title: "RabbitMQ 進階：高可用與效能優化實戰"
date: 2021-01-20 09:40:00 +0800
categories: [Message Queue, Architecture]
tags: [RabbitMQ, High Availability, Performance, Cluster]
---

## 前言

自從上個月開始使用 RabbitMQ 處理 IoT heartbeat 後，系統運作穩定。但隨著裝置數量從 3,000 台增長到 8,000 台，開始遇到一些挑戰：

1. **單點故障風險**：RabbitMQ 只有一個節點，如果掛掉整個系統會停擺
2. **效能瓶頸**：尖峰時段訊息處理速度跟不上
3. **記憶體壓力**：佇列積壓導致記憶體使用率飆高

這篇文章記錄我如何解決這些問題，建立高可用的 RabbitMQ 架構。

## 單機效能調校

在考慮 Cluster 之前，先確保單機效能已經優化到極致。

### 調整連線和通道數

```java
@Bean
public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    factory.setHost(rabbitHost);
    factory.setPort(rabbitPort);
    factory.setUsername(username);
    factory.setPassword(password);
    
    // 連線池設定
    factory.setChannelCacheSize(50);  // 快取的通道數
    factory.setChannelCheckoutTimeout(5000);  // 取得通道的逾時時間
    
    // 連線設定
    factory.setConnectionTimeout(30000);
    factory.setRequestedHeartBeat(30);
    
    return factory;
}
```

### 調整 Prefetch Count

這是最重要的效能參數之一：

```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory) {
    
    SimpleRabbitListenerContainerFactory factory = 
        new SimpleRabbitListenerContainerFactory();
    
    factory.setConnectionFactory(connectionFactory);
    factory.setPrefetchCount(20);  // 每個消費者預取 20 條訊息
    factory.setConcurrentConsumers(5);  // 初始消費者數量
    factory.setMaxConcurrentConsumers(10);  // 最大消費者數量
    
    return factory;
}
```

**Prefetch Count 調校原則**：
- 訊息處理時間短（< 100ms）：設定 50-100
- 訊息處理時間中等（100ms - 1s）：設定 10-20
- 訊息處理時間長（> 1s）：設定 1-5

### 批次處理

將多個訊息合併處理，減少資料庫連線次數：

```java
@Service
@Slf4j
public class BatchHeartbeatConsumer {
    
    private final List<HeartbeatMessage> buffer = new CopyOnWriteArrayList<>();
    private final int BATCH_SIZE = 100;
    private final int BATCH_TIMEOUT_MS = 5000;
    
    @Autowired
    private DeviceService deviceService;
    
    @RabbitListener(queues = "heartbeat.process")
    public void consumeHeartbeat(HeartbeatMessage heartbeat) {
        buffer.add(heartbeat);
        
        if (buffer.size() >= BATCH_SIZE) {
            flush();
        }
    }
    
    @Scheduled(fixedDelay = 5000)
    public void flushPeriodically() {
        if (!buffer.isEmpty()) {
            flush();
        }
    }
    
    private synchronized void flush() {
        if (buffer.isEmpty()) {
            return;
        }
        
        List<HeartbeatMessage> batch = new ArrayList<>(buffer);
        buffer.clear();
        
        try {
            deviceService.batchUpdateStatus(batch);
            log.info("Flushed {} messages", batch.size());
        } catch (Exception e) {
            log.error("Failed to flush batch", e);
            // 重新加入 buffer 或寫入 DLQ
        }
    }
}
```

### RabbitMQ 設定檔調校

編輯 `/etc/rabbitmq/rabbitmq.conf`：

```conf
# TCP 連線設定
tcp_listen_options.backlog = 128
tcp_listen_options.nodelay = true
tcp_listen_options.linger.on = true
tcp_listen_options.linger.timeout = 0

# 記憶體設定
vm_memory_high_watermark.relative = 0.6
vm_memory_high_watermark_paging_ratio = 0.75

# 磁碟空間設定
disk_free_limit.absolute = 2GB

# 訊息大小限制
max_message_size = 134217728

# 通道數量限制
channel_max = 2048
```

重啟 RabbitMQ：

```bash
sudo systemctl restart rabbitmq-server
```

## RabbitMQ Cluster 高可用架構

單機調校有極限，要達到真正的高可用需要建立 Cluster。

### Cluster 架構設計

我建立了 3 節點的 Cluster：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ RabbitMQ-1  │────│ RabbitMQ-2  │────│ RabbitMQ-3  │
│ 10.0.1.10   │     │ 10.0.1.11   │     │ 10.0.1.12   │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                          │
                   ┌──────────────┐
                   │ Load Balancer│
                   └──────────────┘
```

### 節點準備

在三台 EC2 上分別安裝 RabbitMQ，然後設定主機名稱：

**節點 1**：
```bash
sudo hostnamectl set-hostname rabbitmq-1
echo "10.0.1.10 rabbitmq-1" | sudo tee -a /etc/hosts
echo "10.0.1.11 rabbitmq-2" | sudo tee -a /etc/hosts
echo "10.0.1.12 rabbitmq-3" | sudo tee -a /etc/hosts
```

**節點 2 和 3**：同樣設定，修改對應的主機名稱。

### 同步 Erlang Cookie

RabbitMQ 節點透過 Erlang Cookie 認證，必須相同：

```bash
# 在節點 1
sudo cat /var/lib/rabbitmq/.erlang.cookie

# 複製 cookie 內容，在節點 2 和 3 設定
sudo sh -c 'echo "ABCDEFGHIJKLMNOPQRSTUVWXYZ" > /var/lib/rabbitmq/.erlang.cookie'
sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie
sudo chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

# 重啟 RabbitMQ
sudo systemctl restart rabbitmq-server
```

### 加入 Cluster

在節點 2：

```bash
sudo rabbitmqctl stop_app
sudo rabbitmqctl join_cluster rabbit@rabbitmq-1
sudo rabbitmqctl start_app
```

在節點 3：

```bash
sudo rabbitmqctl stop_app
sudo rabbitmqctl join_cluster rabbit@rabbitmq-1
sudo rabbitmqctl start_app
```

驗證 Cluster 狀態：

```bash
sudo rabbitmqctl cluster_status
```

輸出：

```
Cluster status of node rabbit@rabbitmq-1 ...
Basics

Cluster name: rabbit@rabbitmq-1

Disk Nodes

rabbit@rabbitmq-1
rabbit@rabbitmq-2
rabbit@rabbitmq-3

Running Nodes

rabbit@rabbitmq-1
rabbit@rabbitmq-2
rabbit@rabbitmq-3
```

## 佇列鏡像（Queue Mirroring）

預設情況下，佇列只存在於一個節點。啟用鏡像後，佇列會複製到其他節點。

### 設定 HA Policy

```bash
sudo rabbitmqctl set_policy ha-all "^heartbeat\." \
  '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}' \
  --priority 1 \
  --apply-to queues
```

參數說明：
- **ha-mode**：`all`（所有節點）、`exactly`（指定數量）、`nodes`（指定節點）
- **ha-params**：與 ha-mode 搭配使用
- **ha-sync-mode**：`automatic`（自動同步）、`manual`（手動同步）

查看 Policy：

```bash
sudo rabbitmqctl list_policies
```

### 驗證鏡像

```bash
sudo rabbitmqctl list_queues name slave_pids synchronised_slave_pids
```

## 負載平衡

使用 HAProxy 分散連線到各節點：

### 安裝 HAProxy

```bash
sudo yum install haproxy -y
```

### 設定 HAProxy

編輯 `/etc/haproxy/haproxy.cfg`：

```conf
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    retries 3
    timeout connect 5000ms
    timeout client 600000ms
    timeout server 600000ms

# RabbitMQ AMQP
listen rabbitmq_amqp
    bind *:5672
    mode tcp
    balance roundrobin
    option tcpka
    server rabbitmq-1 10.0.1.10:5672 check inter 5s rise 2 fall 3
    server rabbitmq-2 10.0.1.11:5672 check inter 5s rise 2 fall 3
    server rabbitmq-3 10.0.1.12:5672 check inter 5s rise 2 fall 3

# RabbitMQ Management UI
listen rabbitmq_management
    bind *:15672
    mode http
    balance roundrobin
    option httpchk GET /api/healthchecks/node
    server rabbitmq-1 10.0.1.10:15672 check inter 5s rise 2 fall 3
    server rabbitmq-2 10.0.1.11:15672 check inter 5s rise 2 fall 3
    server rabbitmq-3 10.0.1.12:15672 check inter 5s rise 2 fall 3

# Stats
listen stats
    bind *:8080
    mode http
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:password123
```

啟動 HAProxy：

```bash
sudo systemctl start haproxy
sudo systemctl enable haproxy
```

### 應用程式連線設定

```yaml
spring:
  rabbitmq:
    addresses: haproxy-host:5672
    username: admin
    password: StrongPassword123
    virtual-host: /
    connection-timeout: 15000
    requested-heartbeat: 30
```

## 監控 Cluster

### 使用 Management Plugin

瀏覽器存取 `http://haproxy-host:15672`，可以看到：
- 各節點狀態
- 佇列分佈
- 訊息流量
- 連線數

### Prometheus + Grafana

更專業的監控方案：

#### 啟用 Prometheus Plugin

```bash
sudo rabbitmq-plugins enable rabbitmq_prometheus
```

#### Prometheus 設定

`prometheus.yml`：

```yaml
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets:
        - 'rabbitmq-1:15692'
        - 'rabbitmq-2:15692'
        - 'rabbitmq-3:15692'
```

#### 重要指標

- `rabbitmq_queue_messages`：佇列訊息數
- `rabbitmq_queue_messages_ready`：待處理訊息數
- `rabbitmq_queue_messages_unacknowledged`：未確認訊息數
- `rabbitmq_queue_consumers`：消費者數量
- `rabbitmq_node_mem_used`：記憶體使用量
- `rabbitmq_node_fd_used`：檔案描述符使用量

## 災難恢復測試

### 測試節點失效

模擬節點 2 故障：

```bash
# 在節點 2
sudo systemctl stop rabbitmq-server
```

觀察：
1. HAProxy 自動將流量導向節點 1 和 3
2. 鏡像佇列仍然可用
3. 應用程式無感知

節點恢復後：

```bash
sudo systemctl start rabbitmq-server
```

佇列會自動同步回來。

### 測試網路分區

模擬網路分區（Network Partition）：

```bash
# 在節點 2 封鎖與節點 1 的連線
sudo iptables -A INPUT -s 10.0.1.10 -j DROP
sudo iptables -A OUTPUT -d 10.0.1.10 -j DROP
```

RabbitMQ 會偵測到分區，預設行為是暫停節點。

設定自動處理策略：

```bash
sudo rabbitmqctl set_policy ha-all ".*" \
  '{"ha-mode":"all","ha-sync-mode":"automatic","ha-promote-on-shutdown":"always","ha-promote-on-failure":"always"}' \
  --priority 1 \
  --apply-to queues
```

恢復網路：

```bash
sudo iptables -F
```

## 效能測試

使用 RabbitMQ PerfTest 工具測試：

```bash
# 下載 PerfTest
wget https://github.com/rabbitmq/rabbitmq-perf-test/releases/download/v2.15.0/rabbitmq-perf-test-2.15.0.jar

# 執行測試
java -jar rabbitmq-perf-test-2.15.0.jar \
  --uri amqp://admin:password@haproxy-host:5672 \
  --producers 10 \
  --consumers 10 \
  --rate 1000 \
  --queue-pattern "test-%d" \
  --queue-pattern-from 1 \
  --queue-pattern-to 3 \
  --time 60
```

### 測試結果

單節點：
- 吞吐量：約 8,000 msg/s
- 平均延遲：20 ms
- 99th 百分位延遲：50 ms

3 節點 Cluster（有鏡像）：
- 吞吐量：約 6,000 msg/s（稍微下降，因為同步開銷）
- 平均延遲：25 ms
- 99th 百分位延遲：60 ms

雖然吞吐量稍降，但換來了高可用性，是值得的取捨。

## Dead Letter Exchange (DLX)

處理失敗訊息的機制：

```java
@Bean
public Queue heartbeatQueue() {
    return QueueBuilder.durable("heartbeat.process")
        .withArgument("x-message-ttl", 3600000)  // 1 小時過期
        .withArgument("x-dead-letter-exchange", "dlx.exchange")
        .withArgument("x-dead-letter-routing-key", "heartbeat.failed")
        .withArgument("x-max-length", 100000)  // 最多 10 萬條訊息
        .build();
}

@Bean
public Queue deadLetterQueue() {
    return QueueBuilder.durable("heartbeat.failed").build();
}

@Bean
public DirectExchange deadLetterExchange() {
    return new DirectExchange("dlx.exchange");
}

@Bean
public Binding deadLetterBinding() {
    return BindingBuilder
        .bind(deadLetterQueue())
        .to(deadLetterExchange())
        .with("heartbeat.failed");
}
```

處理 DLQ 的訊息：

```java
@RabbitListener(queues = "heartbeat.failed")
public void handleFailedMessage(Message message) {
    log.error("Failed message: {}", new String(message.getBody()));
    
    // 分析失敗原因
    Map<String, Object> headers = message.getMessageProperties().getHeaders();
    String reason = (String) headers.get("x-first-death-reason");
    
    // 記錄到資料庫或發送告警
    alertService.sendFailedMessageAlert(message, reason);
}
```

## 記憶體管理

### 設定記憶體警戒線

```bash
sudo rabbitmqctl set_vm_memory_high_watermark 0.5
```

當記憶體使用超過 50% 時，RabbitMQ 會：
1. 阻擋新的發布者
2. 開始將訊息寫入磁碟
3. 清除內部快取

### 監控記憶體使用

```bash
sudo rabbitmqctl status | grep memory
```

### Lazy Queue

對於不需要極致效能的佇列，使用 Lazy 模式：

```java
@Bean
public Queue lazyQueue() {
    return QueueBuilder.durable("heartbeat.archive")
        .withArgument("x-queue-mode", "lazy")
        .build();
}
```

Lazy Queue 會將大部分訊息寫入磁碟，減少記憶體使用。

## 實際部署經驗

### 從單節點遷移到 Cluster

1. **建立 Cluster**：在新的 EC2 上建立 3 節點 Cluster
2. **設定鏡像**：啟用 HA Policy
3. **設定 HAProxy**：導流到新 Cluster
4. **測試**：小流量測試
5. **切換**：逐步將流量從舊節點切到新 Cluster
6. **監控**：密切觀察指標
7. **下線舊節點**

整個過程約 2 小時，無停機時間。

### 維護程序

定期維護：
- **每週**：檢查 Cluster 狀態、清理 DLQ
- **每月**：檢視效能指標、調整參數
- **每季**：更新 RabbitMQ 版本、備份設定

## 遇到的坑

### 問題 1：節點間時鐘不同步

**現象**：Cluster 狀態異常，訊息遺失

**解決**：安裝 NTP 同步時鐘

```bash
sudo yum install chrony -y
sudo systemctl start chronyd
sudo systemctl enable chronyd
```

### 問題 2：Erlang OOM

**現象**：節點突然崩潰，記憶體耗盡

**原因**：佇列積壓過多訊息

**解決**：
1. 設定 `x-max-length` 限制佇列長度
2. 增加消費者數量
3. 調高記憶體限制

### 問題 3：腦裂（Split Brain）

**現象**：網路分區後，兩邊各自運作，資料不一致

**解決**：設定 `pause_minority` 模式

```bash
# 在 rabbitmq.conf
cluster_partition_handling = pause_minority
```

## 成本分析

3 節點 Cluster 的成本：

- **EC2**：3 × t3.medium = 3 × $30 = $90/月
- **EBS**：3 × 50GB = $15/月
- **流量**：可忽略（內網傳輸免費）

總計：約 $105/月

相比單節點（約 $35/月），成本增加 3 倍，但獲得：
- 高可用性
- 更好的效能
- 無單點故障

對於生產環境，絕對值得。

## 總結

建立 RabbitMQ Cluster 後，系統穩定性大幅提升：

**改善前**（單節點）：
- 可用性：約 99%（月停機時間 7 小時）
- 吞吐量：8,000 msg/s
- 單點故障風險

**改善後**（3 節點 Cluster）：
- 可用性：> 99.9%（月停機時間 < 1 小時）
- 吞吐量：6,000 msg/s（但可線性擴展）
- 無單點故障

RabbitMQ Cluster 的設定雖然有一定複雜度，但對於關鍵業務系統來說是必要的投資。

經過這三個月的實戰，對 AWS 和訊息佇列有了深入的理解。接下來計畫探索更多主題，例如容器化部署和微服務架構。

## 參考資料

- [RabbitMQ Clustering Guide](https://www.rabbitmq.com/clustering.html)
- [RabbitMQ HA Guide](https://www.rabbitmq.com/ha.html)
- [Production Checklist](https://www.rabbitmq.com/production-checklist.html)
