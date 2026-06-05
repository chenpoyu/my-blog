---
layout: post
title: "混沌工程入門"
date: 2019-04-29 10:00:00 +0800
categories: [DevOps, Reliability]
tags: [Testing, DevOps, Reliability]
---

上週研究了服務網格流量管理（參考 [服務網格流量管理進階](/posts/2019/04/22/service-mesh-traffic-management/)），這週來學習一個聽起來很瘋狂的概念：混沌工程 (Chaos Engineering)。

什麼是混沌工程？**主動在生產環境製造故障，測試系統的韌性**。

聽起來很危險？一開始我也這麼覺得。但去年有次經驗改變了我的想法：

凌晨 3 點，資料庫主機掛了。我們的系統完全無法使用，因為沒有人知道 failover 機制是否真的能運作（從來沒測試過）。折騰了 2 小時才恢復服務，損失慘重。

如果我們平時就定期測試 failover，就不會措手不及了。

這就是混沌工程的核心概念：**與其等故障發生，不如主動製造故障，確保系統能承受**。

> 相關工具：Chaos Monkey、Gremlin、Chaos Toolkit、Litmus Chaos

## 混沌工程是什麼

混沌工程是在分散式系統上進行實驗的學科，目的是建立系統在混亂條件下的信心。

核心原則：
1. **建立穩態假設**：定義系統正常運作的指標
2. **設計實驗**：設計會破壞系統的實驗
3. **最小化爆炸半徑**：從小範圍開始
4. **自動化實驗**：持續執行

Netflix 的 Chaos Monkey 最有名，隨機關閉生產環境的 instances，確保系統能承受單點故障。

## 為什麼需要混沌工程

分散式系統有很多潛在故障：
- 伺服器當機
- 網路延遲
- 磁碟滿了
- 資料庫連線池耗盡
- 依賴服務掛了
- 記憶體洩漏
- CPU 滿載

這些問題在生產環境一定會發生，問題是：**你的系統準備好了嗎？**

混沌工程讓你：
1. 發現系統的弱點
2. 驗證容錯機制真的有效
3. 培養團隊處理故障的能力
4. 建立對系統的信心

## Chaos Monkey

Netflix 開源的混沌工程工具，隨機關閉 instances。

### Simian Army

Netflix 的混沌工程工具集：
- **Chaos Monkey**：隨機終止 instances
- **Chaos Gorilla**：模擬整個 Availability Zone 故障
- **Latency Monkey**：注入網路延遲
- **Conformity Monkey**：關閉不符合最佳實踐的 instances
- **Security Monkey**：檢查安全配置
- **Janitor Monkey**：清理未使用的資源

### 使用 Chaos Monkey

安裝：
```bash
# 使用 Spinnaker 整合 Chaos Monkey
# 或直接使用 Spring Boot Chaos Monkey
```

Spring Boot 整合：

`pom.xml`：
```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>chaos-monkey-spring-boot</artifactId>
    <version>2.0.2</version>
</dependency>
```

`application.yml`：
```yaml
chaos:
  monkey:
    enabled: true
    assaults:
      level: 5                    # 1-10，越高越猛
      latencyActive: true         # 啟用延遲攻擊
      latencyRangeStart: 1000     # 延遲 1-5 秒
      latencyRangeEnd: 5000
      exceptionsActive: true      # 啟用例外攻擊
      killApplicationActive: false  # 不要真的關閉應用
    watcher:
      controller: true            # 監控 Controller
      restController: true
      service: true               # 監控 Service
      repository: true            # 監控 Repository
```

Chaos Monkey 會隨機：
- 延遲方法執行
- 拋出例外
- 讓方法回傳 null

測試你的錯誤處理是否健全。

啟動應用：
```bash
java -jar myapp.jar --chaos.monkey.enabled=true
```

使用 API 控制：
```bash
# 啟用 Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/enable

# 查看狀態
curl http://localhost:8080/actuator/chaosmonkey/status

# 變更設定
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults \
  -H "Content-Type: application/json" \
  -d '{"level":8,"latencyActive":true}'
```

