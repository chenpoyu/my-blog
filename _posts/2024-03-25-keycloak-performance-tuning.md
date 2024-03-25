---
layout: post
title: "Keycloak 效能調校筆記"
date: 2024-03-25 13:50:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, Performance, Optimization]
---

這週花了些時間研究 Keycloak 的效能調校。測試環境只有幾個人用，感覺不出差異。但正式環境如果有幾百幾千個使用者同時登入，效能就很關鍵了。

做了一些壓力測試，發現預設設定在高負載下會有瓶頸。這週就來記錄一下調校的心得。

## 效能測試工具

要調校效能，首先要能量測。我用 JMeter 來做壓力測試：

建立一個測試計畫，模擬 100 個使用者同時登入：

1. Thread Group：100 threads，每秒啟動 10 個
2. HTTP Request：POST 到 `/realms/company/protocol/openid-connect/token`
3. 帶上 username、password、client_id 等參數
4. Aggregate Report：統計回應時間、吞吐量、錯誤率

跑完測試後發現：

```
平均回應時間: 850ms
90th percentile: 1200ms
95th percentile: 1500ms
吞吐量: 45 requests/sec
錯誤率: 0%
```

850ms 有點慢，理想上登入應該在 500ms 內完成。來看看哪裡可以優化。

## JVM 記憶體調整

第一個要調的是 JVM heap size。Keycloak 預設的 heap 太小，在高負載下會頻繁 GC。

編輯 `conf/keycloak.conf`：

```bash
# Heap size 設為實體記憶體的 50-75%
# 假設伺服器有 8GB RAM
JAVA_OPTS="-Xms4g -Xmx4g -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m"

# 使用 G1GC（Java 11+ 預設）
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"

# GC log（方便追蹤）
JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:file=/var/log/keycloak/gc.log:time,uptime:filecount=5,filesize=10m"
```

重啟後再測試：

```
平均回應時間: 620ms (降低 27%)
90th percentile: 850ms
吞吐量: 65 requests/sec (提升 44%)
```

效果蠻明顯的！

## 資料庫連線池

Keycloak 對資料庫的存取很頻繁，連線池的設定很重要。

檢查 PostgreSQL 的連線狀況：

```sql
SELECT count(*) FROM pg_stat_activity 
WHERE datname = 'keycloak';
```

發現只有 10 幾個連線，預設的連線池太小了。

調整 `conf/keycloak.conf`：

```properties
# 連線池大小
db-pool-initial-size=25
db-pool-min-size=25
db-pool-max-size=100

# 連線存活時間
db-pool-max-lifetime=1800000
```

這個設定會保持至少 25 個連線，高峰時可以到 100 個。

再測試：

```
平均回應時間: 480ms (再降低 23%)
90th percentile: 680ms
吞吐量: 85 requests/sec
```

持續在進步！

## Cache 優化

Keycloak 用 Infinispan 做 cache，預設設定對小規模環境夠用，但可以再調整。

編輯 `conf/cache-ispn.xml`（Keycloak 23+ 要自己建立這個檔案）：

```xml
<infinispan>
    <cache-container name="keycloak">
        <!-- Realm cache -->
        <local-cache name="realms">
            <memory max-count="1000"/>
        </local-cache>
        
        <!-- User cache -->
        <local-cache name="users">
            <memory max-count="10000"/>
        </local-cache>
        
        <!-- Session cache -->
        <distributed-cache name="sessions" owners="2">
            <expiration lifespan="-1"/>
        </distributed-cache>
    </cache-container>
</infinispan>
```

調高 cache size 可以減少資料庫查詢，但會用更多記憶體。要根據實際情況平衡。

## 密碼 Hash 演算法

Keycloak 預設用 PBKDF2 來 hash 密碼，這是故意設計成「慢」的，防止暴力破解。但 iteration 次數太高會影響登入速度。

在 Realm settings → Password Policy 可以調整：

- **Hashing Iterations**：預設是 27500 次，可以降到 10000-20000 之間

降低 iteration 會讓登入快一點，但也會降低安全性。要在安全和效能之間取捨。

