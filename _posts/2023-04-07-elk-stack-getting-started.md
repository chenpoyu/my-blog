---
layout: post
title: "ELK Stack 入門：搭建你的日誌分析平台"
date: 2023-04-07 09:00:00 +0800
categories: [Observability, Logging]
tags: [ELK Stack, Elasticsearch, Logstash, Kibana, Logging]
---

上週我們理解了 Metrics、Logs、Traces 的差異。從今天開始，我們要深入學習「日誌管理」的王者：**ELK Stack**。

這個名字常常聽到，但真正搞懂它的人不多。今天我們要從零開始，搭建一個完整的日誌分析平台。

## ELK Stack 是什麼？

ELK 是三個開源專案的縮寫：

- **E**lasticsearch：分散式搜尋引擎，儲存和搜尋日誌
- **L**ogstash：日誌收集和轉換工具
- **K**ibana：視覺化介面，類似 Grafana

後來又加入了 **Beats**（輕量級日誌收集器），所以現在也叫 **Elastic Stack**。

### 資料流向

```
┌──────────┐
│   App    │ 產生日誌
└─────┬────┘
      │ (寫入檔案或 stdout)
      ↓
┌──────────┐
│ Filebeat │ 收集日誌
└─────┬────┘
      │ (傳輸)
      ↓
┌──────────┐
│ Logstash │ 解析、過濾、轉換
└─────┬────┘
      │ (索引)
      ↓
┌──────────────┐
│Elasticsearch │ 儲存和搜尋
└─────┬────────┘
      │
      ↓
┌──────────┐
│  Kibana  │ 視覺化查詢
└──────────┘
```

## 快速安裝（Docker Compose）

廢話不多說，先把環境建起來。

建立 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false  # 開發環境暫時關閉安全性
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.6.0
    container_name: logstash
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    environment:
      - "LS_JAVA_OPTS=-Xms256m -Xmx256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.6.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk
    depends_on:
      - elasticsearch

volumes:
  es-data:

networks:
  elk:
    driver: bridge
```

建立 Logstash 設定檔 `logstash/pipeline/logstash.conf`：

```ruby
input {
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # 之後會加入過濾規則
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
  
  stdout {
    codec => rubydebug
  }
}
```

啟動：

```bash
docker-compose up -d
```

等待約 1 分鐘，然後檢查：

```bash
# Elasticsearch 是否正常
curl http://localhost:9200

# Kibana 是否可訪問
open http://localhost:5601
```

## 第一筆日誌：手動發送

用 `curl` 發送一筆 JSON 日誌：

```bash
echo '{"message":"Hello ELK", "level":"INFO", "timestamp":"2023-04-07T10:00:00Z"}' | \
  nc localhost 5000
```

然後在 Kibana 中：

1. 點選左側選單 **Discover**
2. 點選 **Create data view**
3. Index pattern: `app-logs-*`
4. Timestamp field: `@timestamp`
5. 點選 **Create data view**

你應該會看到剛剛發送的日誌！

## 從應用程式發送日誌

### Java + Logback

加入依賴（`pom.xml`）：

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.3</version>
</dependency>
```

設定 `logback-spring.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- Console 輸出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Logstash 輸出 -->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>localhost:5000</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"my-java-app","environment":"production"}</customFields>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="LOGSTASH" />
    </root>
</configuration>
```

在程式碼中：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@RestController
public class OrderController {
    private static final Logger log = LoggerFactory.getLogger(OrderController.class);
    
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        log.info("Creating order for user {}", request.getUserId());
        
        try {
            Order order = orderService.create(request);
            log.info("Order created successfully: {}", order.getId());
            return order;
        } catch (InsufficientInventoryException e) {
            log.error("Failed to create order: insufficient inventory", e);
            throw e;
        } catch (Exception e) {
            log.error("Failed to create order: unexpected error", e);
            throw e;
        }
    }
}
```

### .NET Core + Serilog

安裝 NuGet 套件：

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Network
```

設定 `Program.cs`：

