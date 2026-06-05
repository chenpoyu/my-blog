---
layout: post
title: "Kubernetes 容器編排入門"
date: 2018-12-31 09:00:00 +0800
categories: [DevOps, Container, Kubernetes]
tags: [Docker, Kubernetes, DevOps, High Availability]
---

前幾週學會了 Docker 和 Docker Compose，但這些工具主要用於單機環境。如果要在多台機器上部署容器、實現負載平衡、自動擴展，就需要容器編排工具。

這週開始研究 Kubernetes（簡稱 K8s），目前最流行的容器編排平台。

> 使用版本：Kubernetes 1.13（2018 年 12 月剛發布）

## 為什麼需要 Kubernetes

上週我們用 Docker Compose 管理多容器應用，但面臨一些問題：

### 單機限制

Docker Compose 只能在單台機器上運行。如果：
- 流量增加，需要橫向擴展（多台機器）
- 單台機器掛掉，服務就中斷

### 手動擴展

要增加實例數量：
```bash
docker-compose up -d --scale app=3
```

但這只能在同一台機器上擴展，而且：
- 沒有負載平衡
- 沒有健康檢查和自動重啟
- 無法跨多台機器

### 更新和回滾

更新應用時：
```bash
docker-compose down
docker-compose up -d
```

這會造成服務中斷（downtime）。

## Kubernetes 能做什麼

Kubernetes 是 Google 開源的容器編排系統，提供：

1. **服務發現和負載平衡**：自動分配流量到健康的容器
2. **自動擴展**：根據 CPU 使用率自動增減容器數量
3. **自我修復**：容器掛掉自動重啟
4. **滾動更新**：零停機時間更新應用
5. **密鑰和設定管理**：安全地管理敏感資訊
6. **跨機器部署**：在多台機器組成的叢集中部署

簡單說：Kubernetes 讓你像管理一台超級電腦一樣管理多台機器。

## Kubernetes 核心概念

### Cluster（叢集）

一組機器（nodes）組成的集合。

- **Master Node**：控制平面，管理整個叢集
- **Worker Node**：執行容器的機器

### Pod

Kubernetes 的最小部署單位，通常包含一個或多個容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
  - name: my-app
    image: my-shop-api:1.0.0
    ports:
    - containerPort: 8080
```

Pod 裡的容器：
- 共享網路（localhost）
- 共享儲存空間
- 一起被排程到同一個 Node

### Deployment

管理 Pod 的部署和更新。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3  # 執行 3 個 Pod
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-shop-api:1.0.0
        ports:
        - containerPort: 8080
```

Deployment 負責：
- 確保指定數量的 Pod 正在運行
- 滾動更新
- 回滾到先前版本

### Service

為一組 Pod 提供穩定的網路端點。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

Service 類型：
- **ClusterIP**：叢集內部訪問（預設）
- **NodePort**：透過 Node 的 port 對外開放
- **LoadBalancer**：使用雲端負載平衡器（AWS ELB、GCP LB）

### Namespace

虛擬叢集，用來隔離資源。

```bash
# 建立 namespace
kubectl create namespace dev
kubectl create namespace prod

# 在特定 namespace 部署
kubectl apply -f deployment.yaml -n dev
```

## 安裝 Kubernetes

### 本機開發：Minikube

Minikube 在本機跑一個單節點的 Kubernetes 叢集，適合學習和開發。

**安裝 Minikube（macOS）：**
```bash
brew install minikube
```

**啟動 Minikube：**
```bash
minikube start --cpus=2 --memory=4096

# 😄  minikube v1.6.2 on Darwin 10.14
# ✨  Automatically selected the hyperkit driver
# 🔥  Creating hyperkit VM ...
# 🐳  Preparing Kubernetes v1.13.0 on Docker 18.09.9 ...
# 🚀  Launching Kubernetes ...
# 🏄  Done! kubectl is now configured to use "minikube"
```

**驗證安裝：**
```bash
kubectl version --short
# Client Version: v1.13.0
# Server Version: v1.13.0

kubectl get nodes
# NAME       STATUS   ROLES    AGE   VERSION
# minikube   Ready    master   1m    v1.13.0
```

### 安裝 kubectl

kubectl 是 Kubernetes 的命令列工具。

```bash
# macOS
brew install kubectl

# 驗證
kubectl version --client
```

## 第一個 Kubernetes 應用

### 部署 Nginx

```bash
# 建立 Deployment
kubectl create deployment nginx --image=nginx:1.15

# 查看 Deployment
kubectl get deployments
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# nginx   1/1     1            1           10s

# 查看 Pod
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-5c7588df-xxxxx     1/1     Running   0          20s
```

### 暴露服務

```bash
# 建立 Service
kubectl expose deployment nginx --port=80 --type=NodePort

# 查看 Service
kubectl get services
# NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# nginx   NodePort   10.96.123.456   <none>        80:30123/TCP   5s

# 訪問服務（Minikube）
minikube service nginx
# 會自動打開瀏覽器訪問 Nginx
```

### 擴展應用

```bash
# 擴展到 3 個實例
kubectl scale deployment nginx --replicas=3

# 查看 Pod
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-5c7588df-xxxxx     1/1     Running   0          2m
# nginx-5c7588df-yyyyy     1/1     Running   0          5s
# nginx-5c7588df-zzzzz     1/1     Running   0          5s
```

### 更新應用

```bash
# 更新映像檔
kubectl set image deployment/nginx nginx=nginx:1.16

# 查看滾動更新狀態
kubectl rollout status deployment/nginx
# Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas...
# deployment "nginx" successfully rolled out

# 查看歷史
kubectl rollout history deployment/nginx
```

### 回滾

