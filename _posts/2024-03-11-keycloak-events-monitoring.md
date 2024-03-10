---
layout: post
title: "Keycloak 事件監控與日誌分析"
date: 2024-03-11 16:10:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, Monitoring, Events]
---

這週深入研究了 Keycloak 的事件系統。正式環境運作後，需要知道誰登入了、誰登入失敗、有沒有異常的登入行為等等。Keycloak 的 Event 機制可以記錄這些資訊。

## Event 的兩種類型

Keycloak 的事件分兩種：

**Login Events**：使用者相關的事件，像是登入、登出、註冊、密碼變更、Token 刷新等。這些事件會包含使用者資訊、IP 位址、Client ID 等。

**Admin Events**：管理操作的事件，像是建立使用者、修改 Client 設定、刪除 Role 等。記錄了誰在什麼時候做了什麼管理操作。

兩種事件分開管理，可以針對不同需求設定不同的保存策略。

## 啟用 Event Logging

進入 Realm settings → Events：

**Login Events Settings**：
1. 把 `Save Events` 打開
2. 選擇要記錄的事件類型（預設全選）
3. 設定 `Expiration`，比如保留 7 天或 30 天
4. 點 Save

**Admin Events Settings**：
1. 把 `Save Events` 打開
2. 決定要不要 `Include Representation`（會記錄完整的物件內容，比較占空間）
3. 點 Save

設定好後，切到 Events 頁籤就能看到即時的事件記錄。

## 查看登入事件

在 Events 頁籤，可以看到所有的登入相關事件：

```
LOGIN              testuser    192.168.1.100    spring-app
LOGIN_ERROR        john.doe    192.168.1.101    spring-app
REFRESH_TOKEN      testuser    192.168.1.100    spring-app
LOGOUT             testuser    192.168.1.100    spring-app
```

點進去可以看到詳細資訊，包括：
- 事件類型
- Realm 名稱
- Client ID
- 使用者 ID 和 Username
- IP 位址
- 時間戳
- 錯誤訊息（如果有）

這對追蹤問題很有幫助。比如使用者說登入不了，可以直接查 Events，看是密碼錯誤還是帳號被鎖定。

## 查看管理事件

切到 Admin Events 頁籤，可以看到所有的管理操作：

```
CREATE   admin   users/123-456-789
UPDATE   admin   clients/abc-def-123
DELETE   admin   roles/finance-manager
```

如果有啟用 `Include Representation`，點進去可以看到完整的 JSON，包括修改前後的內容。

這對稽核很重要。如果有安全事件，可以追查是誰改了什麼設定。

## Event Listener

Keycloak 可以把事件發送到外部系統，這樣就不用一直進管理介面查。

內建的 Event Listener 有：
- **jboss-logging**：輸出到 Keycloak 的 log 檔
- **email**：發送 email 通知（需要設定 SMTP）

在 Realm settings → Events → Event Listeners 可以啟用。

不過內建的功能比較陽春，實務上通常會整合到 ELK 或其他監控系統。

## 整合 ELK Stack

要把 Keycloak 的事件送到 Elasticsearch，有幾種做法：

**方法一：解析 Log 檔**

啟用 jboss-logging 後，事件會輸出到 `standalone/log/server.log`。用 Filebeat 收集 log，送到 Logstash 解析，再存到 Elasticsearch。

這種方式比較簡單，但 log 格式不太好解析，而且混雜了其他 log。

**方法二：自訂 Event Listener SPI**

寫一個 Keycloak 的 Extension，直接把事件送到 Elasticsearch 或 Kafka。

```java
public class ElasticsearchEventListener implements EventListenerProvider {
    
    private RestHighLevelClient elasticsearchClient;
    
    @Override
    public void onEvent(Event event) {
        IndexRequest request = new IndexRequest("keycloak-events")
            .source("type", event.getType().toString(),
                    "realm", event.getRealmId(),
                    "userId", event.getUserId(),
                    "ipAddress", event.getIpAddress(),
                    "timestamp", event.getTime());
        
        try {
            elasticsearchClient.index(request, RequestOptions.DEFAULT);
        } catch (IOException e) {
            // Handle error
        }
    }
    
    // ... 其他方法
}
```

