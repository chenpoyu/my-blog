---
layout: post
title: "AWS RDS 與 CloudWatch：資料庫管理與監控實戰"
date: 2021-01-06 11:20:00 +0800
categories: [AWS, Database]
tags: [AWS, RDS, CloudWatch, MySQL, PostgreSQL, Monitoring]
---

## 前言

過去幾週陸續接觸了 EC2、Lambda、RabbitMQ 和 Step Functions，這些服務都需要一個穩定的資料庫作為後盾。雖然可以在 EC2 上自己安裝 MySQL 或 PostgreSQL，但考慮到維護成本和可用性，我決定試試 AWS RDS (Relational Database Service)。

同時，資料庫的監控也非常重要，這時候 CloudWatch 就派上用場了。

## 為什麼選擇 RDS？

### 傳統自建資料庫的痛點

1. **備份管理**：需要自己寫腳本定期備份
2. **高可用性**：設定主從複製、容錯移轉很複雜
3. **效能調校**：需要深入了解資料庫參數
4. **修補更新**：定期更新資料庫版本，需要停機維護
5. **擴展困難**：垂直擴展需要停機，水平擴展需要額外設定

### RDS 的優勢

- **自動備份**：每日自動備份，保留期可設定（最長 35 天）
- **Multi-AZ 部署**：自動容錯移轉，高可用性
- **自動修補**：在維護視窗自動更新
- **效能監控**：整合 CloudWatch，提供豐富的指標
- **輕鬆擴展**：幾個點擊就能調整規格

## 支援的資料庫引擎

RDS 支援多種資料庫：
- MySQL 8.0, 5.7
- PostgreSQL 14, 13, 12
- MariaDB
- Oracle
- SQL Server
- Amazon Aurora（MySQL 和 PostgreSQL 相容）

我們的專案選擇 **MySQL 8.0**，因為：
- 團隊熟悉 MySQL
- 應用程式已使用 MySQL
- 成本相對較低

## 建立 RDS 實例

### 1. 選擇資料庫引擎

在 RDS 控制台選擇「建立資料庫」：
- 引擎：MySQL 8.0.27
- 版本：使用最新的穩定版

### 2. 選擇使用案例

- **生產環境**：Multi-AZ 部署，高可用性
- **開發/測試**：單一 AZ，節省成本

初期我選擇開發/測試模式，等正式上線再切換。

### 3. 實例規格

選擇 `db.t3.micro`：
- 2 vCPU
- 1 GB RAM
- 符合免費方案（每月 750 小時）

未來正式環境會考慮 `db.t3.medium` 或 `db.r5.large`。

### 4. 儲存設定

- **儲存類型**：General Purpose (SSD)
- **分配儲存空間**：20 GB
- **啟用儲存自動擴展**：是（最大 1000 GB）

這樣當資料量增加時，RDS 會自動擴展儲存空間。

### 5. 連線設定

**VPC 和子網路**：
- 選擇與 EC2 相同的 VPC
- 子網路群組：自動建立

**公開存取**：
- 開發階段：是（方便從本機連線）
- 生產環境：否（僅允許 VPC 內部存取）

**安全群組**：
建立新的安全群組，允許 MySQL 埠（3306）：

```
Type: MySQL/Aurora
Protocol: TCP
Port: 3306
Source: My IP（開發階段）或 EC2 Security Group（生產環境）
```

### 6. 資料庫認證

- 主使用者名稱：`admin`
- 主密碼：強密碼（至少 8 字元，包含大小寫、數字、特殊字元）

**重要**：將密碼存到 AWS Secrets Manager，不要寫在程式碼中。

### 7. 其他設定

- **初始資料庫名稱**：`iotdb`
- **備份保留期間**：7 天
- **維護視窗**：週日 03:00-04:00（流量最低時段）
- **啟用增強監控**：是（60 秒間隔）
- **啟用效能洞見**：是（可以分析慢查詢）

## 連接到 RDS

### 從本機連接

取得 RDS 端點：`mydb.abc123.us-east-1.rds.amazonaws.com`

