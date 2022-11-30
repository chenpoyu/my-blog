---
layout: post
title: "資料庫遷移策略"
date: 2019-05-06 10:00:00 +0800
categories: [DevOps, Database]
tags: [Database Migration, MySQL, PostgreSQL, Zero Downtime]
---

上週學習了混沌工程（參考 [混沌工程入門](/posts/2019/04/29/chaos-engineering-intro/)），這週面臨一個實際挑戰：資料庫遷移。

我們的情況：
- 現有：MySQL 5.7，單機，效能瓶頸
- 目標：PostgreSQL 11，叢集，更好的效能和功能
- 要求：**零停機遷移**，資料不能遺失

這是個大工程，需要仔細規劃。

> 使用工具：pg_chameleon、AWS DMS、Flyway、Liquibase

## 為什麼遷移資料庫

我們的痛點：
1. **效能瓶頸**：單機 MySQL，讀寫都在同一台，已達極限
2. **功能需求**：需要 JSON 查詢、全文搜尋，PostgreSQL 更適合
3. **複寫延遲**：MySQL 主從複寫延遲嚴重（5-10 秒）
4. **備份恢復慢**：200GB 資料，備份 2 小時，恢復 4 小時

團隊討論後決定遷移到 PostgreSQL + Patroni（自動 failover）。

## 遷移策略

### 策略一：停機遷移（Big Bang）

最簡單但代價最高：

1. 凌晨 2 點，關閉應用
2. 匯出 MySQL 資料
3. 轉換資料格式
4. 匯入 PostgreSQL
5. 測試
6. 切換應用到新資料庫
7. 開啟應用

停機時間：4-8 小時

**優點**：簡單直接
**缺點**：長時間停機不可接受

### 策略二：雙寫策略

應用同時寫兩個資料庫：

```java
@Transactional
public void createOrder(Order order) {
    // 寫入 MySQL（主要）
    mysqlOrderRepository.save(order);
    
    try {
        // 寫入 PostgreSQL（新）
        postgresqlOrderRepository.save(order);
    } catch (Exception e) {
        logger.error("Failed to write to PostgreSQL", e);
        // 不影響主流程
    }
}
```

流程：
1. 修改應用，雙寫
2. 歷史資料遷移
3. 驗證資料一致性
4. 切換讀取到 PostgreSQL
5. 停止寫入 MySQL

**優點**：停機時間短（只需切換讀取）
**缺點**：程式碼複雜，資料一致性難保證

### 策略三：CDC（Change Data Capture）

我們選擇的方案：使用 CDC 工具同步資料。

```
MySQL (主) 
   ↓ (CDC 即時同步)
PostgreSQL (新)
   ↓ (切換)
PostgreSQL (主)
```

**優點**：
- 零停機
- 資料一致性高
- 可以慢慢測試
- 可以隨時回滾

**缺點**：
- 需要額外工具
- 複雜度較高

## 使用 pg_chameleon 遷移

pg_chameleon 是 Python 寫的工具，可以從 MySQL 複寫到 PostgreSQL。

### 安裝

```bash
# 建立虛擬環境
python3 -m venv venv
source venv/bin/activate

# 安裝
pip install pg_chameleon

# 初始化配置
chameleon set_configuration_files
```

### 配置

`~/.pg_chameleon/configuration/default.yaml`：
```yaml
---
sources:
  mysql:
    db_conn:
      host: "mysql-master.example.com"
      port: "3306"
      user: "repl_user"
      password: "password"
      charset: "utf8"
      database: "ecommerce"
    
    schema_mappings:
      ecommerce: ecommerce_schema
    
    limit_tables:
      - orders
      - order_items
      - users
      - products
    
    skip_tables: []
    
    grant_select_to:
      - app_user
    
    lock_timeout: "120s"
    my_server_id: 100
    replica_batch_size: 10000
    sleep_loop: 1
    
    type: mysql
    
target:
  host: "postgresql-master.example.com"
  port: "5432"
  user: "postgres"
  password: "password"
  database: "ecommerce"
  charset: "utf8"
```

