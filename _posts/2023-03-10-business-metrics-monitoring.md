---
layout: post
title: "業務指標監控：讓數據驅動產品決策"
date: 2023-03-10 09:00:00 +0800
categories: [Observability, Monitoring]
tags: [Business Metrics, KPI, Product Analytics, Monitoring]
---

前面幾週我們學了監控系統資源、API 效能、資料庫狀態。但如果只看這些，你永遠不知道**使用者真正在乎什麼**。

今天我們要跳出技術層面，談「業務指標監控」——讓數據告訴你產品是否成功。

## 技術指標 vs 業務指標

先看一個真實案例：

某電商平台的技術指標一切正常：
- ✅ API 回應時間 < 200ms
- ✅ 錯誤率 < 0.1%
- ✅ CPU 使用率 60%

但業務指標卻在下滑：
- ❌ 訂單轉換率從 5% 降到 3%
- ❌ 客單價從 $100 降到 $80
- ❌ 使用者留存率下降 20%

**問題在哪？技術團隊根本不知道這些事正在發生。**

## 什麼是業務指標？

業務指標（Business Metrics）是「對公司有直接影響」的數據。

### 電商範例

```promql
# 訂單數
increase(orders_created_total[1h])

# 訂單轉換率
increase(orders_completed_total[1h]) / increase(orders_created_total[1h]) * 100

# 總營收
sum(order_amount_total)

# 客單價（Average Order Value）
sum(order_amount_total) / count(orders_completed_total)

# 購物車放棄率
(increase(cart_created_total[1h]) - increase(orders_created_total[1h])) / increase(cart_created_total[1h]) * 100
```

### SaaS 範例

```promql
# 新註冊使用者數
increase(users_registered_total[1d])

# 活躍使用者數（DAU）
count(rate(user_actions_total[24h]) > 0)

# 付費轉換率
count(users{plan="paid"}) / count(users) * 100

# 流失率（Churn Rate）
increase(users_cancelled_total[30d]) / count(users{plan="paid"})

# 月經常性收入（MRR）
sum(subscription_amount_monthly)
```

### 內容平台範例

```promql
# 每日活躍使用者（DAU）
count(rate(user_sessions_total[24h]) > 0)

# 使用者參與度（Engagement）
sum(rate(content_views_total[1h])) / count(users)

# 內容發佈數
increase(content_published_total[1h])

# 平均觀看時長
sum(rate(content_view_duration_seconds[1h])) / sum(rate(content_views_total[1h]))
```

## 如何在程式碼中埋點？

### 範例 1：電商訂單流程（Java）

```java
@Service
public class OrderService {
    
    // 定義業務指標
    private static final Counter ordersCreated = Counter.build()
        .name("orders_created_total")
        .help("Total orders created")
        .labelNames("product_category", "payment_method")
        .register();
    
    private static final Counter ordersCompleted = Counter.build()
        .name("orders_completed_total")
        .help("Total orders completed")
        .labelNames("product_category", "payment_method")
        .register();
    
    private static final Counter orderRevenue = Counter.build()
        .name("order_amount_total")
        .help("Total revenue")
        .labelNames("product_category", "currency")
        .register();
    
    private static final Histogram orderValue = Histogram.build()
        .name("order_value_dollars")
        .help("Order value distribution")
        .buckets(10, 50, 100, 500, 1000, 5000)
        .register();
    
    public Order createOrder(CreateOrderRequest request) {
        ordersCreated.labels(
            request.getProductCategory(),
            request.getPaymentMethod()
        ).inc();
        
        Order order = orderRepository.save(request);
        return order;
    }
    
    public void completeOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        ordersCompleted.labels(
            order.getProductCategory(),
            order.getPaymentMethod()
        ).inc();
        
        orderRevenue.labels(
            order.getProductCategory(),
            order.getCurrency()
        ).inc(order.getAmount());
        
        orderValue.observe(order.getAmount());
        
        // 業務邏輯...
    }
}
```

