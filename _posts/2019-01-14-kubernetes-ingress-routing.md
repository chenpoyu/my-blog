---
layout: post
title: "Kubernetes Ingress 路由和負載平衡"
date: 2019-01-14 11:00:00 +0800
categories: [DevOps, Kubernetes]
tags: [Kubernetes, Ingress, Nginx, Load Balancer, Routing]
---

前幾週學會了在 Kubernetes 部署應用（參考 [Kubernetes 進階](/posts/2019/01/07/kubernetes-configmap-secret-volume/)），但每個 Service 都要用不同的 port 訪問，很不方便。

例如：
- 商品服務：`http://my-k8s-cluster:30001`
- 訂單服務：`http://my-k8s-cluster:30002`
- 使用者服務：`http://my-k8s-cluster:30003`

理想的方式應該是：
- `http://api.mycompany.com/products` → 商品服務
- `http://api.mycompany.com/orders` → 訂單服務
- `http://api.mycompany.com/users` → 使用者服務

這週研究 Kubernetes Ingress，實現基於路徑和域名的路由。

> 使用版本：Nginx Ingress Controller 0.21.0

## Service 的限制

Kubernetes Service 有幾種類型：

### ClusterIP（預設）
只能在叢集內部訪問，外部無法連接。

### NodePort
透過 Node 的 port 對外開放：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30001  # 30000-32767
```

問題：
- Port 範圍有限（30000-32767）
- 每個服務佔用一個 port
- 無法做基於路徑的路由
- 沒有 SSL/TLS 終止

### LoadBalancer
使用雲端提供的負載平衡器（AWS ELB、GCP Load Balancer）。

問題：
- 每個 Service 都會建立一個 Load Balancer（貴！）
- 本機環境（Minikube）無法使用

## Ingress 是什麼

Ingress 是 Kubernetes 的 API 物件，用來管理外部訪問叢集內服務的規則。

特色：
- **基於路徑路由**：`/api/products` → 商品服務
- **基於域名路由**：`api.mycompany.com` → API 服務
- **SSL/TLS 終止**：統一處理 HTTPS
- **單一入口點**：一個 IP/域名對應多個服務

Ingress 本身只是規則定義，需要 **Ingress Controller** 來實際執行。

常見的 Ingress Controller：
- **Nginx Ingress Controller**（最流行）
- Traefik
- HAProxy
- Kong

我們使用 Nginx Ingress Controller。

## 安裝 Nginx Ingress Controller

### Minikube 安裝

```bash
minikube addons enable ingress

# 驗證
kubectl get pods -n kube-system | grep ingress
# nginx-ingress-controller-xxxxx   1/1     Running
```

超簡單！

### 一般 Kubernetes 叢集安裝

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.21.0/deploy/mandatory.yaml

# 建立 LoadBalancer Service
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.21.0/deploy/provider/cloud-generic.yaml
```

檢查狀態：
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## 第一個 Ingress

### 準備服務

先部署兩個簡單的服務：

`app1.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 80
```

`app2.yaml`：類似 app1，改名為 app2。

部署：
```bash
kubectl apply -f app1.yaml
kubectl apply -f app2.yaml
```

### 建立 Ingress

`ingress.yaml`：
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /app1
        backend:
          serviceName: app1-service
          servicePort: 80
      - path: /app2
        backend:
          serviceName: app2-service
          servicePort: 80
```

部署：
```bash
kubectl apply -f ingress.yaml
```

查看：
```bash
kubectl get ingress
# NAME         HOSTS   ADDRESS        PORTS   AGE
# my-ingress   *       192.168.64.5   80      1m
```

### 測試

```bash
# Minikube
minikube ip
# 192.168.64.5

curl http://192.168.64.5/app1
# App1 的回應

curl http://192.168.64.5/app2
# App2 的回應
```

成功！同一個 IP，不同路徑路由到不同服務。

## 基於域名的路由

單一 IP 服務多個域名。

`ingress-host.yaml`：
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: app1-service
          servicePort: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: app2-service
          servicePort: 80
```

測試（修改本機 hosts）：
```bash
# macOS/Linux
sudo nano /etc/hosts

# 加入
192.168.64.5 app1.example.com
192.168.64.5 app2.example.com
```

測試：
```bash
curl http://app1.example.com
# App1

curl http://app2.example.com
# App2
```

## 實際專案：Spring Boot 微服務

我們有三個服務：
- **產品服務**：`/api/products`
- **訂單服務**：`/api/orders`
- **使用者服務**：`/api/users`

### 部署服務

