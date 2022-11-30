---
layout: post
title: "效能測試策略"
date: 2019-05-20 10:00:00 +0800
categories: [DevOps, Performance]
tags: [Performance Testing, JMeter, Gatling, Load Testing, Stress Testing]
---

上週建立了備份和災難恢復機制（參考 [備份與災難恢復](/posts/2019/05/13/backup-disaster-recovery/)），這週來談效能測試。

我們的痛點：上線前沒有做效能測試，結果第一天就因為負載過高而當機。

這週要學習：如何測試系統在高負載下的表現，找出效能瓶頸，確保系統能承受預期的流量。

> 使用工具：JMeter 5.1、Gatling 3.1.2、Locust 0.11.0

## 效能測試類型

### 1. 負載測試 (Load Testing)

模擬正常負載，驗證系統是否符合效能需求。

**目的**：
- 確認系統能處理預期的使用者數量
- 測量回應時間、吞吐量
- 找出效能基準線

**場景**：
- 1000 個並發使用者
- 每秒 500 個請求
- 持續 30 分鐘

### 2. 壓力測試 (Stress Testing)

超出正常負載，找出系統的極限。

**目的**：
- 找出系統的瓶頸
- 確認系統在超載時的行為（優雅降級 vs 崩潰）
- 測試恢復能力

**場景**：
- 逐步增加負載
- 從 100 → 500 → 1000 → 2000 → 5000 使用者
- 找出系統開始變慢或失敗的點

### 3. 峰值測試 (Spike Testing)

突然的流量暴增（例如：促銷活動）。

**目的**：
- 驗證系統能否應對突然的流量高峰
- 測試自動擴展機制

**場景**：
- 平時 100 使用者
- 突然增加到 5000 使用者
- 持續 5 分鐘
- 回到 100 使用者

### 4. 耐久測試 (Soak Testing)

長時間運行，發現記憶體洩漏等問題。

**目的**：
- 發現記憶體洩漏
- 發現資源未釋放
- 確認系統穩定性

**場景**：
- 正常負載（500 使用者）
- 持續 24-72 小時

## Apache JMeter

Java 寫的經典效能測試工具。

### 安裝

```bash
# macOS
brew install jmeter

# 啟動 GUI
jmeter
```

### 建立測試計畫

**場景**：測試訂單查詢 API

1. **新增 Thread Group**：
   - Number of Threads: 100（100 個並發使用者）
   - Ramp-Up Period: 10（10 秒內啟動所有使用者）
   - Loop Count: 10（每個使用者執行 10 次）

2. **新增 HTTP Request**：
   - Server: api.example.com
   - Port: 443
   - Protocol: https
   - Path: /orders/123
   - Method: GET

3. **新增 Header**：
   - Authorization: Bearer ${token}
   - Content-Type: application/json

4. **新增 Listener（結果監聽器）**：
   - View Results Tree（查看每個請求）
   - Summary Report（摘要報告）
   - Aggregate Report（聚合報告）

執行測試：點擊 "Start"

### 使用 CSV 資料

測試不同的訂單 ID：

`orders.csv`：
```csv
orderId
123
456
789
```

新增 **CSV Data Set Config**：
- Filename: orders.csv
- Variable Names: orderId

修改 HTTP Request：
- Path: /orders/${orderId}

每個請求會使用不同的訂單 ID。

### 斷言（Assertion）

驗證回應是否正確：

**Response Assertion**：
- Field to Test: Response Code
- Pattern: 200

**JSON Assertion**：
- Assert JSON Path: $.status
- Expected Value: success

測試失敗時會在報告中顯示。

### 命令列執行

GUI 模式耗資源，生產測試用命令列：

```bash
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/
```

參數：
- `-n`：非 GUI 模式
- `-t`：測試計畫檔案
- `-l`：結果檔案
- `-e -o`：產生 HTML 報告

查看報告：
```bash
open report/index.html
```

### 分散式測試

單機不夠力？使用分散式測試。

**架構**：
```
Controller (MacBook)
  ↓ 控制
Agent 1 (AWS m5.large)  ┐
Agent 2 (AWS m5.large)  ├─ 產生負載
Agent 3 (AWS m5.large)  ┘
```

配置 Agent：
```bash
# 每台 Agent
jmeter-server
```