### 範例 2：使用者行為追蹤（.NET Core）

```csharp
public class UserActivityService
{
    private static readonly Counter _userRegistrations = Metrics.CreateCounter(
        "users_registered_total",
        "Total user registrations",
        new CounterConfiguration
        {
            LabelNames = new[] { "source", "plan" }
        });
    
    private static readonly Counter _userActions = Metrics.CreateCounter(
        "user_actions_total",
        "Total user actions",
        new CounterConfiguration
        {
            LabelNames = new[] { "action_type", "user_id" }
        });
    
    private static readonly Gauge _activeUsers = Metrics.CreateGauge(
        "users_active_current",
        "Current active users");
    
    public async Task<User> RegisterUser(RegisterRequest request)
    {
        var user = new User { /* ... */ };
        await _userRepository.SaveAsync(user);
        
        _userRegistrations.WithLabels(
            request.Source,  // "google", "facebook", "direct"
            request.Plan     // "free", "pro", "enterprise"
        ).Inc();
        
        return user;
    }
    
    public void TrackAction(string userId, string actionType)
    {
        _userActions.WithLabels(actionType, userId).Inc();
    }
    
    public void UpdateActiveUsers(int count)
    {
        _activeUsers.Set(count);
    }
}
```

## 北極星指標（North Star Metric）

每個產品都應該有一個「北極星指標」——最能代表產品價值的單一指標。

### 為什麼需要北極星指標？

因為你不可能同時優化 100 個指標。北極星指標讓團隊聚焦。

### 常見的北極星指標

| 產品類型 | 北極星指標 |
|---------|-----------|
| 電商 | 月活躍買家數 (Monthly Active Buyers) |
| SaaS | 週活躍使用者數 (Weekly Active Users) |
| 社群媒體 | 每日互動次數 (Daily Engagements) |
| 串流平台 | 每週觀看時數 (Weekly Watch Hours) |
| 市集平台 | 每月成交訂單數 (Monthly Completed Orders) |

### 如何選擇北極星指標？

問三個問題：

1. **是否反映產品的核心價值？**（不是虛榮指標）
2. **是否可衡量？**（能用數據追蹤）
3. **團隊是否能影響它？**（不是完全外部因素）

### 實戰案例：Spotify 的北極星指標

Spotify 的北極星指標是：**每月聽音樂的時數（Monthly Music Listening Hours）**

為什麼不是「註冊使用者數」？
- 因為註冊了但不聽音樂，對產品沒價值
- 只有真正使用產品，才會付費、才會留存

監控實作：

```promql
# 北極星指標
sum(increase(music_play_duration_seconds[30d])) / 3600  # 轉換成小時

# 拆解指標
sum(increase(music_play_duration_seconds[30d])) by (user_segment)  # 按使用者分群
sum(increase(music_play_duration_seconds[30d])) by (device_type)   # 按裝置類型
```

## 漏斗分析（Funnel Analysis）

很多業務流程是「漏斗狀」的，每一步都會流失使用者。

### 電商購物漏斗

```
瀏覽商品 (1000 人)
    ↓ (70% 轉換)
加入購物車 (700 人)
    ↓ (50% 轉換)
填寫資料 (350 人)
    ↓ (80% 轉換)
完成付款 (280 人)
```

**總轉換率：28%**

### Prometheus 實作

```java
// 定義各階段指標
private static final Counter productViews = Counter.build()
    .name("funnel_product_views_total").register();

private static final Counter addToCart = Counter.build()
    .name("funnel_add_to_cart_total").register();

private static final Counter checkout = Counter.build()
    .name("funnel_checkout_total").register();

private static final Counter payment = Counter.build()
    .name("funnel_payment_completed_total").register();
```

### 計算各階段轉換率

