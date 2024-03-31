---
layout: post
title: "Keycloak 高可用架構部署"
date: 2024-04-01 15:30:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, High Availability, Clustering]
---

這週研究了 Keycloak 的高可用部署方案。單機部署有個風險：Keycloak 掛了，所有系統都不能登入。這對關鍵業務來說是不能接受的。

高可用架構的目標是：任何一台 Keycloak 掛掉，服務還是能正常運作。這週就來看看怎麼做。

## 高可用架構設計

基本的 HA 架構是這樣：

```
                        Load Balancer
                              │
                 ┌────────────┼────────────┐
                 │            │            │
          Keycloak 1    Keycloak 2    Keycloak 3
                 │            │            │
                 └────────────┼────────────┘
                              │
                      PostgreSQL HA
                      (Primary + Standby)
```

關鍵元件：

1. **多台 Keycloak**：至少 2 台，建議 3 台
2. **Load Balancer**：分散流量，健康檢查
3. **共享資料庫**：所有 Keycloak 連同一個資料庫
4. **Cluster 通訊**：Keycloak 之間同步 cache 和 session

## 資料庫高可用

Keycloak 本身是 stateless 的（狀態在 cache），但資料庫是 single point of failure。要先把資料庫做成 HA。

PostgreSQL 可以用 streaming replication：

**Primary 節點**：接受寫入和讀取

**Standby 節點**：持續從 Primary 同步資料，只接受讀取

設定 `postgresql.conf`：

```ini
# Primary
wal_level = replica
max_wal_senders = 3
wal_keep_size = 128MB

# Standby
hot_standby = on
```

用 PgPool-II 或 Patroni 來做自動 failover。當 Primary 掛掉，自動把 Standby 提升為 Primary。

或者直接用雲端服務的 managed database（AWS RDS、Azure Database、Google Cloud SQL），它們內建 HA 機制。

## Keycloak Cluster 設定

多台 Keycloak 要組成 cluster，讓 cache 和 session 能同步。

在每台 Keycloak 上設定 `conf/keycloak.conf`：

```properties
# Cache 設定
cache=ispn
cache-stack=tcp

# Cluster 名稱
cache-config-file=cache-ispn.xml
```

建立 `conf/cache-ispn.xml`：

```xml
<infinispan>
    <jgroups>
        <stack name="tcp">
            <transport type="TCP" socket-binding="jgroups-tcp"/>
            <!-- 列出所有節點 -->
            <protocol type="TCPPING">
                <property name="initial_hosts">
                    10.0.1.10[7600],10.0.1.11[7600],10.0.1.12[7600]
                </property>
                <property name="port_range">0</property>
            </protocol>
            <protocol type="MERGE3"/>
            <protocol type="FD_SOCK"/>
            <protocol type="FD_ALL"/>
            <protocol type="VERIFY_SUSPECT"/>
            <protocol type="pbcast.NAKACK2"/>
            <protocol type="UNICAST3"/>
            <protocol type="pbcast.STABLE"/>
            <protocol type="pbcast.GMS"/>
            <protocol type="MFC"/>
            <protocol type="FRAG3"/>
        </stack>
    </jgroups>
    
    <cache-container name="keycloak">
        <!-- Session cache - 分散式 -->
        <distributed-cache name="sessions" owners="2">
            <expiration lifespan="-1"/>
        </distributed-cache>
        
        <!-- Client session cache -->
        <distributed-cache name="clientSessions" owners="2">
            <expiration lifespan="-1"/>
        </distributed-cache>
        
        <!-- Offline session cache -->
        <distributed-cache name="offlineSessions" owners="2">
            <expiration lifespan="-1"/>
        </distributed-cache>
    </cache-container>
</infinispan>
```

`owners="2"` 表示每個 session 會複製到 2 個節點。這樣任一節點掛掉，session 不會遺失。

啟動後檢查 cluster 狀態：

```bash
# 在任一節點執行
bin/kcadm.sh config credentials --server http://localhost:8080 \
  --realm master --user admin
bin/kcadm.sh get serverinfo
```

在 `systeminfo` 下應該能看到所有節點的資訊。

## Load Balancer 設定

用 HAProxy 當 Load Balancer：

```
global
    maxconn 4096

defaults
    mode http
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend keycloak_frontend
    bind *:443 ssl crt /etc/ssl/certs/keycloak.pem
    default_backend keycloak_backend

backend keycloak_backend
    balance roundrobin
    option httpchk GET /health/ready
    http-check expect status 200
    
    server keycloak1 10.0.1.10:8080 check inter 5s rise 2 fall 3
    server keycloak2 10.0.1.11:8080 check inter 5s rise 2 fall 3
    server keycloak3 10.0.1.12:8080 check inter 5s rise 2 fall 3
```

關鍵設定：

- **balance roundrobin**：輪流分配請求
- **option httpchk**：健康檢查，呼叫 `/health/ready` endpoint
- **check inter 5s**：每 5 秒檢查一次
- **rise 2 fall 3**：連續 2 次成功視為 up，連續 3 次失敗視為 down

