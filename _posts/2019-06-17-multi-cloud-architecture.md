---
layout: post
title: "多雲架構設計"
date: 2019-06-17 10:00:00 +0800
categories: [DevOps, Cloud]
tags: [Multi-Cloud, AWS, Azure, GCP, Cloud Native]
---

上週學習了 DevSecOps（參考 [DevSecOps 入門](/posts/2019/06/10/devsecops-intro/)），這週來研究多雲架構 (Multi-Cloud Architecture)。

我們的情況：
- 目前：全部在 AWS
- 問題：單一雲端廠商風險（鎖定、價格、可用性）
- 需求：考慮多雲策略

CTO 的問題：「如果 AWS 掛了怎麼辦？如果價格突然漲 50% 怎麼辦？」

這週研究多雲的可行性和挑戰。

> 雲端平台：AWS、Azure、GCP

## 為什麼多雲

### 優點

**1. 避免供應商鎖定 (Vendor Lock-in)**

單一雲端的風險：
- 價格調漲無法抵抗
- 服務條款變更
- 技術路線不符合需求
- 遷移困難（已深度整合）

**2. 提高可用性**

- 單一雲端區域故障 → 影響所有服務
- 多雲 → 雲端 A 掛了，切換到雲端 B

**3. 成本優化**

- 不同雲端價格不同
- 選擇最便宜的服務
- 談判籌碼（競爭壓力）

**4. 符合法規要求**

- 某些國家要求資料在地化
- 單一雲端可能沒有該地區的資料中心

**5. 最佳服務組合**

- AWS：最完整的服務
- GCP：最強的 AI/ML、Kubernetes (GKE)
- Azure：最佳 .NET 整合、企業市場

### 缺點

**1. 複雜度大增**

- 不同的 API、工具、概念
- 需要多套技能
- 監控、日誌分散

**2. 成本增加**

- 跨雲資料傳輸費用高
- 需要更多人力維護
- 重複的基礎設施

**3. 一致性挑戰**

- 不同的網路模型
- 不同的 IAM 系統
- 不同的資料庫

## 多雲策略

### 策略一：Redundant Multi-Cloud（冗餘多雲）

在多個雲端部署相同的應用，災難恢復用。

```
AWS (主要)              Azure (備援)
├─ K8s Cluster         ├─ AKS Cluster
├─ RDS PostgreSQL      ├─ Azure Database
├─ S3                  ├─ Blob Storage
└─ Route53             └─ Traffic Manager
   ↓                      ↓
  使用者 ←─ (故障切換) ─→ 使用者
```

**優點**：
- 高可用性
- 災難恢復

**缺點**：
- 成本高（雙倍）
- 資料同步複雜

**適合**：關鍵系統（金融、醫療）

### 策略二：Hybrid Multi-Cloud（混合多雲）

不同功能用不同雲端。

```
AWS
├─ 核心業務（訂單、支付）
└─ RDS PostgreSQL

GCP
├─ 資料分析
├─ BigQuery
└─ AI/ML (Vertex AI)

Azure
├─ 辦公系統（AD、Exchange）
└─ CosmosDB
```

**優點**：
- 使用各雲端優勢
- 成本效益

**缺點**：
- 資料在多個雲端
- 需要跨雲整合

**適合**：大企業

### 策略三：Cloud-Agnostic（雲端無關）

使用開源工具，容易在雲端間遷移。

**基礎設施**：
- Kubernetes（而不是 AWS ECS、Azure AKS）
- Terraform（而不是 CloudFormation）
- Prometheus/Grafana（而不是 CloudWatch）

**資料庫**：
- PostgreSQL（而不是 RDS）
- MongoDB（而不是 DynamoDB）
- Redis（而不是 ElastiCache）

**儲存**：
- S3-compatible API（MinIO、Ceph）

**優點**：
- 容易遷移
- 不被鎖定

**缺點**：
- 放棄雲端特有功能
- 需要自己維護

**適合**：新創公司

## 實作：Kubernetes 多雲

我們選擇策略三（Cloud-Agnostic），使用 Kubernetes。

### 部署到多個雲端

