---
layout: post
title: "雲端成本優化"
date: 2019-06-03 10:00:00 +0800
categories: [DevOps, Cloud]
tags: [Cost Optimization, AWS, FinOps, Cloud Economics]
---

上週學習了 SRE 實踐（參考 [SRE 實踐：錯誤預算與 SLO](/posts/2019/05/27/sre-error-budget-slo/)），這週面對一個現實問題：AWS 帳單。

上個月收到帳單：$12,000！比預期多了 $4,000。老闆臉都綠了。

CTO 要我們：「兩週內降低 30% 成本，但不能影響服務品質。」

這週開始學習雲端成本優化（FinOps）。

> 工具：AWS Cost Explorer、Kubecost、CloudHealth

## 成本分析

### 查看 AWS Cost Explorer

登入 AWS Console → Cost Explorer

**按服務分類**（上個月）：
```
EC2：           $6,000 (50%)
RDS：           $2,400 (20%)
EBS：           $1,200 (10%)
Data Transfer： $1,200 (10%)
S3：            $600   (5%)
其他：          $600   (5%)
```

EC2 是最大開銷！

**按標籤分類**：
```
production：    $8,000
staging：       $2,000
development：   $2,000
```

### 識別浪費

#### 1. 閒置資源

**停止的 EC2 但 EBS 還在收費**：
```bash
# 查找停止的 instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=stopped" \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# 輸出
i-0123456789abcdef0  old-test-server
i-0fedcba9876543210  backup-server
i-0abcdef1234567890  nobody-knows
```

這些 instances 停止了，但 EBS 還在（每月 $8/100GB）。

**刪除未使用的 EBS**：
```bash
# 查找未連接的 volumes
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[].[VolumeId,Size,CreateTime]' \
  --output table

# 刪除（確認不需要後）
aws ec2 delete-volume --volume-id vol-0123456789abcdef0
```

**節省**：$800/月

#### 2. 過度配置

**資源使用率低**：

查看 EC2 CPU 使用率（CloudWatch）：
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2019-05-01T00:00:00Z \
  --end-time 2019-06-01T00:00:00Z \
  --period 3600 \
  --statistics Average
```

結果：
```
m5.2xlarge (8 vCPU, 32GB)
平均 CPU：15%  ← 太浪費！
平均記憶體：8GB (25%)
```

**降級 instance type**：
- 從 m5.2xlarge → m5.large（2 vCPU, 8GB）
- 成本：$0.096/hr → $0.048/hr（省 50%）

**節省**：$1,000/月（多個 instances）

#### 3. 開發/測試環境浪費

**問題**：開發環境 24/7 運行，但晚上和週末沒人用。

**解決**：自動關機

使用 AWS Instance Scheduler：

安裝 CloudFormation template：
```bash
aws cloudformation create-stack \
  --stack-name instance-scheduler \
  --template-url https://s3.amazonaws.com/solutions-reference/aws-instance-scheduler/latest/instance-scheduler.template
```

配置排程：
```json
{
  "name": "office-hours",
  "periods": [{
    "description": "Office hours",
    "begintime": "08:00",
    "endtime": "20:00",
    "weekdays": ["mon-fri"]
  }]
}
```

標記 instances：
```bash
aws ec2 create-tags \
  --resources i-0123456789abcdef0 \
  --tags Key=Schedule,Value=office-hours
```

開發環境自動：
- 週一至週五 8:00 啟動
- 週一至週五 20:00 關閉
- 週末關閉

**節省**：每天運行 12 小時 vs 24 小時 = 省 50%
- 開發環境成本：$2,000 → $1,000

**總節省**：$1,000/月

## Reserved Instances & Savings Plans

### Reserved Instances (RI)

承諾使用 1-3 年，換取折扣（高達 72%）。

**Standard RI（標準預留實例）**：
- 折扣：最高 72%
- 限制：固定 instance type、region、OS
- 適合：穩定、可預測的工作負載

**Convertible RI（可轉換預留實例）**：
- 折扣：最高 54%
- 彈性：可以改變 instance type
- 適合：需要彈性但使用穩定

### 購買策略

分析過去 3 個月的 usage：
```bash
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute" \
  --lookback-period-in-days SIXTY_DAYS
