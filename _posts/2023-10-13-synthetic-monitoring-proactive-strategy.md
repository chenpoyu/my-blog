---
layout: post
title: "Synthetic Monitoring：主動式監控策略"
date: 2023-10-13 09:00:00 +0800
categories: [Observability, Synthetic Monitoring]
tags: [Synthetic Monitoring, Testing, Automation, User Experience]
---

被動監控（Reactive Monitoring）只能在問題發生後才知道。

**Synthetic Monitoring** 可以在使用者發現之前，主動發現問題。

今天我們來談談如何實作主動式監控。

## Synthetic Monitoring vs Real User Monitoring

| 特性 | Synthetic Monitoring | Real User Monitoring (RUM) |
|------|---------------------|---------------------------|
| 監控方式 | 模擬使用者行為 | 真實使用者的資料 |
| 監控頻率 | 持續（每分鐘） | 只有使用者訪問時 |
| 覆蓋範圍 | 預定義的流程 | 所有使用者行為 |
| 偵測速度 | 快（主動） | 慢（被動） |
| 成本 | 較高 | 較低 |

**建議**：兩者都要做。

## Synthetic Monitoring 的應用場景

### 場景 1：監控關鍵業務流程

每 5 分鐘模擬一次使用者的完整流程：

```
1. 訪問首頁
2. 搜尋商品
3. 加入購物車
4. 結帳（使用測試信用卡）
5. 確認訂單
```

如果任何一步失敗，立即告警。

### 場景 2：監控 API 的正確性

不只監控 API 是否回應 200，還要驗證回應內容是否正確。

```
GET /api/products/123
→ 回應：{"id": 123, "name": "iPhone", "price": 999}
→ 驗證：price > 0
```

### 場景 3：監控第三方服務

監控你依賴的第三方服務（Stripe、SendGrid、S3）是否正常。

## 工具選擇

### 1. Selenium / Playwright（自建）

用瀏覽器自動化工具模擬使用者。

**優點**：
- 完全可自訂
- 免費

**缺點**：
- 需要自己維護
- 需要自己部署

### 2. Grafana Synthetic Monitoring

Grafana Cloud 提供的 Synthetic Monitoring 服務。

**優點**：
- 整合 Grafana
- 多地點監控
- 簡單易用

**缺點**：
- 付費

### 3. Checkly

專門的 Synthetic Monitoring 服務。

**優點**：
- 強大的功能
- 多地點監控
- 整合 Prometheus

**缺點**：
- 付費

## 實作：用 Playwright 實作 Synthetic Monitoring

### 安裝 Playwright

```bash
npm install playwright
```

### 編寫測試腳本

```javascript
// synthetic-monitor.js
const { chromium } = require('playwright');
const { register } = require('prom-client');

// Prometheus Metrics
const checkoutSuccessGauge = new Gauge({
  name: 'checkout_success',
  help: 'Checkout process success (1=success, 0=failure)'
});

const checkoutDurationGauge = new Gauge({
  name: 'checkout_duration_seconds',
  help: 'Checkout process duration in seconds'
});

async function runCheckoutFlow() {
  const startTime = Date.now();
  let success = 0;
  
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext();
  const page = await context.newPage();
  
  try {
    // Step 1: 訪問首頁
    await page.goto('https://example.com');
    console.log('✓ Visited homepage');
    
    // Step 2: 搜尋商品
    await page.fill('input[name="search"]', 'iPhone');
    await page.click('button[type="submit"]');
    await page.waitForSelector('.product-list');
    console.log('✓ Searched for product');
    
    // Step 3: 加入購物車
    await page.click('.product-list .product:first-child .add-to-cart');
    await page.waitForSelector('.cart-count');
    const cartCount = await page.textContent('.cart-count');
    if (cartCount !== '1') throw new Error('Cart count is incorrect');
    console.log('✓ Added to cart');
    
    // Step 4: 結帳
    await page.click('.checkout-button');
    await page.waitForSelector('.checkout-form');
    await page.fill('input[name="card_number"]', '4242424242424242');  // 測試卡號
    await page.fill('input[name="exp_month"]', '12');
    await page.fill('input[name="exp_year"]', '2025');
    await page.fill('input[name="cvc"]', '123');
    await page.click('button[type="submit"]');
    console.log('✓ Filled checkout form');
    
    // Step 5: 確認訂單
    await page.waitForSelector('.order-confirmation');
    const orderNumber = await page.textContent('.order-number');
    if (!orderNumber) throw new Error('Order number is missing');
    console.log(`✓ Order confirmed: ${orderNumber}`);
    
    success = 1;
  } catch (error) {
    console.error('✗ Checkout flow failed:', error.message);
    success = 0;
  } finally {
    await browser.close();
    
    const duration = (Date.now() - startTime) / 1000;
    checkoutSuccessGauge.set(success);
    checkoutDurationGauge.set(duration);
    
    console.log(`Duration: ${duration}s, Success: ${success}`);
  }
}

// 每 5 分鐘執行一次
setInterval(runCheckoutFlow, 5 * 60 * 1000);
runCheckoutFlow();  // 立即執行一次

// 提供 Metrics 端點
const express = require('express');
const app = express();
app.get('/metrics', (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(register.metrics());
});
app.listen(9090, () => console.log('Metrics server running on :9090'));
```