`products-deployment.yaml`（其他類似）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: products-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: products-api
  template:
    metadata:
      labels:
        app: products-api
    spec:
      containers:
      - name: products-api
        image: registry.mycompany.com/myshop/products-api:1.0.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: products-service
spec:
  selector:
    app: products-api
  ports:
  - port: 80
    targetPort: 8080
```

### 建立 Ingress

`api-ingress.yaml`：
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /api/products(/|$)(.*)
        backend:
          serviceName: products-service
          servicePort: 80
      - path: /api/orders(/|$)(.*)
        backend:
          serviceName: orders-service
          servicePort: 80
      - path: /api/users(/|$)(.*)
        backend:
          serviceName: users-service
          servicePort: 80
```

`rewrite-target: /$2` 的作用：
- 請求：`/api/products/123`
- 轉發到服務：`/123`（移除 `/api/products` 前綴）

### 測試

```bash
curl http://api.mycompany.com/api/products
curl http://api.mycompany.com/api/orders/123
curl http://api.mycompany.com/api/users/profile
```

## HTTPS/TLS 支援

### 產生自簽名憑證（測試用）

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=api.mycompany.com/O=mycompany"
```

### 建立 TLS Secret

```bash
kubectl create secret tls api-tls-secret \
  --key tls.key \
  --cert tls.crt
```

### 設定 Ingress 使用 TLS

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.mycompany.com
    secretName: api-tls-secret
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /api/products
        backend:
          serviceName: products-service
          servicePort: 80
```

現在訪問 `http://api.mycompany.com` 會自動重定向到 `https://api.mycompany.com`。

### 使用 Let's Encrypt（免費 SSL）

生產環境建議使用 cert-manager 自動管理 Let's Encrypt 憑證。

安裝 cert-manager：
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml
```

建立 ClusterIssuer：
```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@mycompany.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Ingress 加上 annotation：
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

cert-manager 會自動申請憑證並更新 Secret。

## Ingress 進階設定

### 自訂錯誤頁面

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/custom-http-errors: "404,503"
    nginx.ingress.kubernetes.io/default-backend: error-page-service
```

### 速率限制

限制每個 IP 每秒最多 10 個請求：
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
```

### CORS 設定

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://www.mycompany.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
```

### Basic Auth

建立 htpasswd：
```bash
htpasswd -c auth admin
# 輸入密碼
```

建立 Secret：
```bash
kubectl create secret generic basic-auth --from-file=auth
```

Ingress 設定：
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
```

### Whitelist IP

只允許特定 IP 訪問：
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
```

### 自訂 Nginx 設定

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
```

## 監控和除錯

### 查看 Ingress 詳細資訊

```bash
kubectl describe ingress api-ingress
```

### 查看 Nginx Ingress Controller 日誌

```bash
kubectl logs -n ingress-nginx deployment/nginx-ingress-controller -f
```

### 測試 Ingress 路由

```bash
# 直接訪問 Ingress Controller
kubectl get svc -n ingress-nginx

# Port forward
kubectl port-forward -n ingress-nginx svc/ingress-nginx 8080:80

curl -H "Host: api.mycompany.com" http://localhost:8080/api/products
```

### 常見錯誤

**503 Service Temporarily Unavailable**
- 後端服務不存在或沒有健康的 Pod
- Service selector 不正確

**404 Not Found**
- Ingress path 設定錯誤
- rewrite-target 設定錯誤

**502 Bad Gateway**
- 後端服務回應錯誤
- 連接逾時

## 遇到的問題

### 問題一：路徑路由不正確

設定：
```yaml
- path: /api/products
```

請求 `/api/products/123` 轉發到後端變成 `/api/products/123`，但後端只有 `/123`。

解決方法：使用 rewrite-target
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /api/products(/|$)(.*)
```

### 問題二：SSL 憑證過期

Let's Encrypt 憑證 90 天過期，忘記更新。

解決方法：使用 cert-manager 自動更新，或設定監控提醒。

### 問題三：同一個域名有多個 Ingress

兩個 Ingress 都定義了 `api.mycompany.com`，結果衝突。

解決方法：
1. 合併成一個 Ingress
2. 或使用 `nginx.ingress.kubernetes.io/merge-prefix` annotation

## 心得

Ingress 大幅簡化了服務對外暴露的方式。以前每個服務都要開一個 NodePort，現在只要一個 Ingress 就能管理所有路由。

而且 SSL/TLS 終止、速率限制、認證這些功能都可以在 Ingress 層統一處理，不用每個服務都實作一次。

下週要研究 Kubernetes 的監控和日誌收集，使用 Prometheus 和 Grafana 監控叢集狀態。