```

AWS 建議：
```
Instance Type: m5.large
Region: ap-southeast-1
Quantity: 10
Term: 1 year
Payment: All Upfront
Estimated Savings: $3,600/year
```

購買 RI：
```bash
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id xxx \
  --instance-count 10
```

### Savings Plans

更彈性的選擇（2019 年推出）：

**Compute Savings Plans**：
- 承諾每小時花費（例如：$10/hour）
- 適用於 EC2、Fargate、Lambda
- 任何 region、instance type、OS

**EC2 Instance Savings Plans**：
- 承諾特定 instance family（例如：M5）
- 任何 region、size、OS

**我們的策略**：
- Production 穩定工作負載：Standard RI（折扣最高）
- Production 彈性工作負載：Compute Savings Plans
- Staging/Dev：On-Demand + 自動關機

**節省**：$2,400/月

## Kubernetes 成本優化

### 安裝 Kubecost

```bash
kubectl create namespace kubecost
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --set kubecostToken="your-token"
```

查看 Dashboard：
```bash
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090
open http://localhost:9090
```

### 分析成本

**按 Namespace**：
```
production:     $4,000/月
  - order-service:   $1,200
  - payment-service: $800
  - user-service:    $600
  - ...

staging:        $1,500/月
development:    $800/月
```

**按 Pod**：
```
order-service-7d8c9f-x7k2m:
  CPU 請求：    2 cores → 實際使用：0.3 cores (15%)  ← 過度配置！
  記憶體請求：  4Gi → 實際使用：800Mi (20%)  ← 過度配置！
  成本：        $120/月
```

### 優化 Resource Requests

**問題**：requests 設定太高，浪費資源。

修改前：
```yaml
resources:
  requests:
    cpu: "2000m"
    memory: "4Gi"
  limits:
    cpu: "4000m"
    memory: "8Gi"
```

根據實際使用量（P95）：
- CPU：0.5 cores
- 記憶體：1.2Gi

修改後：
```yaml
resources:
  requests:
    cpu: "600m"      # 0.5 + 20% 緩衝
    memory: "1.5Gi"  # 1.2 + 25% 緩衝
  limits:
    cpu: "1000m"
    memory: "2Gi"
```

**結果**：
- Node 利用率：30% → 70%
- 所需 Nodes：20 → 10
- **節省**：$2,000/月

### Cluster Autoscaler 優化

**問題**：Cluster Autoscaler 反應慢，經常 over-provision。

優化配置：
```yaml
# cluster-autoscaler deployment
command:
  - ./cluster-autoscaler
  - --scale-down-delay-after-add=5m      # 擴展後 5 分鐘才能縮減
  - --scale-down-unneeded-time=5m        # Node 空閒 5 分鐘後縮減
  - --skip-nodes-with-local-storage=false
  - --skip-nodes-with-system-pods=false
  - --expander=least-waste               # 選擇浪費最少的 node type
```

**結果**：Node 數量波動減少，成本降低 15%。

## 資料庫優化

### RDS 成本：$2,400/月

分析：
```
db.r5.2xlarge (8 vCPU, 64GB)
儲存：500GB (gp2)
備份：500GB
Multi-AZ：Yes
```

### 優化一：降級 Instance

查看使用率（CloudWatch）：
- CPU：平均 25%，峰值 50%
- 記憶體：使用 30GB (47%)

**降級**：
- db.r5.2xlarge → db.r5.xlarge（4 vCPU, 32GB）
- 成本：$0.58/hr → $0.29/hr

**節省**：$200/月

### 優化二：儲存優化

**問題**：使用 gp2（通用 SSD），但大多數操作是順序讀取。

**優化**：
- 熱資料（最近 3 個月）：gp2（200GB）
- 冷資料（3 個月前）：歸檔到 S3（300GB）

建立歸檔機制：
```python
# archive_old_orders.py
import psycopg2
import boto3
import csv
from datetime import datetime, timedelta