配置 Controller：
```properties
# jmeter.properties
remote_hosts=agent1.example.com,agent2.example.com,agent3.example.com
```

執行：
```bash
jmeter -n -t test-plan.jmx -r -l results.jtl
```

`-r`：使用所有 remote agents

## Gatling

Scala 寫的高效能測試工具，程式碼定義測試。

### 安裝

```bash
# macOS
brew install gatling

# 或下載
wget https://repo1.maven.org/maven2/io/gatling/highcharts/gatling-charts-highcharts-bundle/3.1.2/gatling-charts-highcharts-bundle-3.1.2.zip
unzip gatling-charts-highcharts-bundle-3.1.2.zip
```

### 建立測試

`user-files/simulations/OrderSimulation.scala`：
```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderSimulation extends Simulation {
  
  // HTTP 配置
  val httpProtocol = http
    .baseUrl("https://api.example.com")
    .acceptHeader("application/json")
    .authorizationHeader("Bearer YOUR_TOKEN")
  
  // 場景定義
  val scn = scenario("Order API Test")
    .exec(http("Get Order")
      .get("/orders/123")
      .check(status.is(200))
      .check(jsonPath("$.status").is("success")))
    .pause(1)  // 每次請求間隔 1 秒
  
  // 負載模型
  setUp(
    scn.inject(
      atOnceUsers(10),           // 立即 10 個使用者
      rampUsers(50) during (10.seconds),  // 10 秒內增加到 50
      constantUsersPerSec(20) during (30.seconds)  // 30 秒內保持每秒 20 個使用者
    ).protocols(httpProtocol)
  )
}
```

執行：
```bash
gatling
# 選擇 OrderSimulation
```

報告自動產生在 `results/` 目錄。

### 進階場景

**多步驟流程**：
```scala
val scn = scenario("E-commerce Flow")
  .exec(http("Browse Products")
    .get("/products")
    .check(status.is(200))
    .check(jsonPath("$[0].id").saveAs("productId")))  // 儲存第一個產品 ID
  
  .pause(2)
  
  .exec(http("View Product Detail")
    .get("/products/${productId}")  // 使用儲存的 ID
    .check(status.is(200)))
  
  .pause(1)
  
  .exec(http("Add to Cart")
    .post("/cart/items")
    .body(StringBody("""{"productId": "${productId}", "quantity": 1}"""))
    .check(status.is(201)))
  
  .pause(2)
  
  .exec(http("Checkout")
    .post("/orders")
    .body(StringBody("""{"paymentMethod": "credit_card"}"""))
    .check(status.is(201))
    .check(jsonPath("$.orderId").saveAs("orderId")))
  
  .exec(http("View Order")
    .get("/orders/${orderId}")
    .check(status.is(200)))
```

模擬真實使用者的完整流程。

**條件判斷**：
```scala
.exec(http("Create Order")
  .post("/orders")
  .body(StringBody("""{"items": [...]}"""))
  .check(status.saveAs("statusCode")))

.doIf(session => session("statusCode").as[Int] == 201) {
  exec(http("Get Order")
    .get("/orders/${orderId}"))
}
```

**循環**：
```scala
.repeat(10) {
  exec(http("Poll Order Status")
    .get("/orders/${orderId}/status")
    .check(jsonPath("$.status").saveAs("orderStatus")))
  .pause(1)
  .exitHereIf(session => session("orderStatus").as[String] == "completed")
}
```

### 負載模型

**階梯式增長**：
```scala
setUp(
  scn.inject(
    incrementUsersPerSec(5)    // 每秒增加 5 個使用者
      .times(5)                // 重複 5 次
      .eachLevelLasting(30.seconds)  // 每階段持續 30 秒
      .separatedByRampsLasting(10.seconds)  // 階段間爬升 10 秒
      .startingFrom(10)        // 從 10 個使用者開始
  ).protocols(httpProtocol)
)
```

結果：10 → 15 → 20 → 25 → 30 → 35 使用者/秒

**高峰模擬**：
```scala
setUp(
  scn.inject(
    constantUsersPerSec(10) during (60.seconds),   // 平時 10/秒
    rampUsersPerSec(10) to (100) during (10.seconds),  // 10 秒內暴增到 100/秒
    constantUsersPerSec(100) during (30.seconds),  // 維持 30 秒
    rampUsersPerSec(100) to (10) during (10.seconds)   // 10 秒內降回 10/秒
  ).protocols(httpProtocol)
)
```

