---
layout: post
title: "測試環境的可觀測性與 CI/CD 整合"
date: 2023-09-08 09:00:00 +0800
categories: [Observability, Testing]
tags: [Testing, CI/CD, Integration Testing, Performance Testing]
---

大部分團隊只在生產環境做可觀測性，但**測試環境同樣重要**。

今天我們來談談如何在測試環境和 CI/CD 流程中整合可觀測性。

## 為什麼測試環境需要可觀測性？

### 場景 1：整合測試失敗

```bash
✗ test_create_order
  Expected: 200
  Actual: 500
```

但是，**為什麼失敗**？測試框架只告訴你結果，不告訴你原因。

### 場景 2：效能測試

```bash
Requests per second: 100 req/s
95th percentile latency: 2.5s  ← 太慢了！
```

但是，**哪個服務慢**？是資料庫？還是第三方 API？

## 測試環境的可觀測性架構

```
測試程式 → 應用程式 → OpenTelemetry Collector → Jaeger (測試環境)
                                              → Prometheus (測試環境)
                                              → Loki (測試環境)
```

重點：**測試環境和生產環境的可觀測性系統要分開**。

## 在測試中加入 trace_id

### Java + JUnit

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import org.junit.jupiter.api.Test;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class OrderControllerTest {
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private Tracer tracer;
    
    @Test
    public void testCreateOrder() {
        // 建立一個 Span
        Span span = tracer.spanBuilder("test-create-order").startSpan();
        
        try (var scope = span.makeCurrent()) {
            String traceId = span.getSpanContext().getTraceId();
            System.out.println("Trace ID: " + traceId);
            
            // 發送請求時，自動帶上 trace_id
            ResponseEntity<Order> response = restTemplate.postForEntity(
                "/api/orders",
                new OrderRequest("iPhone", 999.00),
                Order.class
            );
            
            assertEquals(200, response.getStatusCodeValue());
            
            // 如果測試失敗，輸出 Jaeger 連結
            if (response.getStatusCodeValue() != 200) {
                System.out.println("Jaeger: http://jaeger:16686/trace/" + traceId);
            }
        } finally {
            span.end();
        }
    }
}
```

### .NET + xUnit

```csharp
using System.Diagnostics;
using Xunit;

public class OrderControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderControllerTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task TestCreateOrder()
    {
        var activity = new Activity("test-create-order").Start();
        var traceId = activity.TraceId.ToString();
        
        Console.WriteLine($"Trace ID: {traceId}");
        
        var request = new HttpRequestMessage(HttpMethod.Post, "/api/orders");
        request.Content = JsonContent.Create(new { product = "iPhone", price = 999.00 });
        
        var response = await _client.SendAsync(request);
        
        Assert.Equal(200, (int)response.StatusCode);
        
        if ((int)response.StatusCode != 200)
        {
            Console.WriteLine($"Jaeger: http://jaeger:16686/trace/{traceId}");
        }
        
        activity.Stop();
    }
}
```

### 測試失敗時的輸出

```
✗ test_create_order
  Expected: 200
  Actual: 500
  Trace ID: 0af7651916cd43dd8448eb211c80319c
  Jaeger: http://jaeger:16686/trace/0af7651916cd43dd8448eb211c80319c
