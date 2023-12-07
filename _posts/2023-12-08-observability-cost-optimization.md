---
layout: post
title: "Observability 成本優化：不讓監控費用失控"
date: 2023-12-08 09:00:00 +0800
categories: [Observability, Cost Optimization]
tags: [Cost Optimization, FinOps, Metrics, Logs, Traces]
---

「這個月的 Observability 帳單怎麼這麼貴？！」

當你的系統規模成長，**監控成本**也會跟著暴增。

一個 1000 台機器的系統，每個月的 Observability 成本可能高達 **$10,000 - $50,000**。

## Observability 成本結構

### 1. Metrics 成本

**費用來源：**
- 時間序列數量（Cardinality）
- 資料保存時間（Retention）
- 查詢頻率

**典型費用：**
- Datadog：$0.08 per metric per month
- New Relic：$0.30 per 1000 data points
- Prometheus（自建）：儲存成本 + 運維成本

### 2. Logs 成本

**費用來源：**
- 日誌量（Volume）
- 索引數量
- 資料保存時間

**典型費用：**
- Elasticsearch（自建）：$100 - $500 per TB per month
- Splunk：$1,800 per GB per day
- Datadog：$0.10 per million log events

### 3. Traces 成本

**費用來源：**
- Span 數量
- Sampling Rate
- 資料保存時間

**典型費用：**
- Jaeger（自建）：儲存成本 + 運維成本
- Datadog APM：$31 per host per month
- New Relic：$0.30 per 1M spans

### 成本分佈

```
📊 典型 Observability 成本分佈

Logs:    50% - 70%  ████████████████████
Metrics: 20% - 30%  ████████
Traces:  10% - 20%  ████
```

## 成本優化策略

### 策略 1：降低 Metrics Cardinality

#### 問題：高 Cardinality 標籤

```yaml
# ❌ 錯誤：使用 User ID 作為標籤
http_requests_total{user_id="12345", method="GET", path="/api/orders"}
http_requests_total{user_id="67890", method="GET", path="/api/orders"}

# 問題：10 萬個使用者 = 10 萬個時間序列！
```

#### 解決方案：使用適當的標籤

```yaml
# ✅ 正確：只使用有限值的標籤
http_requests_total{method="GET", path="/api/orders", status="200"}

# 如果需要追蹤特定使用者，改用 Logs 或 Traces
```

#### Prometheus Relabeling

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'application'
    metric_relabel_configs:
      # 移除高 Cardinality 標籤
      - source_labels: [__name__]
        regex: 'http_request_duration_seconds_.*'
        action: drop
        
      # 只保留重要標籤
      - source_labels: [path]
        regex: '/api/orders/[0-9]+'
        replacement: '/api/orders/:id'
        target_label: path
```

### 策略 2：優化 Log 儲存

#### 分層儲存策略

```yaml
# Elasticsearch Index Lifecycle Management (ILM)
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**成本節省：**
- Hot：高效能 SSD，保留 7 天
- Warm：一般 SSD，保留 30 天
- Cold：HDD 或 S3，保留 90 天
- Delete：超過 90 天刪除

**節省成本：50% - 70%**

#### 只索引重要欄位

```yaml
# Elasticsearch Index Template
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "message": {"type": "text", "index": false},  # 不索引
        "trace_id": {"type": "keyword"},
        "user_id": {"type": "keyword", "index": false}  # 不索引
      }
    }
  }
}
```

### 策略 3：智慧 Sampling

#### Trace Sampling 策略

```python
# opentelemetry_config.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.trace.sampling import ParentBasedTraceIdRatioBased

# 基本 Sampling：只保留 10% 的 Traces
sampler = ParentBasedTraceIdRatioBased(0.1)

# 進階 Sampling：根據條件決定
class SmartSampler:
    def should_sample(self, span):
        # 1. 錯誤請求：100% 保留
        if span.status.is_error:
            return True
            
        # 2. 慢請求：100% 保留
        if span.duration > 1000:  # > 1s
            return True
            
        # 3. 重要端點：50% 保留
        if span.attributes.get('http.route') in ['/api/payment', '/api/order']:
            return random.random() < 0.5
            
        # 4. 一般請求：5% 保留
        return random.random() < 0.05
```

**成本節省：80% - 90%**

### 策略 4：Log Sampling

#### Application-Level Sampling

```java
// Java + Logback
import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.filter.Filter;
import ch.qos.logback.core.spi.FilterReply;

public class SamplingFilter extends Filter<ILoggingEvent> {
    
    @Override
    public FilterReply decide(ILoggingEvent event) {
        // ERROR/WARN：100% 保留
        if (event.getLevel().isGreaterOrEqual(Level.WARN)) {
            return FilterReply.ACCEPT;
        }
        
        // INFO：10% 保留
        if (event.getLevel() == Level.INFO) {
            return Math.random() < 0.1 ? FilterReply.ACCEPT : FilterReply.DENY;
        }
        
        // DEBUG：1% 保留
        return Math.random() < 0.01 ? FilterReply.ACCEPT : FilterReply.DENY;
    }
}
```

#### Fluentd Sampling

```conf
# fluentd.conf
<filter application.**>
  @type sampling
  
  # INFO logs: 10% sampling
  <rule>
    level INFO
    sample_rate 10
  </rule>
  
  # DEBUG logs: 1% sampling
  <rule>
    level DEBUG
    sample_rate 100
  </rule>
</filter>
```

