---
layout: post
title: "日誌聚合分析：從海量日誌中發現規律"
date: 2023-05-19 09:00:00 +0800
categories: [Observability, Logging]
tags: [Elasticsearch, Aggregations, Kibana, Visualizations, Analytics]
---

查日誌不只是找錯誤，更重要的是**發現趨勢與規律**。

今天我們來談談如何用 Elasticsearch Aggregations 做日誌分析。

## 什麼是 Aggregations？

Aggregations 就是 Elasticsearch 的**分組統計**功能，類似 SQL 的 `GROUP BY`。

**SQL 範例**：
```sql
SELECT status_code, COUNT(*)
FROM logs
GROUP BY status_code
ORDER BY COUNT(*) DESC
LIMIT 10;
```

**Elasticsearch Aggregations**：
```json
{
  "aggs": {
    "status_codes": {
      "terms": {
        "field": "status_code",
        "size": 10,
        "order": { "_count": "desc" }
      }
    }
  }
}
```

## Aggregations 的三種類型

### 1. Metric Aggregations（指標聚合）

計算單一數值：`count`, `sum`, `avg`, `min`, `max`, `percentiles`

**範例：計算平均回應時間**

```json
{
  "aggs": {
    "avg_duration": {
      "avg": {
        "field": "duration"
      }
    }
  }
}
```

### 2. Bucket Aggregations（桶聚合）

把資料分組到不同的「桶」裡：`terms`, `date_histogram`, `range`, `histogram`

**範例：按小時統計錯誤數量**

```json
{
  "aggs": {
    "errors_per_hour": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      }
    }
  }
}
```

### 3. Pipeline Aggregations（管道聚合）

對其他 Aggregation 的結果再做計算：`derivative`, `cumulative_sum`, `moving_avg`

**範例：計算錯誤數量的變化率**

```json
{
  "aggs": {
    "errors_per_hour": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      },
      "aggs": {
        "error_rate_change": {
          "derivative": {
            "buckets_path": "_count"
          }
        }
      }
    }
  }
}
```

## 實戰案例 1：Top 10 最常見的錯誤

### 查詢

```json
GET logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-1d" } } }
      ]
    }
  },
  "aggs": {
    "top_errors": {
      "terms": {
        "field": "errorType.keyword",
        "size": 10
      }
    }
  }
}
```

### 結果

```json
{
  "aggregations": {
    "top_errors": {
      "buckets": [
        { "key": "PaymentTimeoutException", "doc_count": 1234 },
        { "key": "DatabaseConnectionException", "doc_count": 567 },
        { "key": "NullPointerException", "doc_count": 345 }
      ]
    }
  }
}
```

### Kibana 視覺化

1. **Visualize → Pie Chart**
2. **Buckets → Split slices**
3. **Aggregation → Terms**
4. **Field → errorType.keyword**
5. **Size → 10**

結果：一個圓餅圖顯示最常見的錯誤類型。

## 實戰案例 2：回應時間的 P50 / P95 / P99

### 查詢

```json
GET logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "endpoint.keyword": "/api/orders" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "latency_percentiles": {
      "percentiles": {
        "field": "duration",
        "percents": [50, 95, 99]
      }
    }
  }
}
```

### 結果

```json
{
  "aggregations": {
    "latency_percentiles": {
      "values": {
        "50.0": 123.5,
        "95.0": 456.2,
        "99.0": 1234.8
      }
    }
  }
}
```

**解讀**：
- **P50**：50% 的請求在 123ms 內完成
- **P95**：95% 的請求在 456ms 內完成
- **P99**：99% 的請求在 1234ms 內完成

## 實戰案例 3：每小時的錯誤趨勢

### 查詢

```json
GET logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      }
    }
  }
}
```

### 結果

```json
{
  "aggregations": {
    "errors_over_time": {
      "buckets": [
        { "key_as_string": "2023-05-19T00:00:00", "doc_count": 123 },
        { "key_as_string": "2023-05-19T01:00:00", "doc_count": 145 },
        { "key_as_string": "2023-05-19T02:00:00", "doc_count": 987 }  // 突然暴增！
      ]
    }
  }
}
```

發現：凌晨 2 點錯誤數量暴增 6 倍！進一步調查原因。

### Kibana 視覺化

1. **Visualize → Line Chart**
2. **X-axis → Date Histogram**
3. **Field → @timestamp**
4. **Interval → Hourly**
5. **Y-axis → Count**

結果：一條折線圖顯示錯誤數量隨時間的變化。

## 實戰案例 4：按服務統計錯誤率

### 查詢