使用 MySQL CLI：

```bash
mysql -h mydb.abc123.us-east-1.rds.amazonaws.com \
      -P 3306 \
      -u admin \
      -p
```

或使用 MySQL Workbench 等 GUI 工具。

### 從 Java 應用程式連接

```java
// application.properties
spring.datasource.url=jdbc:mysql://mydb.abc123.us-east-1.rds.amazonaws.com:3306/iotdb?useSSL=true&requireSSL=true
spring.datasource.username=admin
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

# 連線池設定
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

**最佳實踐**：密碼不要寫在設定檔中，而是透過環境變數或 Secrets Manager 取得。

### 使用 Secrets Manager

```java
@Configuration
public class DatabaseConfig {
    
    @Value("${aws.secretsmanager.secret-name}")
    private String secretName;
    
    @Bean
    public DataSource dataSource() {
        // 從 Secrets Manager 取得密碼
        String password = getSecret(secretName);
        
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(System.getenv("DB_URL"));
        config.setUsername(System.getenv("DB_USERNAME"));
        config.setPassword(password);
        config.setMaximumPoolSize(20);
        
        return new HikariDataSource(config);
    }
    
    private String getSecret(String secretName) {
        AWSSecretsManager client = AWSSecretsManagerClientBuilder.standard()
            .withRegion("us-east-1")
            .build();
        
        GetSecretValueRequest request = new GetSecretValueRequest()
            .withSecretId(secretName);
        
        GetSecretValueResult result = client.getSecretValue(request);
        
        // 解析 JSON 格式的 secret
        ObjectMapper mapper = new ObjectMapper();
        try {
            JsonNode node = mapper.readTree(result.getSecretString());
            return node.get("password").asText();
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse secret", e);
        }
    }
}
```

## CloudWatch 監控

RDS 自動將指標發送到 CloudWatch，可以監控：

### 基本指標（免費）

- **CPUUtilization**：CPU 使用率
- **DatabaseConnections**：資料庫連線數
- **FreeableMemory**：可用記憶體
- **FreeStorageSpace**：可用儲存空間
- **ReadLatency / WriteLatency**：讀寫延遲
- **ReadThroughput / WriteThroughput**：讀寫吞吐量
- **ReadIOPS / WriteIOPS**：每秒 I/O 操作數

### 增強監控（需付費）

啟用增強監控後，可以取得更細緻的資料：
- 每個程序的 CPU 使用率
- 記憶體使用詳情
- 檔案系統使用率
- 網路頻寬

監控間隔可設定為 1, 5, 10, 15, 30, 60 秒。

## 建立 CloudWatch 告警

### 1. CPU 使用率告警

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name rds-high-cpu \
  --alarm-description "RDS CPU usage above 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=mydb \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:database-alerts
```

### 2. 儲存空間告警

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name rds-low-storage \
  --alarm-description "RDS storage space below 2GB" \
  --metric-name FreeStorageSpace \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 2000000000 \
  --comparison-operator LessThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=mydb \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:database-alerts
```

### 3. 連線數告警

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name rds-high-connections \
  --alarm-description "RDS connections above 80" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Average \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=mydb \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:database-alerts
```

## Performance Insights

RDS Performance Insights 是一個強大的效能分析工具，可以：
- 識別資料庫瓶頸
- 分析哪些查詢消耗最多資源
- 查看資料庫負載的時間軸

### 啟用 Performance Insights

在 RDS 控制台：
1. 選擇實例
2. 修改
3. 勾選「啟用 Performance Insights」
4. 保留期間：7 天（免費）或更長（付費）

### 分析慢查詢

Performance Insights 會顯示：
- Top SQL：執行最頻繁或最耗時的查詢
- 等待事件：查詢在等待什麼（CPU、I/O、鎖定等）
- 資料庫負載：按時間、使用者、主機分組

實際案例：
```
Top SQL by Load:
1. SELECT * FROM devices WHERE status = 'online'  (45% load)
2. INSERT INTO heartbeats (device_id, timestamp, ...)  (30% load)
3. UPDATE devices SET last_seen = ?  (15% load)
```