**AWS (EKS)**：
```bash
# 建立 EKS cluster
eksctl create cluster \
  --name production-aws \
  --region ap-southeast-1 \
  --nodegroup-name standard-workers \
  --node-type m5.large \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10
```

**GCP (GKE)**：
```bash
# 建立 GKE cluster
gcloud container clusters create production-gcp \
  --region asia-southeast1 \
  --machine-type n1-standard-2 \
  --num-nodes 3 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10
```

**Azure (AKS)**：
```bash
# 建立 AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name production-azure \
  --location southeastasia \
  --node-count 3 \
  --node-vm-size Standard_D2_v2 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10
```

### 統一部署

使用相同的 Kubernetes manifests：

`order-service.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry.azurecr.io/order-service:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

部署到所有叢集：
```bash
# AWS
kubectl --context=eks-aws apply -f order-service.yaml

# GCP
kubectl --context=gke-gcp apply -f order-service.yaml

# Azure
kubectl --context=aks-azure apply -f order-service.yaml
```

## 多雲資料管理

### 資料同步

**問題**：資料在多個雲端，如何同步？

**解決方案一：主從複寫**

```
AWS RDS (主) ──複寫→ GCP CloudSQL (從)
               ──複寫→ Azure Database (從)
```

PostgreSQL 設定（AWS 主）：
```sql
-- 建立複寫使用者
CREATE USER replicator WITH REPLICATION PASSWORD 'password';

-- 配置
-- postgresql.conf
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
```

GCP 從節點：
```bash
# 使用 pglogical
CREATE EXTENSION pglogical;

SELECT pglogical.create_subscription(
    subscription_name := 'aws_to_gcp',
    provider_dsn := 'host=aws-db.xxx.rds.amazonaws.com dbname=ecommerce user=replicator password=xxx'
);
```

**解決方案二：Event-driven 同步**

```
AWS
├─ Order Service → SQS → Lambda → 寫入 S3
                                    ↓
                                  複製到 GCP
                                    ↓
GCP                              Cloud Storage
├─ Order Service ← Pub/Sub ← Cloud Function
```

使用事件驅動，避免直接資料庫複寫的複雜度。

### 跨雲資料傳輸

**問題**：資料傳輸費用高。

AWS → GCP：$0.09/GB
GCP → AWS：$0.12/GB

**優化**：
1. **壓縮**：傳輸前壓縮（省 80%）
2. **增量傳輸**：只傳輸變更部分
3. **批次傳輸**：晚上低流量時段傳輸
4. **專線**：AWS Direct Connect + GCP Interconnect（大量資料）

### 分散式資料庫

使用支援多雲的資料庫：

**CockroachDB**（開源，類似 Google Spanner）：

```sql
-- 建立 multi-region database
CREATE DATABASE ecommerce PRIMARY REGION "aws-southeast-1" 
  REGIONS "gcp-asia-southeast1", "azure-southeastasia";

-- 資料自動分散到三個雲端
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INT,
    total DECIMAL,
    created_at TIMESTAMP
) LOCALITY REGIONAL BY ROW;
```

資料自動複寫到三個雲端，任一雲端故障不影響服務。

## 多雲網路

### Service Mesh（Istio Multi-Cluster）

連接多個 Kubernetes 叢集。

**架構**：
```
AWS EKS              GCP GKE              Azure AKS
  ├─ Istio             ├─ Istio             ├─ Istio
  ├─ order-service     ├─ order-service     ├─ order-service
  └─ payment-service   └─ payment-service   └─ payment-service
     ↕                    ↕                    ↕
  ────────────── Service Mesh ──────────────
```

安裝 Istio（每個叢集）：
```bash
# AWS
istioctl install --set profile=default \
  --set values.global.meshID=multi-cloud \
  --set values.global.network=aws-network

# GCP
istioctl install --set profile=default \
  --set values.global.meshID=multi-cloud \
  --set values.global.network=gcp-network

# Azure
istioctl install --set profile=default \
  --set values.global.meshID=multi-cloud \
  --set values.global.network=azure-network
```

配置跨叢集通訊：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-network-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
  - port:
      number: 15443
      name: tls
      protocol: TLS
    tls:
      mode: AUTO_PASSTHROUGH
    hosts:
    - "*.global"
```

