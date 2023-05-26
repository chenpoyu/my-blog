---
layout: post
title: "告警策略：日誌告警的藝術"
date: 2023-05-26 09:00:00 +0800
categories: [Observability, Logging, Alerting]
tags: [Elasticsearch, Kibana, Watcher, Alerting, On-call]
---

你的告警系統是幫助你，還是騷擾你？

我見過最糟的告警系統：**每天發送 500 條告警，但沒人看。**

今天我們來談談如何設計**有效的日誌告警策略**。

## 告警的三大問題

### 問題 1：Alert Fatigue（告警疲勞）

```
凌晨 3 點：收到 50 條告警
凌晨 3:05：全部標記為已讀，繼續睡覺
早上 9 點：發現昨晚系統真的掛了
```

**原因**：太多無用的告警，導致團隊麻痺。

### 問題 2：False Positive（誤報）

```
告警：資料庫連線失敗！
實際情況：健康檢查的連線失敗（每分鐘 1 次），不影響業務
```

**原因**：沒有正確區分「錯誤」和「問題」。

### 問題 3：No Alert（漏報）

```
使用者：你們的網站是不是掛了？
工程師：咦，怎麼沒收到告警？
```

**原因**：只監控技術指標，沒監控業務指標。

## 好的告警策略的五個原則

### 1. Actionable（可行動的）

**壞告警**：
```
[ERROR] 資料庫查詢失敗
```

收到這個告警，你要做什麼？不知道。

**好告警**：
```
[CRITICAL] 訂單資料庫連線池耗盡（100/100）
可能影響：使用者無法下單
建議行動：
1. 重啟 order-service
2. 檢查是否有慢查詢（SELECT * FROM orders WHERE ...)
3. 如果 5 分鐘內未恢復，escalate 到資深工程師
```

### 2. Specific（具體的）

**壞告警**：
```
系統錯誤數量過高
```

**好告警**：
```
payment-service 的 PaymentTimeoutException 在過去 5 分鐘內發生 50 次（正常值 < 5）
```

### 3. Contextual（有上下文的）

告警要包含足夠的資訊，讓你不用登入系統就能判斷嚴重性。

```
[CRITICAL] order-service 錯誤率：5.2%（正常值 < 0.5%）
時間：2023-05-26 10:15:23
影響範圍：所有使用者
受影響的 API：POST /api/orders
錯誤類型：PaymentTimeoutException
Dashboard：https://kibana.example.com/dashboard/order-service
Runbook：https://wiki.example.com/runbook/payment-timeout
```

### 4. Prioritized（有優先級的）

不是所有錯誤都要半夜把人叫起來。

| 優先級 | 說明 | 通知方式 | 範例 |
|--------|------|----------|------|
| **P1 (Critical)** | 影響所有使用者，立即處理 | 電話 + SMS | 網站完全無法訪問 |
| **P2 (High)** | 影響部分使用者，1 小時內處理 | PagerDuty | 付款功能失敗率 > 5% |
| **P3 (Medium)** | 影響有限，當天處理 | Email + Slack | 日誌儲存空間 > 85% |
| **P4 (Low)** | 不影響業務，下週處理 | 只記錄 | 某個 API 的 P99 延遲略高 |

### 5. Avoiding Noise（避免雜訊）

**使用 Aggregation 而非單一事件**：

壞：
```
每次錯誤都發送一條告警 → 1 分鐘內收到 100 條告警
```

好：
```
5 分鐘內錯誤數量 > 50 次才發送一條告警
```

## Kibana Watcher 實戰

### 範例 1：錯誤率超過閾值

{% raw %}
```json
PUT _watcher/watch/error-rate-alert
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "filter": [
                { "term": { "service": "payment-service" } },
                { "range": { "@timestamp": { "gte": "now-5m" } } }
              ]
            }
          },
          "aggs": {
            "total": {
              "value_count": { "field": "_id" }
            },
            "errors": {
              "filter": { "term": { "level": "ERROR" } }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": """
        def total = ctx.payload.aggregations.total.value;
        def errors = ctx.payload.aggregations.errors.doc_count;
        if (total == 0) return false;
        def errorRate = errors / total * 100;
        return errorRate > 5.0;
      """
    }
  },
  "actions": {
    "notify_slack": {
      "slack": {
        "message": {
          "to": ["#alerts"],
          "text": """
[CRITICAL] payment-service 錯誤率過高
錯誤率：{{ctx.payload.aggregations.errors.doc_count}} / {{ctx.payload.aggregations.total.value}} = {{ctx.payload.error_rate}}%
時間：{{ctx.execution_time}}
Dashboard：https://kibana.example.com/app/dashboards#/view/payment-service
          """
        }
      }
    },
    "page_oncall": {
      "pagerduty": {
        "event_type": "trigger",
        "description": "payment-service error rate > 5%",
        "severity": "critical"
      }
    }
  },
  "throttle_period": "15m"  // 15 分鐘內只發送一次
}
```
{% endraw %}