發現第一個查詢沒有索引，新增索引後負載下降到 5%：

```sql
CREATE INDEX idx_devices_status ON devices(status);
```

## 自動備份與快照

### 自動備份

RDS 每天自動執行完整備份，並持續記錄交易日誌，可以：
- 還原到任意時間點（Point-in-Time Recovery）
- 保留期間：0-35 天
- 備份視窗：可自訂或讓 AWS 自動選擇

### 手動快照

重要操作前建立手動快照：

```bash
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-before-migration
```

手動快照不會自動刪除，需要手動管理。

### 還原資料庫

從快照還原：

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-restored \
  --db-snapshot-identifier mydb-before-migration
```

## Multi-AZ 高可用性

啟用 Multi-AZ 後，RDS 會：
1. 在不同的可用區建立備用實例
2. 同步複製資料到備用實例
3. 主實例故障時自動容錯移轉（通常 1-2 分鐘）
4. DNS 端點保持不變，應用程式無需修改

### 啟用 Multi-AZ

```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --multi-az \
  --apply-immediately
```

**成本影響**：Multi-AZ 會使成本加倍（因為有兩個實例）。

### 測試容錯移轉

```bash
aws rds reboot-db-instance \
  --db-instance-identifier mydb \
  --force-failover
```

我測試了一次，容錯移轉時間約 90 秒，應用程式短暫斷線後自動重連。

## Read Replicas 讀取副本

如果讀取流量很高，可以建立讀取副本：

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-replica \
  --source-db-instance-identifier mydb \
  --availability-zone us-east-1b
```

### 使用讀取副本

在應用程式中設定讀寫分離：

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource writeDataSource() {
        // 主資料庫（寫入）
        return DataSourceBuilder.create()
            .url("jdbc:mysql://mydb.abc123.us-east-1.rds.amazonaws.com:3306/iotdb")
            .username("admin")
            .password(getPassword())
            .build();
    }
    
    @Bean
    public DataSource readDataSource() {
        // 讀取副本（查詢）
        return DataSourceBuilder.create()
            .url("jdbc:mysql://mydb-replica.abc123.us-east-1.rds.amazonaws.com:3306/iotdb")
            .username("admin")
            .password(getPassword())
            .build();
    }
}

@Service
public class DeviceService {
    
    @Autowired
    @Qualifier("writeDataSource")
    private DataSource writeDataSource;
    
    @Autowired
    @Qualifier("readDataSource")
    private DataSource readDataSource;
    
    public void updateDevice(Device device) {
        // 使用主資料庫
        JdbcTemplate writeTemplate = new JdbcTemplate(writeDataSource);
        writeTemplate.update("UPDATE devices SET status = ? WHERE id = ?", 
            device.getStatus(), device.getId());
    }
    
    public List<Device> getDevices() {
        // 使用讀取副本
        JdbcTemplate readTemplate = new JdbcTemplate(readDataSource);
        return readTemplate.query("SELECT * FROM devices", 
            new DeviceRowMapper());
    }
}
```

## 參數群組調校

RDS 的參數群組類似 MySQL 的 `my.cnf`，可以調整資料庫參數。

### 建立自訂參數群組

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name my-mysql-params \
  --db-parameter-group-family mysql8.0 \
  --description "Custom MySQL parameters"
```

### 修改參數

常用的調校參數：

```bash
# 增加最大連線數
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-mysql-params \
  --parameters "ParameterName=max_connections,ParameterValue=200,ApplyMethod=pending-reboot"

# 調整查詢快取
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-mysql-params \
  --parameters "ParameterName=query_cache_size,ParameterValue=67108864,ApplyMethod=immediate"

# 啟用慢查詢日誌
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-mysql-params \
  --parameters "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate"

aws rds modify-db-parameter-group \
  --db-parameter-group-name my-mysql-params \
  --parameters "ParameterName=long_query_time,ParameterValue=2,ApplyMethod=immediate"
```

### 套用參數群組

```bash
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-parameter-group-name my-mysql-params \
  --apply-immediately
```

