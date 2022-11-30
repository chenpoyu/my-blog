---
layout: post
title: "ELK Stack 集中式日誌管理"
date: 2019-01-28 14:30:00 +0800
categories: [DevOps, Logging]
tags: [ELK, Elasticsearch, Logstash, Kibana, Fluentd, Logging]
---

上週建立了 Prometheus 監控系統（參考 [Prometheus 和 Grafana 監控 Kubernetes](/posts/2019/01/21/prometheus-grafana-monitoring/)），可以看到系統的指標。但遇到問題時，還是要看日誌才能知道發生了什麼事。

現在的日誌管理方式很原始：
```bash
kubectl logs my-shop-api-xxxxx-xxxxx
```

問題：
1. **分散在各個 Pod**：有 10 個 Pod 就要查 10 次
2. **Pod 重啟後日誌消失**：無法回溯歷史問題
3. **無法搜尋**：想找「某個使用者的所有操作」很困難
4. **沒有統計分析**：無法看趨勢

這週建立 ELK Stack（Elasticsearch + Logstash + Kibana），集中收集、儲存、搜尋所有日誌。

> 使用版本：Elasticsearch 6.5、Kibana 6.5、Fluentd 1.3

## ELK Stack 是什麼

ELK 是三個開源專案的縮寫：

### Elasticsearch
- 分散式搜尋和分析引擎
- 儲存和索引日誌
- 提供強大的搜尋功能

### Logstash
- 日誌收集和處理管道
- 從多個來源收集日誌
- 轉換格式後送到 Elasticsearch

### Kibana
- 資料視覺化平台
- 搜尋和分析日誌
- 建立儀表板

### 架構

```
[應用程式] → 輸出日誌到 stdout
     ↓
[Fluentd] → 收集容器日誌（我們用 Fluentd 取代 Logstash）
     ↓
[Elasticsearch] → 儲存和索引
     ↓
[Kibana] → 視覺化查詢
     ↓
[使用者] → 搜尋日誌
```

為什麼用 Fluentd 而不是 Logstash？
- Fluentd 更輕量（記憶體消耗少）
- 更適合容器環境
- CNCF 專案，K8s 生態系統支援更好

## 安裝 Elasticsearch

### 使用 Helm 安裝

```bash
# 加入 Elastic Helm repo
helm repo add elastic https://helm.elastic.co

# 安裝 Elasticsearch
helm install elastic/elasticsearch \
  --name elasticsearch \
  --namespace logging \
  --set replicas=1 \
  --set minimumMasterNodes=1 \
  --set resources.requests.memory=1Gi \
  --set resources.limits.memory=2Gi
```

單節點設定適合測試，生產環境建議 3 節點以上。

查看狀態：
```bash
kubectl get pods -n logging
# NAME                     READY   STATUS    RESTARTS   AGE
# elasticsearch-master-0   1/1     Running   0          2m
```

等待 Pod 變成 Running。

### 驗證 Elasticsearch

```bash
kubectl port-forward -n logging svc/elasticsearch-master 9200:9200
```

測試：
```bash
curl http://localhost:9200

# {
#   "name" : "elasticsearch-master-0",
#   "cluster_name" : "elasticsearch",
#   "version" : {
#     "number" : "6.5.4",
#     ...
#   }
# }
```

### 檢查叢集健康

```bash
curl http://localhost:9200/_cluster/health?pretty

# {
#   "cluster_name" : "elasticsearch",
#   "status" : "green",
#   "number_of_nodes" : 1,
#   ...
# }
```

`status: green` 表示一切正常。

## 安裝 Kibana

```bash
helm install elastic/kibana \
  --name kibana \
  --namespace logging \
  --set elasticsearchHosts=http://elasticsearch-master:9200
```

查看狀態：
```bash
kubectl get pods -n logging | grep kibana
# kibana-xxxxx   1/1     Running   0          1m
```

### 訪問 Kibana

```bash
kubectl port-forward -n logging svc/kibana-kibana 5601:5601
```

打開瀏覽器：`http://localhost:5601`

第一次訪問會看到歡迎頁面。

## 安裝 Fluentd

Fluentd 作為 DaemonSet 運行在每個 Node 上，收集所有容器的日誌。

### 建立 ConfigMap

`fluentd-config.yaml`：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    # 收集容器日誌
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
    
    # 過濾和解析 Kubernetes metadata
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
    </filter>
    
    # 輸出到 Elasticsearch
    <match **>
      @type elasticsearch
      @id out_es
      @log_level info
      include_tag_key true
      host elasticsearch-master
      port 9200
      logstash_format true
      logstash_prefix fluentd
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>
```

部署：
```bash
kubectl apply -f fluentd-config.yaml
```

### 建立 DaemonSet

`fluentd-daemonset.yaml`：
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.3-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch-master"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc/fluent.conf
          subPath: fluent.conf
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config
```