## Locust

Python 寫的工具，程式碼簡單，適合快速測試。

### 安裝

```bash
pip install locust==0.11.0
```

### 建立測試

`locustfile.py`：
```python
from locust import HttpLocust, TaskSet, task, between

class UserBehavior(TaskSet):
    
    @task(2)  # 權重 2（執行機率較高）
    def get_orders(self):
        self.client.get("/orders", headers={
            "Authorization": "Bearer YOUR_TOKEN"
        })
    
    @task(1)  # 權重 1
    def create_order(self):
        self.client.post("/orders", json={
            "items": [{"productId": 123, "quantity": 1}]
        }, headers={
            "Authorization": "Bearer YOUR_TOKEN"
        })
    
    def on_start(self):
        # 使用者開始時執行（例如：登入）
        response = self.client.post("/auth/login", json={
            "username": "test@example.com",
            "password": "password"
        })
        self.token = response.json()["token"]

class WebsiteUser(HttpLocust):
    task_set = UserBehavior
    wait_time = between(1, 3)  # 每次請求間隔 1-3 秒
    host = "https://api.example.com"
```

### 執行測試

啟動 Web UI：
```bash
locust
```

開啟 http://localhost:8089

輸入：
- Number of users: 100
- Hatch rate: 10（每秒啟動 10 個使用者）

點擊 "Start swarming"

即時查看：
- RPS（每秒請求數）
- 回應時間
- 錯誤率

### 無 UI 模式

```bash
locust --no-web -c 100 -r 10 -t 10m
```

參數：
- `-c 100`：100 個並發使用者
- `-r 10`：每秒啟動 10 個
- `-t 10m`：執行 10 分鐘

## 分析效能指標

### 關鍵指標

**回應時間 (Response Time)**：
- P50（中位數）：50% 請求的回應時間
- P95：95% 請求的回應時間
- P99：99% 請求的回應時間
- Max：最長回應時間

目標範例：
- P50 < 100ms
- P95 < 500ms
- P99 < 1000ms

**吞吐量 (Throughput)**：
- 每秒請求數 (RPS)
- 每秒交易數 (TPS)

目標：系統應該能處理預期流量的 2-3 倍。

**錯誤率 (Error Rate)**：
- HTTP 4xx、5xx 錯誤比例

目標：< 0.1%

**資源使用率**：
- CPU：< 70%
- 記憶體：< 80%
- 網路：< 80%

超過 80% 表示資源不足，需要擴展。

### 效能瓶頸分析

**CPU 瓶頸**：
- 現象：CPU 100%，回應時間線性增長
- 原因：計算密集（例如：加密、壓縮）
- 解決：優化演算法、增加 CPU

**記憶體瓶頸**：
- 現象：記憶體滿，開始 swap，回應時間暴增
- 原因：物件太多、快取過大、記憶體洩漏
- 解決：優化記憶體使用、增加記憶體

**資料庫瓶頸**：
- 現象：應用 CPU/記憶體正常，但回應慢
- 原因：SQL 慢查詢、連線池耗盡、鎖競爭
- 解決：優化 SQL、增加索引、增加連線池

**網路瓶頸**：
- 現象：回應時間不穩定，偶爾很慢
- 原因：頻寬不足、網路延遲
- 解決：增加頻寬、使用 CDN、地域部署

## 實際案例

### 案例一：訂單 API 效能測試

**需求**：訂單查詢 API 應該能承受 500 RPS，P95 < 200ms

**測試**：
```bash
# JMeter 測試
jmeter -n -t order-api-test.jmx -l results.jtl
```

**結果**（初始）：
- 100 RPS：P95 = 150ms ✅
- 300 RPS：P95 = 500ms ❌
- 500 RPS：P95 = 2000ms ❌

**瓶頸分析**：
1. 查看 Prometheus：資料庫 CPU 100%
2. 查看慢查詢日誌：`SELECT * FROM orders WHERE user_id = ?` 沒有索引

**優化**：
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

**結果**（優化後）：
- 100 RPS：P95 = 80ms ✅
- 300 RPS：P95 = 120ms ✅
- 500 RPS：P95 = 180ms ✅

達標！

### 案例二：促銷活動峰值測試