某些參數修改需要重新啟動資料庫。

## 安全性最佳實踐

### 1. 加密

啟用靜態加密（建立時設定，無法後續修改）：

```bash
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abcd-1234 \
  ...
```

### 2. SSL/TLS 連線

強制 SSL 連線：

```sql
-- 建立使用者時要求 SSL
CREATE USER 'app_user'@'%' IDENTIFIED BY 'password' REQUIRE SSL;

-- 或修改現有使用者
ALTER USER 'app_user'@'%' REQUIRE SSL;
```

應用程式連線字串加上 SSL 參數：

```
jdbc:mysql://mydb.abc123.us-east-1.rds.amazonaws.com:3306/iotdb?useSSL=true&requireSSL=true
```

### 3. IAM 資料庫認證

使用 IAM 角色而非密碼連接資料庫：

```java
import com.amazonaws.auth.DefaultAWSCredentialsProviderChain;
import com.amazonaws.services.rds.auth.GetIamAuthTokenRequest;
import com.amazonaws.services.rds.auth.RdsIamAuthTokenGenerator;

public String generateAuthToken() {
    RdsIamAuthTokenGenerator generator = RdsIamAuthTokenGenerator.builder()
        .credentials(new DefaultAWSCredentialsProviderChain())
        .region("us-east-1")
        .build();
    
    return generator.getAuthToken(
        GetIamAuthTokenRequest.builder()
            .hostname("mydb.abc123.us-east-1.rds.amazonaws.com")
            .port(3306)
            .userName("iam_user")
            .build()
    );
}
```

## 成本優化

### 1. Reserved Instances

如果確定會長期使用，可以購買保留實例（1 或 3 年），節省 30-60% 成本。

### 2. 調整實例規格

根據 CloudWatch 指標，如果 CPU 長期低於 20%，可以降低規格。

### 3. 刪除不必要的快照

手動快照會持續計費，定期清理不需要的快照。

### 4. 使用 Aurora Serverless

對於間歇性工作負載，考慮 Aurora Serverless，按實際使用量計費。

## 遇到的問題

### 問題 1：連線數耗盡

**現象**：`Too many connections` 錯誤

**原因**：應用程式連線池設定不當，連線未正確釋放

**解決**：
1. 調整 `max_connections` 參數
2. 優化應用程式連線池設定
3. 修復連線洩漏問題

### 問題 2：儲存空間不足

**現象**：資料庫變成唯讀模式

**原因**：日誌檔案佔用過多空間

**解決**：
1. 啟用儲存自動擴展
2. 設定日誌輪替
3. 定期清理舊資料

### 問題 3：效能下降

**現象**：查詢變慢

**原因**：缺少索引、查詢未優化

**解決**：
1. 使用 Performance Insights 分析
2. 新增適當的索引
3. 優化慢查詢

## 實際使用心得

使用 RDS 一個月後的感想：

### 優點

1. **省時省力**：不需要管理備份、修補、高可用性設定
2. **監控完善**：CloudWatch 整合良好，告警機制成熟
3. **擴展容易**：垂直擴展只需幾個點擊，幾分鐘完成
4. **效能穩定**：沒遇到明顯的效能問題

### 缺點

1. **成本較高**：比自建貴約 30-50%
2. **客製化受限**：某些底層參數無法修改
3. **依賴 AWS**：遷移到其他平台較困難

整體來說，對於中小型團隊，RDS 的便利性遠超過成本差異。

## 下一步

目前 RDS 運作良好，接下來計畫：
1. 測試 Multi-AZ 容錯移轉
2. 建立讀取副本，實作讀寫分離
3. 整合 Lambda 直接存取 RDS（使用 RDS Proxy）

下一篇計畫分享 S3 和 CloudFront 的使用經驗，看看如何建立高效能的靜態資源 CDN。

## 參考資料

- [AWS RDS 使用者指南](https://docs.aws.amazon.com/rds/)
- [CloudWatch 指標參考](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MonitoringOverview.html)
- [RDS 最佳實踐](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html)