## Kubernetes 混沌工程

### Litmus Chaos

CNCF 專案，專門為 Kubernetes 設計。

安裝：
```bash
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v1.0.0.yaml
```

建立 Chaos Experiment：

`pod-delete.yaml`：
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: order-service-chaos
  namespace: default
spec:
  appinfo:
    appns: default
    applabel: "app=order-service"
    appkind: deployment
  chaosServiceAccount: litmus
  monitoring: true
  jobCleanUpPolicy: retain
  experiments:
  - name: pod-delete
    spec:
      components:
        env:
        - name: TOTAL_CHAOS_DURATION
          value: "60"
        - name: CHAOS_INTERVAL
          value: "10"
        - name: FORCE
          value: "false"
```

執行：
```bash
kubectl apply -f pod-delete.yaml
```

Litmus 會：
1. 每 10 秒刪除一個 order-service Pod
2. 持續 60 秒
3. 觀察系統是否還能正常運作

### 常見的 Chaos Experiments

#### 1. Pod Deletion

隨機刪除 Pod：
```yaml
- name: pod-delete
  spec:
    components:
      env:
      - name: TOTAL_CHAOS_DURATION
        value: "300"
      - name: CHAOS_INTERVAL
        value: "30"
```

驗證：
- Kubernetes 是否自動重啟 Pod
- 服務是否持續可用
- 負載平衡是否正常

#### 2. Network Latency

注入網路延遲：
```yaml
- name: pod-network-latency
  spec:
    components:
      env:
      - name: NETWORK_INTERFACE
        value: "eth0"
      - name: NETWORK_LATENCY
        value: "2000"  # 2 秒延遲
      - name: TOTAL_CHAOS_DURATION
        value: "120"
```

驗證：
- 超時設定是否合理
- 重試機制是否有效
- 使用者體驗是否可接受

#### 3. Network Loss

模擬網路封包丟失：
```yaml
- name: pod-network-loss
  spec:
    components:
      env:
      - name: NETWORK_PACKET_LOSS_PERCENTAGE
        value: "50"  # 50% 封包丟失
      - name: TOTAL_CHAOS_DURATION
        value: "120"
```

#### 4. CPU Stress

CPU 負載壓力：
```yaml
- name: pod-cpu-hog
  spec:
    components:
      env:
      - name: CPU_CORES
        value: "2"
      - name: TOTAL_CHAOS_DURATION
        value: "180"
```

驗證：
- HPA 是否自動擴展
- 效能降級是否優雅
- 監控告警是否觸發

#### 5. Memory Stress

記憶體壓力：
```yaml
- name: pod-memory-hog
  spec:
    components:
      env:
      - name: MEMORY_CONSUMPTION
        value: "500"  # 500 MB
      - name: TOTAL_CHAOS_DURATION
        value: "180"
```

#### 6. Disk Fill

填滿磁碟：
```yaml
- name: disk-fill
  spec:
    components:
      env:
      - name: FILL_PERCENTAGE
        value: "80"  # 填滿到 80%
      - name: TOTAL_CHAOS_DURATION
        value: "300"
```

驗證：
- 應用是否正確處理磁碟滿的錯誤
- 日誌輪轉是否正常
- 監控是否告警

## 實驗設計

### 步驟 1：定義穩態

確定系統正常運作的指標：

```yaml
穩態假設：
- API 成功率 > 99.9%
- P95 回應時間 < 200ms
- 錯誤率 < 0.1%
- 所有 Pod 處於 Running 狀態
```

使用 Prometheus 監控：
```promql
# 成功率
sum(rate(http_requests_total{status=~"2.."}[5m])) / 
sum(rate(http_requests_total[5m])) * 100