```bash
# 回滾到上一個版本
kubectl rollout undo deployment/nginx

# 回滾到特定版本
kubectl rollout undo deployment/nginx --to-revision=1
```

## 部署 Spring Boot 應用到 K8s

### 準備映像檔

```bash
# 建置並推送到 Registry
mvn clean package
docker build -t registry.mycompany.com/myshop/my-shop-api:1.0.0 .
docker push registry.mycompany.com/myshop/my-shop-api:1.0.0
```

### 建立 Deployment YAML

`k8s/deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-shop-api
  labels:
    app: my-shop-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-shop-api
  template:
    metadata:
      labels:
        app: my-shop-api
    spec:
      containers:
      - name: my-shop-api
        image: registry.mycompany.com/myshop/my-shop-api:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:mysql://mysql-service:3306/myshop"
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
```

### 建立 Service YAML

`k8s/service.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-shop-api-service
spec:
  selector:
    app: my-shop-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

### 建立 Secret

儲存敏感資訊（密碼）：

```bash
kubectl create secret generic mysql-secret \
  --from-literal=username=shopuser \
  --from-literal=password=shoppass
```

### 部署

```bash
# 部署 Deployment
kubectl apply -f k8s/deployment.yaml

# 部署 Service
kubectl apply -f k8s/service.yaml

# 查看狀態
kubectl get all
```

### 查看日誌

```bash
# 查看所有 Pod 的日誌
kubectl logs -l app=my-shop-api

# 查看特定 Pod 的日誌
kubectl logs my-shop-api-xxxxx-xxxxx

# 即時追蹤日誌
kubectl logs -f my-shop-api-xxxxx-xxxxx
```

### 進入 Pod

```bash
kubectl exec -it my-shop-api-xxxxx-xxxxx -- /bin/sh
```

## ConfigMap 管理設定

不要把設定寫死在 Deployment，用 ConfigMap 管理。

`k8s/configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-shop-config
data:
  application.properties: |
    server.port=8080
    logging.level.root=INFO
    spring.jpa.hibernate.ddl-auto=validate
```

在 Deployment 中使用：

```yaml
spec:
  containers:
  - name: my-shop-api
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
  volumes:
  - name: config-volume
    configMap:
      name: my-shop-config
```

## 常用 kubectl 指令

### 查看資源

```bash
# 查看所有資源
kubectl get all

# 查看特定類型
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes

# 查看詳細資訊
kubectl describe pod my-shop-api-xxxxx

# 查看 YAML
kubectl get deployment my-shop-api -o yaml
```

### 刪除資源

```bash
# 刪除 Pod（Deployment 會自動建立新的）
kubectl delete pod my-shop-api-xxxxx

# 刪除 Deployment
kubectl delete deployment my-shop-api

# 刪除 Service
kubectl delete service my-shop-api-service

# 刪除所有資源
kubectl delete -f k8s/
```

### 除錯

```bash
# 查看 Pod 日誌
kubectl logs pod-name

# 查看上一個容器的日誌（容器重啟後）
kubectl logs pod-name --previous

# 進入 Pod
kubectl exec -it pod-name -- /bin/sh

# Port forwarding（本機訪問 Pod）
kubectl port-forward pod-name 8080:8080
```

## 遇到的問題

### 問題一：Pod 一直 Pending

```bash
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# my-shop-api-xxxxx        0/1     Pending   0          2m
```

查看原因：
```bash
kubectl describe pod my-shop-api-xxxxx
# Events:
#   Warning  FailedScheduling  ... 0/1 nodes available: insufficient memory.
```

原因：Node 資源不足。

解決方法：
1. 減少 Pod 的資源請求
2. 增加 Node（橫向擴展）
3. 使用更大的 Node

### 問題二：ImagePullBackOff

```bash
kubectl get pods
# NAME                     READY   STATUS             RESTARTS   AGE
# my-shop-api-xxxxx        0/1     ImagePullBackOff   0          1m
```

查看原因：
```bash
kubectl describe pod my-shop-api-xxxxx
# Failed to pull image "registry.mycompany.com/myshop/my-shop-api:1.0.0": 
# rpc error: code = Unknown desc = Error response from daemon: 
# pull access denied
```

原因：無法拉取映像檔（可能是權限或網路問題）。

解決方法：建立 imagePullSecrets

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.mycompany.com \
  --docker-username=developer \
  --docker-password=dev123456
```

在 Deployment 中使用：
```yaml
spec:
  template:
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - name: my-shop-api
        # ...
```

### 問題三：CrashLoopBackOff

Pod 不斷重啟：

```bash
kubectl get pods
# NAME                     READY   STATUS             RESTARTS   AGE
# my-shop-api-xxxxx        0/1     CrashLoopBackOff   5          3m
```

查看日誌：
```bash
kubectl logs my-shop-api-xxxxx
```

常見原因：
- 應用程式錯誤（連不上資料庫、設定錯誤）
- 健康檢查失敗
- 資源不足（OOM）

## 心得

Kubernetes 的學習曲線確實陡峭，概念很多，一開始很容易搞混 Pod、Deployment、Service 的關係。

但理解基本概念後，會發現 Kubernetes 真的很強大：
- 自動重啟掛掉的容器
- 滾動更新不需要停機
- 橫向擴展只要改個數字

目前我只在本機的 Minikube 玩，還沒在真實的叢集環境部署。下週要研究如何在多台機器上建立 Kubernetes 叢集，以及如何做好監控和日誌收集。

2018 年最後一篇文章，這一年學了好多東西！從 CI/CD、Docker 到 Kubernetes，DevOps 之路才剛開始。

新年快樂！2019 年繼續加油！🎉