### 初始化

建立複寫使用者（MySQL）：
```sql
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl_user'@'%';
GRANT SELECT ON ecommerce.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;
```

啟用 binlog（MySQL）：
```ini
# /etc/mysql/my.cnf
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW
binlog_row_image = FULL
```

建立目標 schema（PostgreSQL）：
```bash
chameleon create_replica_schema --config default --source mysql
```

### 初始資料載入

```bash
# 第一次全量複製
chameleon init_replica --config default --source mysql
```

這會：
1. 鎖定 MySQL 表格（READ LOCK）
2. 匯出資料
3. 轉換格式（MySQL → PostgreSQL）
4. 匯入 PostgreSQL
5. 記錄 binlog 位置
6. 解鎖表格

時間：視資料量而定（我們 200GB 約 3 小時）。

### 即時同步

```bash
# 啟動持續複寫（監聽 binlog）
chameleon start_replica --config default --source mysql
```

現在 MySQL 的任何變更都會即時同步到 PostgreSQL！

監控複寫延遲：
```bash
chameleon show_status --config default --source mysql
```

輸出：
```
Source: mysql
Status: running
Binlog file: mysql-bin.000123
Binlog position: 45678901
Tables in sync: 4
Lag: 0.5 seconds
```

延遲通常在 1 秒內，可接受。

## 資料一致性驗證

CDC 在跑，但怎麼確定資料真的一致？

### 方法一：行數比對

```bash
# MySQL
mysql -e "SELECT COUNT(*) FROM ecommerce.orders"
# 12345678

# PostgreSQL
psql -c "SELECT COUNT(*) FROM ecommerce_schema.orders"
# 12345678
```

行數一樣，但不夠精確（可能內容不同）。

### 方法二：Checksum 比對

比對每行的 checksum：

MySQL：
```sql
SELECT 
    id,
    MD5(CONCAT_WS(',', id, user_id, total_amount, status, created_at)) as checksum
FROM orders
ORDER BY id;
```

PostgreSQL：
```sql
SELECT 
    id,
    MD5(CONCAT_WS(',', id::text, user_id::text, total_amount::text, status, created_at::text)) as checksum
FROM orders
ORDER BY id;
```

比對輸出檔案：
```bash
diff mysql_checksums.txt postgres_checksums.txt
```

如果有差異，逐行檢查。

### 方法三：使用工具

`pt-table-checksum`（Percona Toolkit）：
```bash
pt-table-checksum \
  --host mysql-master.example.com \
  --databases ecommerce \
  --tables orders,users,products
```

雖然是 MySQL 工具,但可以匯出結果與 PostgreSQL 比對。

## 應用程式改造

### 方案一：抽象層

建立資料庫抽象層，支援多種資料庫：

```java
public interface OrderRepository {
    Order findById(Long id);
    void save(Order order);
    List<Order> findByUserId(Long userId);
}

@Repository
@Profile("mysql")
public class MysqlOrderRepository implements OrderRepository {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public Order findById(Long id) {
        // MySQL specific SQL
        return jdbcTemplate.queryForObject(
            "SELECT * FROM orders WHERE id = ?",
            new OrderRowMapper(), id);
    }
}

@Repository
@Profile("postgresql")
public class PostgresqlOrderRepository implements OrderRepository {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public Order findById(Long id) {
        // PostgreSQL specific SQL
        return jdbcTemplate.queryForObject(
            "SELECT * FROM orders WHERE id = ?",
            new OrderRowMapper(), id);
    }
}
```

配置切換：
```yaml
# application-mysql.yml
spring:
  profiles:
    active: mysql

# application-postgresql.yml
spring:
  profiles:
    active: postgresql
```

切換只需改配置，不需改程式碼。