### 範例 2：特定錯誤突然暴增

{% raw %}
```json
PUT _watcher/watch/payment-timeout-spike
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "filter": [
                { "term": { "errorType": "PaymentTimeoutException" } },
                { "range": { "@timestamp": { "gte": "now-5m" } } }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total.value": {
        "gt": 50  // 5 分鐘內超過 50 次
      }
    }
  },
  "actions": {
    "notify_slack": {
      "slack": {
        "message": {
          "to": ["#alerts"],
          "text": "PaymentTimeoutException 在過去 5 分鐘內發生 {{ctx.payload.hits.total.value}} 次"
        }
      }
    }
  }
}
```
{% endraw %}

### 範例 3：異常的日誌模式

使用 Machine Learning 檢測異常。

{% raw %}
```json
PUT _watcher/watch/anomaly-detection
{
  "trigger": {
    "schedule": {
      "interval": "15m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [".ml-anomalies-*"],
        "body": {
          "size": 1,
          "query": {
            "bool": {
              "filter": [
                { "range": { "timestamp": { "gte": "now-15m" } } },
                { "term": { "result_type": "record" } },
                { "range": { "record_score": { "gte": 75 } } }
              ]
            }
          },
          "sort": [
            { "record_score": "desc" }
          ]
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total.value": {
        "gt": 0
      }
    }
  },
  "actions": {
    "notify_slack": {
      "slack": {
        "message": {
          "to": ["#alerts"],
          "text": """
檢測到異常的日誌模式
異常分數：{{ctx.payload.hits.hits.0._source.record_score}}
影響欄位：{{ctx.payload.hits.hits.0._source.function_name}}
          """
        }
      }
    }
  }
}
```
{% endraw %}

## 告警的進階技巧

### 1. Baseline Comparison（基準比較）

不要只看絕對值，要看相對變化。

**壞告警**：
```
錯誤數量 > 100
```

**好告警**：
```
錯誤數量比過去 7 天的平均值高 3 倍
```

**實作**：

```json
{
  "aggs": {
    "current": {
      "filter": {
        "range": { "@timestamp": { "gte": "now-5m" } }
      }
    },
    "baseline": {
      "filter": {
        "range": { "@timestamp": { "gte": "now-7d", "lte": "now-5m" } }
      }
    }
  }
}
```

### 2. Suppression（抑制）

如果已經有一個 P1 告警，就不要再發送 P2、P3 告警。

**實作**：

在告警中加入 `throttle_period`：

```json
{
  "throttle_period": "1h"  // 1 小時內只發送一次
}
```

或使用外部工具如 **PagerDuty** 的告警分組功能。

### 3. Escalation（升級）

如果 15 分鐘內問題沒有被處理，自動升級到資深工程師。

**實作**：

在 PagerDuty 中設定 **Escalation Policy**：

```
Level 1: On-call engineer（立即通知）
Level 2: Senior engineer（15 分鐘後升級）
Level 3: Engineering manager（30 分鐘後升級）
```

### 4. Root Cause Linking（關聯根因）

如果多個服務同時出現錯誤，很可能是同一個根因。

**範例**：

```
[ERROR] order-service: Database connection timeout
[ERROR] payment-service: Database connection timeout
[ERROR] user-service: Database connection timeout

→ 根因：資料庫掛了
```

**實作**：

用 Elasticsearch 的 **Significant Terms Aggregation** 找出共同的錯誤模式。

## 實戰：從每天 500 條告警降到 10 條

### 問題

我們的團隊每天收到 500+ 條告警，但 95% 都是誤報。

### 分析

```bash
GET logs-*/_search
{
  "size": 0,
  "query": {
    "term": { "alertType": "sent" }
  },
  "aggs": {
    "by_alert_name": {
      "terms": {
        "field": "alertName.keyword",
        "size": 20
      }
    }
  }
}
```

發現：
- **80% 的告警**來自「健康檢查失敗」（實際上不影響業務）
- **15% 的告警**來自「單次錯誤」（應該要聚合）
- **5% 的告警**是真正的問題

### 解決方案

1. **移除無用的告警**：
   - 健康檢查失敗 → 改為「連續 3 次失敗」才告警

2. **聚合單次錯誤**：
   - 從「每次錯誤都告警」改為「5 分鐘內 > 50 次」才告警

3. **加入業務指標**：
   - 監控「訂單成功率」而非「錯誤數量」

4. **設定優先級**：
   - P1：影響所有使用者 → PagerDuty
   - P2：影響部分使用者 → Slack
   - P3：不影響業務 → Email

### 結果

- **告警數量**：從每天 500 條降到 10 條
- **誤報率**：從 95% 降到 10%
- **平均回應時間**：從 30 分鐘降到 5 分鐘（因為不再被雜訊淹沒）

---

**好的告警系統應該是「安靜的」——只在真正需要你的時候才通知你。**