Keycloak 23+ 有內建的 health endpoint：

- `/health/ready`：是否準備好接受流量
- `/health/live`：是否還活著（用於 Kubernetes liveness probe）

## Session Stickiness

要不要設定 session sticky（同一使用者的請求都導到同一台 Keycloak）？

**不用 sticky**：
- 優點：流量分配更均勻，某台掛掉影響最小
- 缺點：cache 命中率較低，跨節點讀取 session 有延遲

**用 sticky**：
- 優點：cache 命中率高，效能較好
- 缺點：流量可能不均，某台掛掉會影響該台的所有使用者

Keycloak 官方建議不用 sticky，因為 session 已經有 replication，跨節點存取的效能影響不大。

如果真的要用，可以用 cookie-based sticky：

```
backend keycloak_backend
    balance roundrobin
    cookie SERVERID insert indirect nocache
    server keycloak1 10.0.1.10:8080 check cookie kc1
    server keycloak2 10.0.1.11:8080 check cookie kc2
```

## 故障測試

設定好後要測試故障情境。

**測試 1：關閉一台 Keycloak**

```bash
# 在 keycloak2 上
systemctl stop keycloak
```

觀察：
- Load Balancer 檢測到 keycloak2 down，停止分配流量給它
- 使用者原本在 keycloak2 的 session 會從 keycloak1 或 keycloak3 讀取（因為有 replication）
- 使用者沒有感覺，繼續正常使用

啟動 keycloak2 後，它會重新加入 cluster，開始接收流量。

**測試 2：網路分區**

模擬網路分區（split brain）：

```bash
# 在 keycloak2 上，封鎖其他節點
iptables -A INPUT -s 10.0.1.10 -j DROP
iptables -A INPUT -s 10.0.1.12 -j DROP
```

觀察：
- keycloak2 認為自己被孤立，但還在運作
- keycloak1 和 keycloak3 認為 keycloak2 掛了
- Load Balancer 會停止分配流量給 keycloak2

這種情況要注意，恢復網路後要確保 cluster 正確 merge。

## Rolling Update

正式環境要更新 Keycloak 版本，不能全部同時停機。要做 rolling update：

1. 從 Load Balancer 移除 keycloak3
2. 等待現有請求處理完（grace period）
3. 停止 keycloak3，更新版本，重啟
4. 確認 keycloak3 正常後，加回 Load Balancer
5. 重複步驟 1-4 處理 keycloak2 和 keycloak1

這樣整個過程服務不中斷。

HAProxy 可以用 admin socket 動態調整：

```bash
# 設定 keycloak3 為 drain 模式（不接受新請求）
echo "set server keycloak_backend/keycloak3 state drain" | \
  socat stdio /var/run/haproxy.sock

# 等待現有請求完成（觀察 active connections）
watch 'echo "show stat" | socat stdio /var/run/haproxy.sock | grep keycloak3'

# 更新完成後恢復
echo "set server keycloak_backend/keycloak3 state ready" | \
  socat stdio /var/run/haproxy.sock
```

## 跨機房部署

如果要做跨機房的 disaster recovery，架構會更複雜：

```
機房 A                        機房 B
┌─────────────────┐          ┌─────────────────┐
│  Load Balancer  │          │  Load Balancer  │
│  Keycloak 1-3   │          │  Keycloak 4-6   │
│  PostgreSQL     │          │  PostgreSQL     │
│  (Primary)      │◄────────►│  (Standby)      │
└─────────────────┘  Repl    └─────────────────┘
```

通常不建議跨機房做 Keycloak cluster（網路延遲太高），而是：

1. 每個機房獨立的 Keycloak cluster
2. 資料庫用跨機房 replication（非同步）
3. DNS 做 geo-routing 或 failover

這樣平時各機房獨立運作，某個機房掛了才切到另一個。

## 監控告警

HA 環境一定要有完整的監控：

**Keycloak 層級**：
- 各節點是否正常
- Cluster 成員數量
- Session replication 延遲

**Load Balancer 層級**：
- 健康檢查狀態
- 流量分配是否均勻
- Backend 回應時間

**資料庫層級**：
- Replication lag
- Connection pool 使用率
- 慢查詢

設定告警規則，任何異常立刻通知。

## 實務心得

HA 部署不難，但要注意細節。最常見的問題是 cluster 通訊：

- 防火牆要開 7600 port（JGroups 預設）
- 確保所有節點能互相連線
- 時間要同步（用 NTP），不然 token 驗證會有問題

另外要做好變更管理。更新設定或版本前，先在測試環境完整測試，包括故障情境。

最後是成本考量。3 台 Keycloak + HA 資料庫，成本是單機的好幾倍。要評估是否真的需要這麼高的可用性，或許 2 台 + 快速恢復機制就夠了。

經過這幾個月的研究，Keycloak 的主要功能都試過了。接下來應該會開始實際應用到一些專案上，看看實戰會遇到什麼新問題。