### 建立 ServiceAccount 和 RBAC

`fluentd-rbac.yaml`：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: logging
```

部署：
```bash
kubectl apply -f fluentd-rbac.yaml
kubectl apply -f fluentd-daemonset.yaml
```

查看狀態：
```bash
kubectl get pods -n logging | grep fluentd
# fluentd-xxxxx   1/1     Running   0          1m
# fluentd-yyyyy   1/1     Running   0          1m
```

每個 Node 都會有一個 Fluentd Pod。

## 在 Kibana 中查詢日誌

### 建立 Index Pattern

1. 打開 Kibana（`http://localhost:5601`）
2. 點「Management」→「Index Patterns」
3. 點「Create index pattern」
4. Index pattern：`fluentd-*`
5. Time Filter field name：選「@timestamp」
6. 點「Create index pattern」

### 查看日誌

1. 點「Discover」
2. 選擇時間範圍（右上角）
3. 就能看到所有容器的日誌了！

日誌欄位：
- `log`：日誌內容
- `kubernetes.pod_name`：Pod 名稱
- `kubernetes.namespace_name`：Namespace
- `kubernetes.container_name`：容器名稱
- `stream`：stdout 或 stderr

### 搜尋日誌

**搜尋特定 Pod 的日誌：**
```
kubernetes.pod_name: my-shop-api-*
```

**搜尋錯誤日誌：**
```
log: *ERROR* OR log: *Exception*
```

**搜尋特定 namespace：**
```
kubernetes.namespace_name: default
```

**組合條件：**
```
kubernetes.namespace_name: default AND log: *ERROR*
```

### 過濾欄位

左側可以選擇要顯示的欄位：
- 點 `kubernetes.pod_name` 旁的「add」
- 點 `log` 旁的「add」

這樣就只顯示這兩個欄位，更清楚。

## 結構化日誌

預設情況下，應用程式的日誌都在 `log` 欄位中，是一大串文字。

更好的做法是輸出 JSON 格式的日誌，Fluentd 會自動解析成欄位。

### Spring Boot 輸出 JSON 日誌

加入依賴：

`pom.xml`：
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.2</version>
</dependency>
```

設定 Logback：

`src/main/resources/logback-spring.xml`：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"my-shop-api"}</customFields>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

現在日誌會輸出成 JSON：
```json
{
  "@timestamp": "2019-01-28T14:30:00.123+08:00",
  "message": "User login successful",
  "logger_name": "com.mycompany.myshop.UserService",
  "level": "INFO",
  "thread_name": "http-nio-8080-exec-1",
  "app": "my-shop-api"
}
```

在 Kibana 中，這些會自動變成獨立欄位，可以直接搜尋：
```
level: ERROR
logger_name: com.mycompany.myshop.UserService
```

### 加入自訂欄位

在程式中使用 MDC（Mapped Diagnostic Context）：

```java
import org.slf4j.MDC;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@RestController
public class UserController {
    private static final Logger log = LoggerFactory.getLogger(UserController.class);
    
    @GetMapping("/api/users/{id}")
    public User getUser(@PathVariable Long id) {
        // 加入 user_id 到 MDC
        MDC.put("user_id", String.valueOf(id));
        
        try {
            log.info("Fetching user");
            User user = userService.getUser(id);
            log.info("User found");
            return user;
        } finally {
            // 清除 MDC
            MDC.clear();
        }
    }
}
```

日誌會包含 `user_id` 欄位：
```json
{
  "message": "Fetching user",
  "user_id": "123",
  ...
}
```

這樣就能搜尋：
```
user_id: 123
```

查看某個使用者的所有操作！

## Kibana 進階功能

### 儲存搜尋

常用的搜尋可以儲存起來：
1. 輸入搜尋條件
2. 點「Save」
3. 輸入名稱：「Production Errors」

下次直接選擇這個儲存的搜尋。

### 建立視覺化

1. 點「Visualize」→「Create a visualization」
2. 選擇類型（Line、Bar、Pie 等）
3. 選擇 index pattern：`fluentd-*`

**範例：錯誤日誌數量趨勢**
- Visualization: Line
- Y-axis: Count
- X-axis: Date Histogram (@timestamp)
- Filter: `level: ERROR`

**範例：各服務的日誌數量**
- Visualization: Pie Chart
- Slice size: Count
- Split slices: Terms → `kubernetes.container_name.keyword`

### 建立儀表板

1. 點「Dashboard」→「Create new dashboard」
2. 點「Add」，選擇剛才建立的視覺化
3. 排列位置和大小
4. 點「Save」

推薦的儀表板內容：
- 總日誌數量（時間趨勢）
- 各 Pod 的日誌數量
- 錯誤日誌數量趨勢
- 最近的錯誤日誌（表格）
- 各 HTTP status code 數量

## 告警

Kibana 的 Watcher 功能可以設定告警（需要 X-Pack，付費功能）。

免費替代方案：使用 ElastAlert。

### 安裝 ElastAlert

`elastalert-deployment.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elastalert
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elastalert
  template:
    metadata:
      labels:
        app: elastalert
    spec:
      containers:
      - name: elastalert
        image: jertel/elastalert-docker:0.2.1
        volumeMounts:
        - name: config
          mountPath: /opt/config
        - name: rules
          mountPath: /opt/rules
      volumes:
      - name: config
        configMap:
          name: elastalert-config
      - name: rules
        configMap:
          name: elastalert-rules