**場景**：促銷開始時,流量從 100 RPS 暴增到 2000 RPS

**測試**（Gatling）：
```scala
setUp(
  scn.inject(
    constantUsersPerSec(100) during (60.seconds),
    rampUsersPerSec(100) to (2000) during (10.seconds),
    constantUsersPerSec(2000) during (120.seconds)
  ).protocols(httpProtocol)
)
```

**結果**（初始）：
- 前 60 秒：正常（P95 = 100ms）
- 暴增時：大量 503 錯誤（50% 錯誤率）
- 原因：Pod 數量不足，HPA 來不及擴展

**優化**：
1. 預先擴展 Pod（促銷前）：
```bash
kubectl scale deployment order-service --replicas=20
```

2. 調整 HPA 更激進：
```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0  # 立即擴展
    policies:
    - type: Percent
      value: 100  # 每次增加 100%
      periodSeconds: 15
```

**結果**（優化後）：
- 暴增時：錯誤率 < 1% ✅
- P95：250ms（稍高但可接受）✅

### 案例三：記憶體洩漏

**測試**：耐久測試（24 小時，500 RPS）

**結果**：
- 0-6 小時：正常
- 6-12 小時：回應時間緩慢增加
- 12 小時後：OOM killed

**分析**：
```bash
# Heap dump
jmap -dump:live,format=b,file=heap.bin <PID>

# 分析（使用 Eclipse MAT）
# 發現：大量 User 物件未釋放
```

**原因**：快取沒有設定過期時間
```java
// 錯誤
Cache<String, User> userCache = CacheBuilder.newBuilder()
    .build();

// 正確
Cache<String, User> userCache = CacheBuilder.newBuilder()
    .maximumSize(10000)
    .expireAfterWrite(1, TimeUnit.HOURS)
    .build();
```

**結果**（修正後）：
- 24 小時測試：記憶體穩定 ✅
- 無 OOM ✅

## CI/CD 整合

將效能測試整合到 CI/CD。

### GitLab CI

`.gitlab-ci.yml`：
```yaml
stages:
  - build
  - test
  - performance-test
  - deploy

performance_test:
  stage: performance-test
  image: openjdk:11
  before_script:
    - wget https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-5.1.tgz
    - tar -xzf apache-jmeter-5.1.tgz
  script:
    - apache-jmeter-5.1/bin/jmeter -n -t test-plan.jmx -l results.jtl -e -o report/
    - python analyze_results.py results.jtl  # 分析結果
  artifacts:
    paths:
      - report/
    expire_in: 7 days
  only:
    - master
  when: manual  # 手動觸發
```

`analyze_results.py`：
```python
import sys
import csv

def analyze_results(jtl_file):
    total = 0
    errors = 0
    latencies = []
    
    with open(jtl_file, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            total += 1
            latencies.append(int(row['elapsed']))
            if row['success'] != 'true':
                errors += 1
    
    error_rate = errors / total * 100
    p95 = sorted(latencies)[int(len(latencies) * 0.95)]
    
    print(f"Total requests: {total}")
    print(f"Error rate: {error_rate:.2f}%")
    print(f"P95 latency: {p95}ms")
    
    # 檢查閾值
    if error_rate > 1.0:
        print("❌ Error rate too high!")
        sys.exit(1)
    
    if p95 > 500:
        print("❌ P95 latency too high!")
        sys.exit(1)
    
    print("✅ Performance test passed!")

if __name__ == '__main__':
    analyze_results(sys.argv[1])
```

不符合效能標準時,CI 失敗。

## 心得

效能測試讓我們發現很多問題：
- 沒有索引的查詢
- 連線池太小
- 快取配置不當
- 記憶體洩漏

這些問題在開發環境不明顯（流量小），但生產環境就會爆發。

Gatling 是我的最愛，程式碼定義測試很靈活，報告也很漂亮。JMeter 功能完整但比較老舊。Locust 適合快速測試。

建議：
1. **早期測試**：不要等到上線前才測
2. **持續測試**：整合到 CI/CD
3. **真實場景**：模擬真實使用者行為
4. **監控指標**：搭配 Prometheus/Grafana
5. **負載餘裕**：系統應該能承受預期流量的 2-3 倍

下週要研究 SRE 實踐，學習 Google 如何運維大規模系統。