s3 = boto3.client('s3')
conn = psycopg2.connect("postgresql://...")

# 查詢 3 個月前的訂單
three_months_ago = datetime.now() - timedelta(days=90)
cur = conn.cursor()
cur.execute("""
    SELECT * FROM orders 
    WHERE created_at < %s
""", (three_months_ago,))

# 匯出到 CSV
with open('/tmp/old_orders.csv', 'w') as f:
    writer = csv.writer(f)
    writer.writerow([desc[0] for desc in cur.description])
    writer.writerows(cur.fetchall())

# 上傳到 S3
s3.upload_file(
    '/tmp/old_orders.csv',
    'my-archive-bucket',
    f'orders/{datetime.now().strftime("%Y-%m")}/orders.csv.gz'
)

# 刪除舊資料（保留索引表）
cur.execute("""
    DELETE FROM orders WHERE created_at < %s
""", (three_months_ago,))
conn.commit()
```

定期執行（每月）。

**節省**：
- gp2：500GB × $0.115/GB = $57.5/月
- S3：300GB × $0.025/GB = $7.5/月
- **省**：$50/月

### 優化三：Read Replica

**問題**：大量查詢在主資料庫執行，影響寫入效能。

**解決**：
1. 建立 Read Replica：
```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-read-replica \
  --source-db-instance-identifier mydb \
  --db-instance-class db.r5.large  # 比主資料庫小
```

2. 應用程式分離讀寫：
```yaml
spring:
  datasource:
    primary:
      url: jdbc:postgresql://mydb.xxx.rds.amazonaws.com:5432/ecommerce
    replica:
      url: jdbc:postgresql://mydb-read-replica.xxx.rds.amazonaws.com:5432/ecommerce
```

```java
@Transactional(readOnly = true)
@ReadOnly  // 自定義 annotation
public List<Order> getOrders(Long userId) {
    // 路由到 read replica
    return orderRepository.findByUserId(userId);
}

@Transactional
public Order createOrder(Order order) {
    // 路由到 primary
    return orderRepository.save(order);
}
```

3. 主資料庫可以降級（寫入壓力降低）：
- db.r5.xlarge → db.r5.large

**成本**：
- 主：$0.29/hr → $0.145/hr
- Replica：$0.145/hr
- 總計：$0.29/hr（相同），但效能提升 2 倍

## 資料傳輸成本優化

### Data Transfer：$1,200/月

分析：
```
Internet → AWS:    $0      (免費)
AWS → Internet:    $1,000  (10TB × $0.09/GB)
跨 AZ:             $200    (2TB × $0.01/GB)
```

### 優化一：使用 CloudFront

**問題**：靜態資源（圖片、CSS、JS）直接從 S3 下載。

**解決**：使用 CloudFront CDN
- S3 → CloudFront：$0.02/GB（內部傳輸）
- CloudFront → Internet：$0.085/GB（比 S3 便宜）
- 額外好處：更快（邊緣節點）

建立 CloudFront Distribution：
```bash
aws cloudfront create-distribution \
  --origin-domain-name my-bucket.s3.amazonaws.com \
  --default-root-object index.html
```

更新應用程式：
```html
<!-- 修改前 -->
<img src="https://my-bucket.s3.amazonaws.com/images/product.jpg">

<!-- 修改後 -->
<img src="https://d123456abcdef.cloudfront.net/images/product.jpg">
```

**節省**：8TB × ($0.09 - $0.085) = $40/月
**額外好處**：回應時間從 500ms → 50ms

### 優化二：減少跨 AZ 傳輸

**問題**：Pod 跨 AZ 呼叫服務。

```
Pod (AZ-a) → Service → Pod (AZ-b)  ← 跨 AZ，收費 $0.01/GB
```

**解決**：啟用 Topology Aware Routing

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  annotations:
    service.kubernetes.io/topology-aware-hints: auto
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
```