```

### 設定告警規則

`elastalert-rule.yaml`（ConfigMap）：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elastalert-rules
  namespace: logging
data:
  error_spike.yaml: |
    name: Error Spike
    type: frequency
    index: fluentd-*
    num_events: 50
    timeframe:
      minutes: 5
    filter:
    - term:
        level: "ERROR"
    alert:
    - "slack"
    slack_webhook_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    slack_channel_override: "#alerts"
    slack_username_override: "ElastAlert"
```

這個規則會在 5 分鐘內出現 50 個 ERROR 日誌時通知 Slack。

## 日誌保留政策

Elasticsearch 會持續累積日誌，最終會塞爆硬碟。

### 使用 Curator 自動清理

Curator 是 Elasticsearch 的索引管理工具。

安裝：
```bash
helm install stable/elasticsearch-curator \
  --name curator \
  --namespace logging \
  --set config.elasticsearch.hosts={elasticsearch-master} \
  --set configMaps.action_file.yml="actions:\n  1:\n    action: delete_indices\n    description: Delete old indices\n    options:\n      ignore_empty_list: True\n    filters:\n    - filtertype: pattern\n      kind: prefix\n      value: fluentd-\n    - filtertype: age\n      source: name\n      direction: older\n      timestring: '%Y.%m.%d'\n      unit: days\n      unit_count: 7"
```

這會刪除 7 天前的日誌。

或使用 CronJob：

`curator-cronjob.yaml`：
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: curator
  namespace: logging
spec:
  schedule: "0 1 * * *"  # 每天凌晨 1 點
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: curator
            image: untergeek/curator:5.6
            args:
            - --config
            - /etc/config/config.yml
            - /etc/config/action.yml
            volumeMounts:
            - name: config
              mountPath: /etc/config
          restartPolicy: OnFailure
          volumes:
          - name: config
            configMap:
              name: curator-config
```

## 遇到的問題

### 問題一：Elasticsearch 啟動失敗

錯誤：`max virtual memory areas vm.max_map_count [65530] is too low`

解決方法（在 Node 上執行）：
```bash
sudo sysctl -w vm.max_map_count=262144
```

永久設定：
```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

Minikube：
```bash
minikube ssh
sudo sysctl -w vm.max_map_count=262144
exit
```

### 問題二：Fluentd 沒有收集到日誌

檢查 Fluentd 日誌：
```bash
kubectl logs -n logging fluentd-xxxxx
```

常見原因：
1. hostPath 路徑不對（不同環境可能不同）
2. 權限問題（RBAC 設定）
3. Elasticsearch 連不上

### 問題三：Kibana 顯示「No results found」

可能原因：
1. 時間範圍太窄，調整為「Last 24 hours」
2. Index pattern 不正確
3. 日誌還沒有傳到 Elasticsearch（等幾分鐘）

檢查 Elasticsearch 是否有資料：
```bash
curl http://localhost:9200/fluentd-*/_search?pretty
```

## 心得

有了 ELK Stack 後，除錯效率大幅提升。以前要登入每個 Pod 看日誌，現在直接在 Kibana 搜尋就好。

上週遇到一個奇怪的 bug，使用者回報「偶爾會看到別人的訂單」。這種問題以前根本查不出來，因為不知道什麼時候發生、在哪個 Pod。現在直接在 Kibana 搜尋：
```
user_id: 123 AND message: *order* AND order_id: 456
```

馬上找到問題日誌，發現是 Session 處理有 bug。

而且 JSON 格式的日誌真的好用很多，以前日誌都是純文字，要用正則表達式解析。現在每個欄位都清清楚楚，搜尋和統計都很方便。

下週要研究 CI/CD Pipeline 的進階應用，整合自動化測試、程式碼掃描、容器安全掃描等。