### 方案二：Feature Toggle

使用 feature flag 控制讀取來源：

```java
@Service
public class OrderService {
    @Autowired
    private MysqlOrderRepository mysqlRepo;
    
    @Autowired
    private PostgresqlOrderRepository postgresqlRepo;
    
    @Autowired
    private FeatureToggle featureToggle;
    
    public Order getOrder(Long id) {
        if (featureToggle.isEnabled("use-postgresql")) {
            return postgresqlRepo.findById(id);
        } else {
            return mysqlRepo.findById(id);
        }
    }
}
```

灰度切換：
```java
// 5% 流量用 PostgreSQL
if (featureToggle.isEnabledForUser("use-postgresql", userId)) {
    return postgresqlRepo.findById(id);
}
```

## Schema 遷移管理

使用 Flyway 管理 schema 變更。

`pom.xml`：
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>5.2.4</version>
</dependency>
```

`application.yml`：
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration/postgresql
    baseline-on-migrate: true
```

Migration 檔案：

`V1__initial_schema.sql`：
```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at);
```

`V2__add_order_items.sql`：
```sql
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

應用啟動時自動執行 migrations。

## 切換流程

準備工作完成後，開始切換：

### 階段一：唯讀測試（Day 1）

```yaml
# 所有讀取從 PostgreSQL，寫入還在 MySQL
featureToggle:
  use-postgresql-read: true
  use-postgresql-write: false
```

監控：
- 回應時間
- 錯誤率
- 資料一致性

如果有問題，立即切回 MySQL。

### 階段二：灰度寫入（Day 2-3）

```yaml
# 5% 寫入到 PostgreSQL
featureToggle:
  use-postgresql-read: true
  use-postgresql-write: 0.05
```

雙寫並比對結果：
```java
public void createOrder(Order order) {
    // 寫入 MySQL（主）
    Order mysqlOrder = mysqlRepo.save(order);
    
    if (featureToggle.isEnabled("use-postgresql-write")) {
        // 寫入 PostgreSQL（測試）
        Order postgresqlOrder = postgresqlRepo.save(order);
        
        // 比對結果
        if (!mysqlOrder.equals(postgresqlOrder)) {
            logger.error("Data mismatch! MySQL: {}, PostgreSQL: {}", 
                mysqlOrder, postgresqlOrder);
            alerting.send("Data inconsistency detected!");
        }
    }
    
    return mysqlOrder;
}
```

逐步增加：5% → 20% → 50%。

### 階段三：全量切換（Day 4）

```yaml
# 100% 切換到 PostgreSQL
featureToggle:
  use-postgresql-read: true
  use-postgresql-write: true
```

監控 24 小時，確認沒問題。

### 階段四：停止 CDC（Day 5）

```bash
# 停止複寫
chameleon stop_replica --config default --source mysql
```

關閉 MySQL 寫入，PostgreSQL 成為唯一資料來源。

### 階段五：下線 MySQL（Day 30）

保留 MySQL 30 天（唯讀），作為備援。

確認沒問題後，完全下線。

## 效能優化

### PostgreSQL 配置

`postgresql.conf`：
```ini
# 連線
max_connections = 200

# 記憶體
shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 64MB
maintenance_work_mem = 1GB

# 寫入效能
wal_buffers = 16MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB

# 查詢優化
random_page_cost = 1.1  # SSD
effective_io_concurrency = 200
```

### 索引優化

PostgreSQL 支援多種索引類型：

```sql
-- B-tree（預設，適合等值和範圍查詢）
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- GIN（適合 JSON、陣列、全文搜尋）
CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- BRIN（適合大表的時序資料）
CREATE INDEX idx_orders_created_at ON orders USING BRIN(created_at);

-- Partial Index（部分索引）
CREATE INDEX idx_active_orders ON orders(user_id) WHERE status = 'active';
```

### 連線池

使用 PgBouncer：
```ini
[databases]
ecommerce = host=postgresql-master.example.com port=5432 dbname=ecommerce

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