```csharp
using Serilog;
using Serilog.Sinks.Network;

var builder = WebApplication.CreateBuilder(args);

// 設定 Serilog
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .WriteTo.Console()
    .WriteTo.TCPSink("localhost", 5000, new Serilog.Formatting.Json.JsonFormatter())
    .Enrich.WithProperty("app", "my-dotnet-app")
    .Enrich.WithProperty("environment", "production")
    .CreateLogger();

builder.Host.UseSerilog();

var app = builder.Build();

app.MapPost("/api/orders", (CreateOrderRequest request, ILogger<Program> logger) =>
{
    logger.LogInformation("Creating order for user {UserId}", request.UserId);
    
    try
    {
        var order = orderService.Create(request);
        logger.LogInformation("Order created successfully: {OrderId}", order.Id);
        return Results.Ok(order);
    }
    catch (InsufficientInventoryException ex)
    {
        logger.LogError(ex, "Failed to create order: insufficient inventory");
        throw;
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to create order: unexpected error");
        throw;
    }
});

app.Run();
```

## 在 Kibana 中搜尋日誌

### 基本搜尋

在 Discover 頁面的搜尋框中：

```
# 搜尋 ERROR 級別的日誌
level:ERROR

# 搜尋特定使用者的日誌
userId:12345

# 搜尋包含「timeout」的訊息
message:*timeout*

# 組合條件
level:ERROR AND app:"my-java-app"
```

### 時間範圍

右上角可以選擇時間範圍：
- Last 15 minutes
- Last 1 hour
- Last 24 hours
- Last 7 days
- Custom（自訂）

### 欄位過濾

左側的 **Available fields** 可以看到所有欄位。點選欄位名稱可以：
- 看到該欄位的統計（Top 5 values）
- 加入快速過濾

### 儲存查詢

當你調好查詢條件後，點選右上角 **Save**，命名為「Production Errors」。

之後就能快速切換到這個查詢。

## 第一個告警：錯誤日誌飆升

在 Kibana 左側選單點選 **Alerts and Insights** → **Rules and Connectors**

建立新規則：

**Rule Type**: Log threshold

**Conditions**:
- Log entries: `level:ERROR`
- Time window: Last 5 minutes
- Threshold: Count > 10

**Actions**: Email（需要先設定 SMTP）

這樣當 5 分鐘內出現超過 10 個 ERROR，就會發送告警郵件。

## ELK vs Prometheus：什麼時候用什麼？

我常被問：「有了 Prometheus 為什麼還需要 ELK？」

### Prometheus 適合的場景

```promql
# 過去 5 分鐘有多少錯誤？
sum(rate(http_requests_total{status=~"5.."}[5m]))
```

**回答**：「每秒 10 個錯誤」

### ELK 適合的場景

```
level:ERROR AND timestamp:[now-5m TO now]
```

**回答**：
```
[10:15:23] ERROR - PaymentGatewayTimeoutException: timeout after 30s
  User ID: 12345, Order ID: 67890
  
[10:15:25] ERROR - DatabaseConnectionException: cannot connect to mysql
  Connection pool exhausted
  
[10:15:27] ERROR - NullPointerException: user.address is null
  at OrderService.calculateShipping(OrderService.java:123)
```

**差異**：
- Prometheus 告訴你「有 10 個錯誤」
- ELK 告訴你「這 10 個錯誤分別是什麼」

## 常見問題

### Q: Elasticsearch 記憶體不足怎麼辦？

Elasticsearch 預設會用一半的系統記憶體。如果你的機器只有 4GB，它會試圖用 2GB。

調整 `docker-compose.yml`：

```yaml
environment:
  - "ES_JAVA_OPTS=-Xms512m -Xmx512m"  # 限制在 512MB
```

### Q: 日誌太多，Elasticsearch 磁碟快滿了？

設定 **Index Lifecycle Management (ILM)**：

```bash
# 7 天後刪除舊日誌
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "delete": {
        "min_age": "7d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Q: 查詢很慢怎麼辦？

1. **減少時間範圍**：不要一次查 30 天的日誌
2. **加入更多過濾條件**：縮小搜尋範圍
3. **使用精確匹配**：`level:ERROR` 比 `message:*error*` 快
4. **考慮分片策略**：每天一個 index（`logs-2023.04.07`）

---

**日誌不是「事後看看就好」，而是「問題排查的關鍵」。**

當你能在 5 分鐘內從幾 GB 的日誌中找到那一筆關鍵錯誤，你就理解了日誌管理的價值。