```promql
# 加入購物車轉換率
increase(funnel_add_to_cart_total[1h]) / increase(funnel_product_views_total[1h]) * 100

# 結帳轉換率
increase(funnel_checkout_total[1h]) / increase(funnel_add_to_cart_total[1h]) * 100

# 付款完成率
increase(funnel_payment_completed_total[1h]) / increase(funnel_checkout_total[1h]) * 100

# 整體轉換率
increase(funnel_payment_completed_total[1h]) / increase(funnel_product_views_total[1h]) * 100
```

### 告警：轉換率異常下降

```yaml
- alert: CheckoutConversionDropped
  expr: |
    (
      increase(funnel_checkout_total[1h]) / increase(funnel_add_to_cart_total[1h])
    ) < 0.4
  for: 30m
  annotations:
    summary: "結帳轉換率下降至 {{ $value | humanizePercentage }}"
    description: "正常應該在 50% 以上，可能付款流程有問題"
```

## 實時業務 Dashboard

在 Grafana 建立一個「業務健康度 Dashboard」：

### Panel 1：營收儀表板

```promql
# 今日營收
sum(increase(order_amount_total[1d]))

# 與昨天比較
(
  sum(increase(order_amount_total[1d]))
  /
  sum(increase(order_amount_total[1d] offset 1d))
  - 1
) * 100
```

顯示為 **Stat Panel**，設定閾值：
- 綠色：成長 > 0%
- 黃色：-10% ~ 0%
- 紅色：< -10%

### Panel 2：訂單轉換漏斗

使用 **Bar Gauge** 顯示：

```promql
increase(funnel_product_views_total[1h])       # 瀏覽
increase(funnel_add_to_cart_total[1h])         # 加入購物車
increase(funnel_checkout_total[1h])            # 結帳
increase(funnel_payment_completed_total[1h])   # 完成付款
```

### Panel 3：使用者分佈（Pie Chart）

```promql
sum by (user_segment) (users_active_current)
```

### Panel 4：每小時營收趨勢

```promql
sum(increase(order_amount_total[1h]))
```

## 用監控驅動產品決策

這不是理論，讓我分享真實案例。

### 案例 1：發現付款流程問題

某天我們看到告警：

> **CheckoutConversionDropped: 結帳轉換率下降至 30%**

立刻查詢：

```promql
# 哪個付款方式出問題？
sum(increase(funnel_checkout_total[1h])) by (payment_method)
```

發現「信用卡付款」的數量異常少。

再查日誌，發現第三方金流 API 在某個時段大量超時。

**行動**：
1. 緊急：切換到備用金流
2. 短期：加入金流 API 的健康檢查
3. 長期：實作 Circuit Breaker，自動切換金流

**結果**：10 分鐘內恢復，避免損失 $50,000 營收。

### 案例 2：A/B 測試驗證

產品經理想把「加入購物車」按鈕從綠色改成紅色。

我們用 Feature Flag + Prometheus 驗證：

```java
@GetMapping("/product/{id}")
public Product getProduct(@PathVariable Long id) {
    boolean useRedButton = featureFlags.isEnabled("red-button", userId);
    
    productViews.labels(
        useRedButton ? "red" : "green"
    ).inc();
    
    // ...
}
```

一週後比較：

```promql
# 綠色按鈕的轉換率
increase(funnel_add_to_cart_total{button_color="green"}[7d]) /
increase(funnel_product_views_total{button_color="green"}[7d]) * 100

# 紅色按鈕的轉換率
increase(funnel_add_to_cart_total{button_color="red"}[7d]) /
increase(funnel_product_views_total{button_color="red"}[7d]) * 100
```

**結果**：紅色按鈕轉換率多 8%，全面推出。

---

**技術團隊不應該只關心「系統有沒有掛」，而要關心「產品有沒有成功」。**

當你能在 Dashboard 上同時看到「API 回應時間」和「訂單轉換率」，你就不再只是工程師，而是產品工程師。

這才是真正的價值。