**成本節省：60% - 80%**

### 策略 5：壓縮與去重

#### Prometheus Remote Write 壓縮

```yaml
# prometheus.yml
remote_write:
  - url: "https://prometheus-remote-storage:9090/api/v1/write"
    queue_config:
      capacity: 10000
      max_shards: 50
      max_samples_per_send: 5000
    
    # 啟用壓縮
    write_relabel_configs:
      - source_labels: [__name__]
        regex: 'go_.*|process_.*'
        action: drop
```

#### Elasticsearch 壓縮

```yaml
# Elasticsearch Index Settings
PUT /logs-2023-12
{
  "settings": {
    "codec": "best_compression",  # 啟用最佳壓縮
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}
```

**成本節省：30% - 50%**

## 成本監控

### Prometheus Metrics 成本監控

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

```promql
# 時間序列數量
prometheus_tsdb_symbol_table_size_bytes

# 每秒樣本數
rate(prometheus_tsdb_head_samples_appended_total[5m])

# 儲存空間
prometheus_tsdb_storage_blocks_bytes

# 預估每月成本（假設 $0.08 per metric）
prometheus_tsdb_symbol_table_size_bytes * 0.08
```

### Elasticsearch Cost Monitoring

```bash
# 檢查索引大小
GET _cat/indices?v&h=index,store.size,pri,rep&s=store.size:desc

# 檢查每日日誌量
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "logs_per_day": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "day"
      }
    }
  }
}
```

## 成本優化檢查清單

### Metrics 優化

- [ ] 移除未使用的 Metrics
- [ ] 降低 Cardinality（移除高變化標籤）
- [ ] 調整 Scrape Interval（15s → 60s）
- [ ] 使用 Recording Rules 預先計算
- [ ] 設定適當的 Retention（30d → 15d）

### Logs 優化

- [ ] 實施分層儲存（Hot/Warm/Cold）
- [ ] 只索引必要欄位
- [ ] 使用 Sampling（ERROR 100%, INFO 10%, DEBUG 1%）
- [ ] 壓縮舊資料
- [ ] 移除重複日誌

### Traces 優化

- [ ] 實施智慧 Sampling
- [ ] 只保留錯誤和慢請求的完整 Trace
- [ ] 降低 Retention（7d → 3d）
- [ ] 使用 Tail-Based Sampling
- [ ] 壓縮 Span 資料

### 基礎設施優化

- [ ] 使用 Spot Instance / Preemptible VM
- [ ] 調整 Replica 數量
- [ ] 使用物件儲存（S3/GCS）作為長期儲存
- [ ] 啟用壓縮
- [ ] 定期清理舊資料

## 實際案例：成本優化 70%

### 優化前

```
每月 Observability 成本：$50,000

Logs (Elasticsearch):     $35,000 (70%)
Metrics (Prometheus):     $10,000 (20%)
Traces (Jaeger):          $5,000  (10%)
```

### 優化措施

1. **Logs Optimization**
   - 實施 ILM 分層儲存：節省 $15,000
   - 只索引重要欄位：節省 $5,000
   - Log Sampling：節省 $5,000
   - **小計：節省 $25,000 (71%)**

2. **Metrics Optimization**
   - 降低 Cardinality：節省 $3,000
   - 調整 Scrape Interval：節省 $1,000
   - **小計：節省 $4,000 (40%)**

3. **Traces Optimization**
   - 智慧 Sampling：節省 $3,000
   - **小計：節省 $3,000 (60%)**

### 優化後

```
每月 Observability 成本：$18,000 (節省 64%)

Logs (Elasticsearch):     $10,000
Metrics (Prometheus):     $6,000
Traces (Jaeger):          $2,000
```

## 最佳實踐

### 1. 成本可見性

建立 Observability Cost Dashboard，追蹤：
- 每日/每月成本趨勢
- 各服務的成本佔比
- 成本異常警報

### 2. 成本歸屬

使用標籤追蹤成本來源：

```yaml
# 所有 Metrics 加上 Team 標籤
http_requests_total{team="order", service="order-api"}
```

### 3. 定期審查

- 每月檢查成本報告
- 移除未使用的 Dashboards 和 Alerts
- 檢查 Retention 設定是否合理

### 4. 自動化優化

```python
# cost_optimizer.py
def auto_optimize():
    # 1. 找出低價值 Metrics
    unused_metrics = find_metrics_with_no_queries(days=30)
    for metric in unused_metrics:
        drop_metric(metric)
    
    # 2. 調整 Sampling Rate
    if error_rate < 0.1:
        reduce_sampling_rate()
    elif error_rate > 1.0:
        increase_sampling_rate()
    
    # 3. 清理舊資料
    delete_data_older_than(days=90)
```

## 總結

Observability 成本優化的關鍵：

1. **降低 Cardinality**：避免高變化標籤
2. **智慧 Sampling**：保留重要資料，捨棄雜訊
3. **分層儲存**：根據資料價值選擇儲存方案
4. **定期審查**：移除未使用的監控項目
5. **成本可見性**：追蹤和歸屬成本

**記住：Observability 的目標是找出問題，而不是收集所有資料。**

---

**相關文章：**
- 《Week 35: Trace Sampling 與成本優化》
- 《Week 19: Elasticsearch Index Lifecycle Management》
- 《Week 49: Observability Maturity Model》