測試後發現降到 20000 後：

```
平均回應時間: 380ms (再降低 21%)
```

登入變快了，但還是在可接受的安全範圍內。

## 啟用 HTTP/2

現代瀏覽器都支援 HTTP/2，可以大幅減少連線開銷。

在 Keycloak 設定檔啟用：

```properties
http-enabled=false
https-enabled=true
http2-enabled=true
```

前面的 Load Balancer 或 Reverse Proxy（Nginx/HAProxy）也要支援 HTTP/2。

## 靜態資源 Cache

Keycloak 的 JS、CSS、圖片等靜態資源可以設定 browser cache，減少重複請求。

如果前面有 Nginx，可以這樣設定：

```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

這樣使用者第二次訪問登入頁面時，靜態資源會從 browser cache 讀取，不用再跟 server 拿。

## Token 設定調整

Token 的簽章驗證也會影響效能。可以調整一些參數：

在 Realm settings → Keys，選擇使用的演算法：

- **RS256**：預設，RSA 簽章，較慢但相容性好
- **ES256**：ECDSA 簽章，較快，但需要較新的 library 支援

如果確定所有 client 都支援，可以改用 ES256：

1. 在 Keys 頁面加入一個 ES256 的 provider
2. 設定為 Active
3. 把 RS256 設為 Passive（保留一段時間做相容）

測試後 Token 驗證快了約 15-20%。

## 水平擴展

單機優化到一個程度後，繼續提升就要靠水平擴展了。

架設多台 Keycloak，前面放 Load Balancer：

```
                    ┌──> Keycloak Node 1
Load Balancer ────┼──> Keycloak Node 2
                    └──> Keycloak Node 3
                            │
                            ▼
                      PostgreSQL
```

Session 要在節點間同步，確保使用者被導到不同節點時還是登入狀態。

Keycloak 用 Infinispan 的 distributed cache 來同步 session。在 cluster 模式下，要設定節點間的通訊：

```xml
<jgroups>
    <stack name="tcp">
        <transport type="TCP" socket-binding="jgroups-tcp"/>
        <protocol type="TCPPING">
            <property name="initial_hosts">
                192.168.1.101[7600],192.168.1.102[7600],192.168.1.103[7600]
            </property>
        </protocol>
    </stack>
</jgroups>
```

## 效能監控

調校完要持續監控，確保效能穩定。用 Prometheus + Grafana 監控關鍵指標：

- **回應時間**：p50、p90、p95、p99
- **吞吐量**：requests per second
- **錯誤率**：4xx、5xx 錯誤比例
- **JVM metrics**：Heap 使用量、GC 頻率
- **Database metrics**：連線數、慢查詢
- **Cache 命中率**：越高越好

設定告警規則：

```yaml
- alert: KeycloakSlowResponse
  expr: histogram_quantile(0.95, keycloak_response_time_seconds) > 1
  for: 5m
  annotations:
    summary: "Keycloak 95th percentile response time > 1s"
```

## 最終測試結果

經過一輪調校後，再跑一次壓力測試：

```
平均回應時間: 320ms (從 850ms 降低 62%)
90th percentile: 520ms
95th percentile: 650ms
吞吐量: 120 requests/sec (從 45 提升 167%)
錯誤率: 0%
```

效果相當不錯！不過這些數字還是要看實際場景，不同的使用模式會有不同的瓶頸。

## 一些建議

效能調校要根據實際監控數據來做，不要盲目調整。每次改一個參數，測試看效果，再決定要不要保留。

優先處理影響最大的部分。通常 JVM 記憶體和資料庫連線池的調整效果最明顯。

不要為了效能犧牲太多安全性。像是密碼 hash iteration、Token 過期時間這些，還是要保持在合理的安全範圍。

最後，硬體升級有時候比軟體調校更直接。如果預算允許，加記憶體、換 SSD、用更快的 CPU 都能立即見效。

下週想研究 Keycloak 的高可用部署，看怎麼設定 Load Balancer、處理 node 故障、做 rolling update 等等。這些對正式環境的穩定性很重要。