### 部署到 Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synthetic-monitor
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: synthetic-monitor
        image: your-registry/synthetic-monitor:latest
        ports:
        - containerPort: 9090
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: synthetic-monitor
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
spec:
  ports:
  - port: 9090
  selector:
    app: synthetic-monitor
```

### Prometheus 抓取 Metrics

```yaml
scrape_configs:
  - job_name: 'synthetic-monitor'
    static_configs:
      - targets: ['synthetic-monitor:9090']
```

### 告警規則

```yaml
groups:
  - name: synthetic
    rules:
      - alert: CheckoutFlowFailed
        expr: checkout_success == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Checkout flow has been failing for 5 minutes"
      
      - alert: CheckoutFlowSlow
        expr: checkout_duration_seconds > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Checkout flow is taking more than 30 seconds"
```

## 實戰：API Synthetic Monitoring

### K6 腳本

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Gauge } from 'k6/metrics';

const apiSuccessRate = new Gauge('api_success_rate');
const apiDuration = new Gauge('api_duration_seconds');

export default function () {
  // Test 1: 建立訂單
  const createOrderResponse = http.post('https://api.example.com/orders', JSON.stringify({
    product_id: 123,
    quantity: 1
  }), {
    headers: { 'Content-Type': 'application/json' }
  });
  
  check(createOrderResponse, {
    'status is 200': (r) => r.status === 200,
    'response has order_id': (r) => JSON.parse(r.body).order_id !== undefined,
    'response time < 1s': (r) => r.timings.duration < 1000
  });
  
  const orderId = JSON.parse(createOrderResponse.body).order_id;
  
  // Test 2: 查詢訂單
  const getOrderResponse = http.get(`https://api.example.com/orders/${orderId}`);
  
  check(getOrderResponse, {
    'status is 200': (r) => r.status === 200,
    'order status is pending': (r) => JSON.parse(r.body).status === 'pending'
  });
  
  // Test 3: 付款
  const paymentResponse = http.post(`https://api.example.com/orders/${orderId}/pay`, JSON.stringify({
    card_number: '4242424242424242'
  }), {
    headers: { 'Content-Type': 'application/json' }
  });
  
  check(paymentResponse, {
    'status is 200': (r) => r.status === 200,
    'payment successful': (r) => JSON.parse(r.body).status === 'paid'
  });
  
  sleep(1);
}
```

### 持續執行

```bash
# 每 5 分鐘執行一次
while true; do
  k6 run --quiet --summary-export=metrics.json api-test.js
  sleep 300
done
```

## Grafana 整合

### Dashboard

```json
{
  "panels": [
    {
      "title": "Checkout Flow Success Rate (24h)",
      "targets": [{
        "expr": "avg_over_time(checkout_success[24h]) * 100"
      }],
      "type": "stat",
      "unit": "percent"
    },
    {
      "title": "Checkout Flow Duration",
      "targets": [{
        "expr": "checkout_duration_seconds"
      }],
      "type": "graph"
    }
  ]
}
```

## 多地點 Synthetic Monitoring

### 架構

```
Synthetic Monitor (北京) → 每 5 分鐘執行測試
Synthetic Monitor (上海) → 每 5 分鐘執行測試
Synthetic Monitor (美國) → 每 5 分鐘執行測試
```

### Prometheus 查詢

```promql
# 各地點的成功率
checkout_success{job="synthetic-monitor"} by (region)

# 各地點的回應時間
checkout_duration_seconds{job="synthetic-monitor"} by (region)
```

## 實戰建議

### 1. 測試關鍵業務流程

不要測試所有功能，只測試關鍵的：
- 註冊
- 登入
- 結帳
- 付款

### 2. 使用測試帳號

不要用真實使用者的資料。

### 3. 清理測試資料

定期清理 Synthetic Monitoring 產生的測試資料（訂單、使用者）。

### 4. 設定合理的頻率

- 關鍵流程：每 5 分鐘
- 次要流程：每 1 小時

---

**Synthetic Monitoring 可以在使用者發現之前，主動發現問題。**

當你的結帳流程在凌晨 3 點壞掉時，Synthetic Monitoring 可以立即告警，而不是等到早上使用者抱怨。