現在 AWS 的 order-service 可以呼叫 GCP 的 payment-service！

```bash
curl order-service.aws/orders
# 內部呼叫 payment-service.gcp（透過 service mesh）
```

### VPN/VPC Peering

**AWS ←→ GCP**：

使用 Cloud VPN：
```bash
# GCP 建立 VPN Gateway
gcloud compute vpn-gateways create gcp-vpn-gw \
  --network=gcp-vpc \
  --region=asia-southeast1

# AWS 建立 Customer Gateway
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip <GCP_VPN_IP> \
  --bgp-asn 65000

# 建立 VPN Connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id <CGW_ID> \
  --vpn-gateway-id <VGW_ID>
```

私有網路打通，資料傳輸不走公網（更安全、更便宜）。

## 多雲 CI/CD

### 統一 Pipeline

`.gitlab-ci.yml`：
```yaml
stages:
  - build
  - test
  - deploy-aws
  - deploy-gcp
  - deploy-azure

build:
  stage: build
  script:
    - docker build -t order-service:${CI_COMMIT_SHA} .
    - docker tag order-service:${CI_COMMIT_SHA} myregistry.azurecr.io/order-service:${CI_COMMIT_SHA}
    - docker push myregistry.azurecr.io/order-service:${CI_COMMIT_SHA}

deploy_aws:
  stage: deploy-aws
  script:
    - kubectl --context=eks-aws set image deployment/order-service order-service=myregistry.azurecr.io/order-service:${CI_COMMIT_SHA}
  only:
    - master

deploy_gcp:
  stage: deploy-gcp
  script:
    - kubectl --context=gke-gcp set image deployment/order-service order-service=myregistry.azurecr.io/order-service:${CI_COMMIT_SHA}
  only:
    - master
  when: manual  # 手動觸發

deploy_azure:
  stage: deploy-azure
  script:
    - kubectl --context=aks-azure set image deployment/order-service order-service=myregistry.azurecr.io/order-service:${CI_COMMIT_SHA}
  only:
    - master
  when: manual
```

一次建置，部署到三個雲端。

### Container Registry

**問題**：每個雲端有自己的 registry（ECR、GCR、ACR）。

**解決**：使用統一的 registry（Azure ACR 支援 geo-replication）：

```bash
# 建立 ACR
az acr create \
  --name myregistry \
  --resource-group myResourceGroup \
  --sku Premium

# 啟用 geo-replication
az acr replication create \
  --registry myregistry \
  --location southeastasia

az acr replication create \
  --registry myregistry \
  --location eastus
```

所有雲端的叢集都從 ACR pull image，且使用最近的 replica（快速、便宜）。

## 多雲監控

### 統一監控（Prometheus + Grafana）

每個雲端部署 Prometheus：
```bash
# AWS
helm install prometheus prometheus-community/prometheus --namespace monitoring

# GCP
helm install prometheus prometheus-community/prometheus --namespace monitoring

# Azure
helm install prometheus prometheus-community/prometheus --namespace monitoring
```

中央 Grafana 聚合資料：
```yaml
# Grafana datasources
apiVersion: 1
datasources:
- name: AWS Prometheus
  type: prometheus
  url: http://prometheus.aws.example.com
  
- name: GCP Prometheus
  type: prometheus
  url: http://prometheus.gcp.example.com
  
- name: Azure Prometheus
  type: prometheus
  url: http://prometheus.azure.example.com
```

Dashboard 查看所有雲端的指標：
```promql
# 總請求數（所有雲端）
sum(
  http_requests_total{job="order-service", cluster="aws"} +
  http_requests_total{job="order-service", cluster="gcp"} +
  http_requests_total{job="order-service", cluster="azure"}
)
```

### 分散式追蹤（Jaeger）

**問題**：請求跨多個雲端，如何追蹤？

```
使用者請求
  ↓
AWS EKS (API Gateway)
  ↓
GCP GKE (Order Service)
  ↓
Azure AKS (Payment Service)
  ↓
回應
```

**解決**：統一的 Jaeger Collector