這種方式比較彈性，但要自己寫 code 和打包部署。

**方法三：用 Graylog 或 Fluentd**

如果原本就有用 Graylog 或 Fluentd 收集 log，可以直接設定收集 Keycloak 的 log 檔，用內建的 parser 解析。

## 監控常見的異常行為

整合到監控系統後，可以設定一些告警規則：

**登入失敗次數異常**：
- 同一帳號短時間內多次登入失敗 → 可能是暴力破解
- 同一 IP 嘗試多個不同帳號 → 可能是掃描攻擊

**異常登入地點**：
- 使用者平常都從台灣登入，突然從陌生國家登入 → 可能是帳號被盜

**管理操作異常**：
- 非上班時間有管理操作 → 需要確認是否為合法操作
- 短時間內大量建立使用者 → 可能是腳本攻擊

**Token 相關**：
- Token 刷新頻率異常 → 可能有程式邏輯問題
- 過期 Token 還在使用 → 可能有人在 replay attack

## 實際設定告警

用 Elasticsearch + Kibana 的話，可以在 Kibana 設定 Watcher：

{% raw %}
```json
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["keycloak-events"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"term": {"type": "LOGIN_ERROR"}},
                {"range": {"timestamp": {"gte": "now-5m"}}}
              ]
            }
          },
          "aggs": {
            "by_user": {
              "terms": {"field": "userId"}
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.aggregations.by_user.buckets.0.doc_count": {
        "gte": 5
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "security@example.com",
        "subject": "Keycloak: Multiple login failures detected",
        "body": "User {{ctx.payload.aggregations.by_user.buckets.0.key}} has {{ctx.payload.aggregations.by_user.buckets.0.doc_count}} failed login attempts in the last 5 minutes."
      }
    }
  }
}
```
{% endraw %}

這個規則會每 5 分鐘檢查一次，如果發現同一使用者在 5 分鐘內登入失敗超過 5 次，就發 email 通知。

## Prometheus Metrics

除了 Events，Keycloak 也可以輸出 Prometheus metrics，監控系統效能。

要啟用 metrics，需要安裝 `keycloak-metrics-spi` 這個 extension。下載 jar 檔放到 `providers/` 目錄，重啟 Keycloak。

然後可以從 `/metrics` endpoint 拿到 metrics：

```
# HELP keycloak_logins_total Total successful logins
# TYPE keycloak_logins_total counter
keycloak_logins_total{realm="company",client="spring-app"} 1234

# HELP keycloak_login_errors_total Total failed login attempts
# TYPE keycloak_login_errors_total counter
keycloak_login_errors_total{realm="company",client="spring-app",error="invalid_credentials"} 56

# HELP keycloak_response_time_seconds Response time in seconds
# TYPE keycloak_response_time_seconds histogram
keycloak_response_time_seconds_bucket{le="0.1"} 890
keycloak_response_time_seconds_bucket{le="0.5"} 980
```

在 Prometheus 設定檔加上 Keycloak 的 target：

```yaml
scrape_configs:
  - job_name: 'keycloak'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'
```

然後就可以在 Grafana 做儀表板，監控 Keycloak 的健康狀態。

## 實務心得

事件監控設定好後，真的幫助很大。之前遇到使用者說登入有問題，都要猜半天。現在直接查 Events，馬上就知道是什麼問題。

不過 Event 資料量會很大，特別是 Token 刷新這種高頻事件。要設定合理的 Expiration，或者只記錄重要的事件類型。

另外要注意隱私問題，Event 裡會包含 IP 位址、User Agent 等資訊，依照 GDPR 或其他隱私法規，可能需要做匿名化或定期清除。

下週想研究 Keycloak 的 Session 管理，看怎麼處理同時登入多個裝置、強制登出等場景。這些對企業應用來說蠻重要的。