```

點選連結，直接跳到 Jaeger 看錯誤。

## 在 CI/CD 中整合可觀測性

### GitHub Actions

```yaml
name: Integration Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
      
      jaeger:
        image: jaegertracing/all-in-one:1.47
        ports:
          - 16686:16686
          - 4317:4317
      
      prometheus:
        image: prom/prometheus:v2.45.0
        ports:
          - 9090:9090
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
      
      - name: Run tests
        env:
          OTEL_EXPORTER_OTLP_ENDPOINT: http://localhost:4317
          OTEL_SERVICE_NAME: order-service-test
        run: mvn test
      
      - name: Extract trace IDs from failed tests
        if: failure()
        run: |
          grep -oP 'Trace ID: \K[a-f0-9]{32}' target/surefire-reports/*.txt > trace_ids.txt
      
      - name: Generate Jaeger links
        if: failure()
        run: |
          while read trace_id; do
            echo "http://localhost:16686/trace/$trace_id"
          done < trace_ids.txt > jaeger_links.txt
      
      - name: Comment on PR
        if: failure() && github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const links = fs.readFileSync('jaeger_links.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Test Failed\n\nJaeger Traces:\n${links}`
            });
```

### GitLab CI

```yaml
stages:
  - test

integration-test:
  stage: test
  image: maven:3.8-openjdk-17
  services:
    - postgres:15
    - jaegertracing/all-in-one:1.47
    - prom/prometheus:v2.45.0
  variables:
    OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317
    OTEL_SERVICE_NAME: order-service-test
  script:
    - mvn test
  after_script:
    - |
      if [ $CI_JOB_STATUS == 'failed' ]; then
        grep -oP 'Trace ID: \K[a-f0-9]{32}' target/surefire-reports/*.txt | while read trace_id; do
          echo "Jaeger: http://jaeger:16686/trace/$trace_id"
        done
      fi
  artifacts:
    when: on_failure
    reports:
      junit: target/surefire-reports/*.xml
```

## 效能測試與可觀測性

### 場景

你用 JMeter、Gatling 或 K6 做效能測試：

```bash
k6 run --vus 100 --duration 30s load-test.js
```

**結果**：

```
http_req_duration..............: avg=2.5s  p95=5s  p99=10s
http_req_failed................: 5.23%
```

但是，**哪個服務慢**？

### 解決方案：在效能測試中注入 trace_id

#### K6 範例

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend } from 'k6/metrics';

// 自訂 Metric
const orderDuration = new Trend('order_duration', true);

export default function () {
  // 產生一個 trace_id
  const traceId = generateTraceId();
  
  const headers = {
    'Content-Type': 'application/json',
    'traceparent': `00-${traceId}-${generateSpanId()}-01`,  // W3C Trace Context
  };
  
  const payload = JSON.stringify({
    product: 'iPhone',
    price: 999.00,
  });
  
  const response = http.post('http://order-service/api/orders', payload, { headers });
  
  orderDuration.add(response.timings.duration);
  
  check(response, {
    'status is 200': (r) => r.status === 200,
  });
  
  if (response.status !== 200) {
    console.log(`Failed request: http://jaeger:16686/trace/${traceId}`);
  }
}

function generateTraceId() {
  return Array.from({ length: 32 }, () => Math.floor(Math.random() * 16).toString(16)).join('');
}

function generateSpanId() {
  return Array.from({ length: 16 }, () => Math.floor(Math.random() * 16).toString(16)).join('');
}
```

### 測試後的分析

#### 1. 在 Jaeger 中查看慢請求

```
minDuration=5s
```

#### 2. 在 Prometheus 中查看趨勢

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))
```

#### 3. 在 Grafana 中建立 Load Test Dashboard

```json
{
  "panels": [
    {
      "title": "Request Rate",
      "targets": [{ "expr": "rate(http_requests_total[1m])" }]
    },
    {
      "title": "P95 Latency",
      "targets": [{ "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))" }]
    },
    {
      "title": "Error Rate",
      "targets": [{ "expr": "rate(http_requests_total{status=\"500\"}[1m]) / rate(http_requests_total[1m])" }]
    }
  ]
}
```

## Chaos Engineering 與可觀測性

### 場景

你用 Chaos Monkey 隨機殺掉一個服務，測試系統的韌性。

```bash
chaos-mesh inject pod-kill --namespace default --label app=payment-service
```

### 問題

系統的錯誤率上升了，但是：
- 哪些請求失敗了？
- 是否有正確的 Retry？
- 是否有 Circuit Breaker？

### 解決方案：在 Chaos 測試前，啟用完整的 Tracing

```yaml
# 暫時提高採樣率到 100%
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  config.yaml: |
    processors:
      tail_sampling:
        policies:
          - name: chaos-test
            type: probabilistic
            probabilistic:
              sampling_percentage: 100  # 100% 採樣
```

然後執行 Chaos 測試：

```bash
# 標記開始
curl -X POST http://prometheus:9090/api/v1/admin/tsdb/snapshot -d 'name=chaos-start'

# 執行 Chaos 測試
chaos-mesh inject pod-kill --namespace default --label app=payment-service --duration 60s

# 標記結束
curl -X POST http://prometheus:9090/api/v1/admin/tsdb/snapshot -d 'name=chaos-end'
```

在 Grafana 中，加上 Annotation：

```json
{
  "annotations": [
    {
      "time": "2023-09-08T10:00:00Z",
      "text": "Chaos: Kill payment-service"
    }
  ]
}
```

## 測試覆蓋率與 Tracing

### 問題

你怎麼知道測試是否覆蓋了所有的程式碼路徑？

### 解決方案：從 Trace 分析測試覆蓋率

#### 1. 在測試中記錄所有的 Span

```java
@Test
public void testAllEndpoints() {
    // 測試各種情況
    testCreateOrder();
    testCreateOrderWithInvalidData();
    testCreateOrderWithTimeout();
    testCreateOrderWithRetry();
}
```

#### 2. 在 Jaeger 中查詢

```
service=order-service
tags={"test": "true"}
```

#### 3. 提取所有的 Span

```bash
curl "http://jaeger:16686/api/traces?service=order-service&tag=test:true" | jq '.data[].spans[].operationName' | sort | uniq
```

**結果**：

```
create-order
validate-order
db-insert
payment-service-call
notification-service-call
```

#### 4. 比對程式碼

```bash
# 程式碼中的所有方法
grep -oP 'public \w+ \K\w+' OrderService.java | sort | uniq

# 測試中覆蓋的方法
curl "http://jaeger:16686/api/traces?service=order-service&tag=test:true" | jq '.data[].spans[].operationName' | sort | uniq

# 差集（沒被測試覆蓋的方法）
comm -23 <(grep -oP 'public \w+ \K\w+' OrderService.java | sort | uniq) <(curl "http://jaeger:16686/api/traces?service=order-service&tag=test:true" | jq '.data[].spans[].operationName' | sort | uniq)
```

## 實戰建議

### 1. 測試環境用固定的採樣率

生產環境可以用 1% 採樣率，但測試環境建議用 **100%**，確保所有測試都被追蹤。

### 2. 測試失敗時，自動產生 Jaeger 連結

在測試框架中加入：

```java
@AfterEach
public void afterEach(TestInfo testInfo) {
    if (testInfo.getTestMethod().isPresent()) {
        String traceId = Span.current().getSpanContext().getTraceId();
        System.out.println("Jaeger: http://jaeger:16686/trace/" + traceId);
    }
}
```

### 3. 在 PR 中顯示效能趨勢

```yaml
- name: Compare performance
  run: |
    # 查詢 main branch 的 P95 延遲
    MAIN_P95=$(curl "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{branch=\"main\"}[5m]))")
    
    # 查詢 PR branch 的 P95 延遲
    PR_P95=$(curl "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{branch=\"$GITHUB_HEAD_REF\"}[5m]))")
    
    # 比較
    echo "| Metric | main | PR | Diff |"
    echo "|--------|------|----|----- |"
    echo "| P95 Latency | ${MAIN_P95}ms | ${PR_P95}ms | ... |"
```

---

**測試環境的可觀測性可以讓你在問題進入生產環境之前發現並解決。**

當你能在 CI/CD 中自動分析效能和錯誤，你就能大幅提升交付品質。