```yaml
# 每個雲端的應用程式都發送 traces 到中央 Jaeger
spring:
  zipkin:
    base-url: http://jaeger-collector.central.example.com:9411
  sleuth:
    sampler:
      probability: 0.1  # 取樣 10%
```

Jaeger UI 可以看到完整的請求流程（跨雲端）。

## 成本管理

### 多雲成本可視化

使用 CloudHealth 或 Kubecost：

```bash
# Kubecost（支援多雲）
helm install kubecost kubecost/cost-analyzer \
  --set prometheus.server.global.external_labels.cluster_id=aws-eks \
  --set kubecostToken="YOUR_TOKEN"

# 每個叢集安裝，中央聚合
```

Dashboard 顯示：
```
總成本：$15,000/月
  AWS：   $8,000 (53%)
  GCP：   $4,000 (27%)
  Azure： $3,000 (20%)

按服務：
  Order Service：  $6,000
    AWS：   $3,500
    GCP：   $1,500
    Azure： $1,000
```

### 成本優化策略

**1. 智慧路由**

根據成本路由請求：

```yaml
# Istio VirtualService
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        region:
          exact: "asia"
    route:
    - destination:
        host: order-service.gcp  # GCP 在亞洲較便宜
      weight: 80
    - destination:
        host: order-service.aws
      weight: 20
  - route:
    - destination:
        host: order-service.aws  # 預設 AWS
```

**2. 批次工作選擇最便宜雲端**

```python
# 計算資料處理任務
import boto3, google.cloud.batch

def process_data(data):
    # 比較價格
    aws_price = get_spot_price('aws', 'c5.large')
    gcp_price = get_spot_price('gcp', 'n1-standard-2')
    
    if aws_price < gcp_price:
        # 在 AWS 執行
        run_on_aws(data)
    else:
        # 在 GCP 執行
        run_on_gcp(data)
```

## 挑戰與解決

### 挑戰一：不同的 IAM 系統

**問題**：AWS IAM、GCP IAM、Azure AD 完全不同。

**解決**：使用 OIDC (OpenID Connect) 統一認證。

```yaml
# Kubernetes RBAC（所有雲端相同）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update"]
```

使用者透過 SSO（Okta、Auth0）登入，取得 OIDC token，所有叢集都接受。

### 挑戰二：不同的網路模型

**問題**：
- AWS：VPC、Security Group
- GCP：VPC、Firewall Rules
- Azure：VNet、NSG

**解決**：Kubernetes Network Policy（抽象層）

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
```

所有雲端用相同規則。

### 挑戰三：供應商特定功能

**問題**：想用 AWS Lambda、GCP BigQuery、Azure Cosmos DB（供應商特定）。

**解決**：抽象層 + Adapter 模式

```java
// 抽象層
public interface StorageService {
    void save(String key, byte[] data);
    byte[] load(String key);
}

// AWS 實作
@Profile("aws")
public class S3StorageService implements StorageService {
    @Autowired
    private AmazonS3 s3Client;
    
    public void save(String key, byte[] data) {
        s3Client.putObject(bucket, key, new ByteArrayInputStream(data), metadata);
    }
}

// Azure 實作
@Profile("azure")
public class BlobStorageService implements StorageService {
    @Autowired
    private BlobServiceClient blobClient;
    
    public void save(String key, byte[] data) {
        blobClient.getBlobContainerClient(container)
            .getBlobClient(key)
            .upload(new ByteArrayInputStream(data), data.length);
    }
}
```

切換雲端只需改配置，不需改程式碼。

## 心得

多雲不是銀彈，複雜度和成本都會增加。

我們的結論：
- **不要**為了多雲而多雲
- **應該**先做 Cloud-Agnostic（Kubernetes、Terraform、開源工具）
- **可以**關鍵服務在多雲部署（災難恢復）
- **不要**每個服務都多雲（成本太高）

實際策略：
- 80% 服務在 AWS（主要）
- 20% 關鍵服務在 AWS + GCP（冗餘）
- 使用開源工具，保持遷移能力
- 定期演練跨雲切換

多雲給我們選擇權，但不一定要 100% 使用。重點是：**不被鎖定，保持靈活**。

下週要研究 DevOps 工具鏈整合，把所有工具串起來。