# P95 延遲
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m]))
```

### 步驟 2：設計實驗

**實驗**：隨機刪除 20% 的 order-service Pods

**假設**：系統仍能維持穩態（成功率 > 99.9%）

**步驟**：
1. 監控穩態指標（實驗前）
2. 執行混沌實驗（刪除 Pods）
3. 監控穩態指標（實驗中）
4. 停止實驗
5. 監控穩態指標（實驗後）

### 步驟 3：最小化爆炸半徑

**不要**一開始就在生產環境搞：

階段 1：本地環境
- Docker Compose
- Minikube

階段 2：開發環境
- Kubernetes 開發叢集
- 非關鍵服務

階段 3：Staging 環境
- 模擬生產環境
- 完整測試

階段 4：生產環境（Canary）
- 特定區域
- 低流量時段
- 小比例 Pods

階段 5：生產環境（全面）
- 所有區域
- 自動化執行

### 步驟 4：執行實驗

```bash
# 記錄開始時間
START_TIME=$(date +%s)

# 記錄穩態指標（實驗前）
echo "=== Before Chaos ==="
kubectl get pods -l app=order-service
curl http://metrics-api/success-rate
curl http://metrics-api/latency

# 執行混沌實驗
kubectl apply -f pod-delete-chaos.yaml

# 監控指標（每 10 秒）
while true; do
  echo "=== During Chaos ($(date)) ==="
  kubectl get pods -l app=order-service
  curl http://metrics-api/success-rate
  curl http://metrics-api/latency
  sleep 10
done

# 停止實驗（60 秒後）
sleep 60
kubectl delete -f pod-delete-chaos.yaml

# 記錄穩態指標（實驗後）
echo "=== After Chaos ==="
kubectl get pods -l app=order-service
curl http://metrics-api/success-rate
curl http://metrics-api/latency

# 計算實驗時間
END_TIME=$(date +%s)
echo "Experiment duration: $((END_TIME - START_TIME)) seconds"
```

### 步驟 5：分析結果

**成功案例**：
```
Before Chaos:
- Pods: 5/5 Running
- Success Rate: 99.95%
- P95 Latency: 150ms

During Chaos:
- Pods: 3/5 Running (2 被刪除，正在重啟)
- Success Rate: 99.92%  ← 稍微下降但在目標內
- P95 Latency: 180ms    ← 稍微增加但可接受

After Chaos:
- Pods: 5/5 Running (已恢復)
- Success Rate: 99.96%
- P95 Latency: 145ms

結論：系統能承受 Pod 故障，自動恢復正常
```

**失敗案例**：
```
During Chaos:
- Pods: 3/5 Running
- Success Rate: 85%     ← 💥 大幅下降！
- P95 Latency: 5000ms   ← 💥 超時！

問題：剩餘 3 個 Pods 無法處理全部流量

Action Items:
1. 增加 minReplicas（5 → 10）
2. 調整 HPA 更快速反應
3. 增加 Pod 資源限制
```

## 自動化混沌工程

### 定期執行

使用 CronJob 定期執行混沌實驗：

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: chaos-experiment
spec:
  schedule: "0 2 * * 1,3,5"  # 每週一、三、五 2:00 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: chaos
            image: litmuschaos/chaos-executor:latest
            args:
            - --experiment=pod-delete
            - --target=order-service
            - --duration=300
          restartPolicy: OnFailure
```

### CI/CD 整合

在部署前執行混沌測試：

`.gitlab-ci.yml`：
```yaml
stages:
  - build
  - test
  - chaos-test
  - deploy

chaos_test:
  stage: chaos-test
  script:
    - kubectl apply -f chaos-experiments/
    - sleep 300
    - kubectl delete -f chaos-experiments/
    - python verify_metrics.py  # 驗證指標是否符合預期
  only:
    - master
  when: manual  # 手動觸發
```