```json
GET logs-*/_search
{
  "size": 0,
  "query": {
    "range": { "@timestamp": { "gte": "now-1h" } }
  },
  "aggs": {
    "by_service": {
      "terms": {
        "field": "service.keyword",
        "size": 10
      },
      "aggs": {
        "total": {
          "value_count": {
            "field": "_id"
          }
        },
        "errors": {
          "filter": {
            "term": { "level": "ERROR" }
          }
        },
        "error_rate": {
          "bucket_script": {
            "buckets_path": {
              "errors": "errors>_count",
              "total": "total"
            },
            "script": "params.errors / params.total * 100"
          }
        }
      }
    }
  }
}
```

### 結果

```json
{
  "aggregations": {
    "by_service": {
      "buckets": [
        {
          "key": "order-service",
          "total": { "value": 10000 },
          "errors": { "doc_count": 50 },
          "error_rate": { "value": 0.5 }  // 0.5%
        },
        {
          "key": "payment-service",
          "total": { "value": 5000 },
          "errors": { "doc_count": 250 },
          "error_rate": { "value": 5.0 }  // 5%，異常！
        }
      ]
    }
  }
}
```

發現：**payment-service 的錯誤率是 order-service 的 10 倍！**

## 實戰案例 5：地理位置分佈

假設你用 GeoIP 解析了使用者的位置。

### 查詢

```json
GET logs-*/_search
{
  "size": 0,
  "query": {
    "range": { "@timestamp": { "gte": "now-1d" } }
  },
  "aggs": {
    "by_country": {
      "terms": {
        "field": "geoip.country_name.keyword",
        "size": 10
      }
    }
  }
}
```

### Kibana 視覺化

1. **Visualize → Map**
2. **Add layer → Choropleth**
3. **Field → geoip.country_name.keyword**
4. **Metric → Count**

結果：一個世界地圖，顏色深淺代表請求數量。

## 巢狀 Aggregations

你可以把 Aggregations 巢狀起來，做更複雜的分析。

### 範例：每個服務的 Top 10 錯誤類型

```json
{
  "aggs": {
    "by_service": {
      "terms": {
        "field": "service.keyword",
        "size": 10
      },
      "aggs": {
        "top_errors": {
          "terms": {
            "field": "errorType.keyword",
            "size": 10
          }
        }
      }
    }
  }
}
```

### 結果

```json
{
  "aggregations": {
    "by_service": {
      "buckets": [
        {
          "key": "order-service",
          "doc_count": 1000,
          "top_errors": {
            "buckets": [
              { "key": "PaymentTimeoutException", "doc_count": 500 },
              { "key": "DatabaseException", "doc_count": 300 }
            ]
          }
        }
      ]
    }
  }
}
```

## Kibana Dashboard 實戰

讓我們建立一個「Service Health Dashboard」。

### Panel 1：錯誤趨勢（Line Chart）

- **X-axis**：`@timestamp`（Hourly）
- **Y-axis**：Count
- **Filter**：`level:ERROR`
- **Split series**：`service.keyword`

### Panel 2：錯誤類型分佈（Pie Chart）

- **Split slices**：`errorType.keyword`（Top 10）
- **Filter**：`level:ERROR`

### Panel 3：回應時間（Heat Map）

- **X-axis**：`@timestamp`（Hourly）
- **Y-axis**：`endpoint.keyword`（Top 10）
- **Color**：`duration`（Average）

### Panel 4：服務錯誤率（Data Table）

- **Rows**：`service.keyword`
- **Metrics**：Count, Error Count, Error Rate

### Panel 5：地理分佈（Map）

- **Choropleth**：`geoip.country_name.keyword`

## 效能優化

### 1. 限制 Aggregation 的範圍

```json
{
  "size": 0,  // 不返回原始文件（只要聚合結果）
  "query": {
    "range": { "@timestamp": { "gte": "now-1h" } }  // 限制時間範圍
  }
}
```

### 2. 使用 Keyword 欄位

```json
// 慢
"field": "errorType"

// 快
"field": "errorType.keyword"
```

### 3. 調整 Shard Size

```json
{
  "aggs": {
    "top_errors": {
      "terms": {
        "field": "errorType.keyword",
        "size": 10,
        "shard_size": 100  // 每個 Shard 返回 Top 100，然後合併成 Top 10
      }
    }
  }
}
```

這樣可以提升準確度，但會增加記憶體使用量。

---

**Aggregations 讓你從「找問題」進化到「發現問題」。**

當你能看到整體趨勢，就不會再陷入「頭痛醫頭」的困境。