Kubernetes 會優先路由到同一 AZ 的 Pod。

**節省**：$150/月

## S3 成本優化

### S3：$600/月

分析：
```
Standard:           500GB × $0.025 = $12.5
請求 (PUT/GET):     $500
資料傳輸:           $87.5
```

### 優化一：Lifecycle Policy

**問題**：所有資料都在 Standard（最貴）。

**解決**：自動轉移到便宜的儲存類別。

```json
{
  "Rules": [
    {
      "Id": "Archive old files",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "backups/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"  // 30 天後 → Infrequent Access
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"  // 90 天後 → Glacier
        }
      ],
      "Expiration": {
        "Days": 365  // 1 年後刪除
      }
    },
    {
      "Id": "Delete incomplete uploads",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

套用：
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

**節省**：
- Standard：500GB → 100GB
- Standard-IA：300GB × $0.0125 = $3.75
- Glacier：100GB × $0.004 = $0.4
- **省**：$200/月

### 優化二：壓縮

**問題**：日誌檔案未壓縮。

```python
# 上傳前壓縮
import gzip
import boto3

s3 = boto3.client('s3')

# 壓縮
with open('app.log', 'rb') as f_in:
    with gzip.open('app.log.gz', 'wb') as f_out:
        f_out.writelines(f_in)

# 上傳
s3.upload_file('app.log.gz', 'my-bucket', 'logs/app.log.gz')
```

壓縮率：10:1（1GB → 100MB）

**節省**：儲存和傳輸都省 90%

## 監控成本

### 設定預算告警

AWS Budgets：
```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

`budget.json`：
```json
{
  "BudgetName": "Monthly Cost Budget",
  "BudgetLimit": {
    "Amount": "10000",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
```

`notifications.json`：
```json
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "EMAIL",
        "Address": "team@example.com"
      }
    ]
  }
]
```

超過 80% 預算時發送告警。

### Cost Anomaly Detection

AWS 會自動偵測異常花費。

啟用：
```bash
aws ce create-anomaly-monitor \
  --monitor-name "My Monitor" \
  --monitor-type DIMENSIONAL \
  --monitor-dimension SERVICE
```

例如：某天 EC2 花費突然增加 50%，自動告警。

## 成果

經過兩週優化：

| 項目 | 原始成本 | 優化後 | 節省 |
|-----|---------|--------|------|
| EC2 | $6,000 | $3,600 | $2,400 (40%) |
| RDS | $2,400 | $2,000 | $400 (17%) |
| EBS | $1,200 | $600 | $600 (50%) |
| Data Transfer | $1,200 | $1,000 | $200 (17%) |
| S3 | $600 | $400 | $200 (33%) |
| **總計** | **$12,000** | **$8,000** | **$4,000 (33%)** |

✅ 達成目標（30% 節省）！

而且服務品質沒有下降（監控 SLO，都達標）。

## 心得

成本優化不只是省錢，也是優化系統的好機會。

過程中發現很多問題：
- 資源過度配置（CPU 只用 15%）
- 閒置資源（忘記刪除的測試環境）
- 架構不合理（跨 AZ 傳輸太多）

優化後系統反而更好：
- 資源利用率提高
- 架構更合理
- 監控更完善

建議：
1. **定期檢視**：每月 review 成本
2. **標籤管理**：所有資源打標籤（team, env, project）
3. **RI/Savings Plans**：穩定工作負載一定要買
4. **自動關機**：開發/測試環境不要 24/7
5. **Right-sizing**：根據實際使用量調整

成本優化是持續的過程，不是一次性的。

下週要研究 DevSecOps，把安全整合到 DevOps 流程。