`verify_metrics.py`：
```python
import requests

def verify_success_rate():
    response = requests.get('http://prometheus/api/v1/query',
        params={'query': 'sum(rate(http_requests_total{status=~"2.."}[5m])) / sum(rate(http_requests_total[5m])) * 100'})
    success_rate = float(response.json()['data']['result'][0]['value'][1])
    
    assert success_rate > 99.9, f"Success rate {success_rate}% is below threshold"
    print(f"Success rate: {success_rate}%")

def verify_latency():
    response = requests.get('http://prometheus/api/v1/query',
        params={'query': 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))'})
    p95_latency = float(response.json()['data']['result'][0]['value'][1])
    
    assert p95_latency < 0.2, f"P95 latency {p95_latency}s is above threshold"
    print(f"P95 latency: {p95_latency}s")

if __name__ == '__main__':
    verify_success_rate()
    verify_latency()
    print("All chaos tests passed!")
```

## 實際案例

### 案例一：發現單點故障

**實驗**：刪除 Redis Pod

**預期**：應用使用降級機制，繼續運作

**實際**：整個應用當機

**原因**：程式碼沒有處理 Redis 連線失敗

**修正**：
```java
@Cacheable(value = "users", unless = "#result == null")
public User getUser(String id) {
    try {
        // 嘗試從 Redis 取得
        return redisTemplate.opsForValue().get("user:" + id);
    } catch (Exception e) {
        logger.warn("Redis unavailable, fallback to database", e);
        // 降級：直接查詢資料庫
        return userRepository.findById(id).orElse(null);
    }
}
```

### 案例二：資料庫 Failover

**實驗**：關閉資料庫主節點

**預期**：自動 failover 到從節點

**實際**：failover 成功，但應用連線池還是連到舊主節點，失敗

**原因**：連線池沒有檢測 failover

**修正**：使用資料庫代理（ProxySQL）或設定連線池自動重連：
```yaml
spring:
  datasource:
    hikari:
      connection-test-query: SELECT 1
      validation-timeout: 3000
      max-lifetime: 300000  # 5 分鐘後更新連線
```

### 案例三：級聯故障

**實驗**：注入 payment-service 延遲（5 秒）

**預期**：只有 payment-service 變慢

**實際**：所有服務都變慢，整個系統幾乎癱瘓

**原因**：沒有超時設定，請求一直等 payment-service 回應，連線池耗盡

**修正**：
1. 加入超時：
```yaml
# Istio VirtualService
timeout: 2s
```

2. 加入熔斷器：
```yaml
# Istio DestinationRule
outlierDetection:
  consecutiveErrors: 5
  interval: 30s
```

3. 限制連線池：
```yaml
connectionPool:
  http:
    http1MaxPendingRequests: 50
```

再次實驗：payment-service 變慢，但其他服務正常。

## 混沌工程文化

### Game Day

定期舉辦「Game Day」，全公司一起：
1. 選擇一個災難場景
2. 在生產環境執行
3. 觀察系統反應
4. 團隊協作處理

例如：
- 關閉一個 Availability Zone
- 模擬資料中心斷網
- DDoS 攻擊

### Blameless Post-Mortem

實驗失敗不是壞事，是學習機會。寫 Post-Mortem：
1. 發生了什麼
2. 為什麼發生
3. 如何修正
4. 如何預防

重點：**不責怪個人**，focus 在系統改進。

## 心得

一開始聽到混沌工程，覺得太瘋狂。主動在生產環境搞破壞？老闆會殺了我！

但實踐後發現，混沌工程讓我們對系統有更大的信心。以前總是擔心「如果 X 壞了怎麼辦」，現在我們知道「X 壞了，系統還能運作」。

最大的收穫是發現了很多隱藏的問題：
- 沒有處理的錯誤
- 不合理的超時設定
- 缺少降級機制
- 單點故障

這些問題在平時不會發現，只有在故障時才會爆發。混沌工程讓我們提前發現並修正。

建議：
1. 從非生產環境開始
2. 從小範圍開始（一個服務、一個 Pod）
3. 有監控和告警
4. 建立回滾計畫

混沌工程不是搞破壞，是建立信心的過程。

下週要研究資料庫遷移策略，如何安全地遷移資料庫而不影響服務。