應用連線改成：
```yaml
spring:
  datasource:
    url: jdbc:postgresql://pgbouncer:6432/ecommerce
```

連線數從 200 降到 25，效能提升 3 倍。

## 回滾計畫

如果 PostgreSQL 有問題，需要快速回滾。

### 回滾步驟

1. **Feature Toggle 切回 MySQL**：
```bash
curl -X POST http://config-service/toggle \
  -d '{"use-postgresql-read": false, "use-postgresql-write": false}'
```

立即生效，所有流量回到 MySQL。

2. **重啟應用**（如果 Toggle 失效）：
```bash
kubectl set env deployment/order-service SPRING_PROFILES_ACTIVE=mysql
kubectl rollout restart deployment/order-service
```

3. **修正 PostgreSQL 問題**

4. **重新切換**

### 資料回補

如果在 PostgreSQL 期間寫入的資料：

選項 1：CDC 反向同步（PostgreSQL → MySQL）

選項 2：手動匯出/匯入
```bash
# 匯出 PostgreSQL 增量資料
psql -c "COPY (SELECT * FROM orders WHERE created_at > '2019-05-06') TO STDOUT WITH CSV" > new_orders.csv

# 匯入 MySQL
mysql -e "LOAD DATA LOCAL INFILE 'new_orders.csv' INTO TABLE orders FIELDS TERMINATED BY ','"
```

## 遇到的問題

### 問題一：資料類型差異

MySQL `DATETIME` vs PostgreSQL `TIMESTAMP`

MySQL：
```sql
SELECT * FROM orders WHERE created_at = '2019-05-06 10:00:00';  -- 有時區問題
```

PostgreSQL：
```sql
SELECT * FROM orders WHERE created_at = '2019-05-06 10:00:00'::timestamp;
```

解決：統一使用 `TIMESTAMP WITH TIME ZONE`。

### 問題二：自增 ID

MySQL：`AUTO_INCREMENT`
PostgreSQL：`SERIAL` 或 `SEQUENCE`

遷移後，sequence 沒有更新：
```sql
-- 錯誤
INSERT INTO orders (id, ...) VALUES (DEFAULT, ...);
-- ERROR: duplicate key value violates unique constraint

-- 修正：更新 sequence
SELECT setval('orders_id_seq', (SELECT MAX(id) FROM orders));
```

### 問題三：SQL 語法差異

MySQL：
```sql
SELECT * FROM orders LIMIT 10 OFFSET 20;
```

PostgreSQL（相同）：
```sql
SELECT * FROM orders LIMIT 10 OFFSET 20;
```

但字串連接不同：

MySQL：
```sql
SELECT CONCAT(first_name, ' ', last_name) FROM users;
```

PostgreSQL：
```sql
SELECT first_name || ' ' || last_name FROM users;
-- 或
SELECT CONCAT(first_name, ' ', last_name) FROM users;  -- 也支援
```

建議：使用 JPA/Hibernate，自動處理差異。

## 心得

資料庫遷移是個大工程，但規劃好就不可怕。

CDC 工具（pg_chameleon）真的很強大，解決了最大的痛點：即時同步。以前的做法是停機遷移，現在可以慢慢測試，確認沒問題再切換。

Feature Toggle 也很重要，可以隨時切回舊系統。這給了我們很大的信心。

遷移後的好處：
- 查詢效能提升 50%（特別是 JOIN 查詢）
- JSON 查詢方便很多
- 複寫延遲從 5-10 秒降到 < 1 秒（Patroni）
- 備份恢復快很多（並行備份）

建議：
1. 充分測試（至少 2 週灰度）
2. 監控所有指標
3. 準備回滾計畫
4. 保留舊資料庫一段時間

下週要研究備份和災難恢復策略，確保資料安全。
