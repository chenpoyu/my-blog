---
layout: post
title: "Kubernetes 進階：ConfigMap、Secret 和 Volume"
date: 2019-01-07 10:30:00 +0800
categories: [DevOps, Kubernetes]
tags: [Kubernetes, ConfigMap, Secret, Volume, Storage]
---

上週學會了 Kubernetes 的基本概念（參考 [Kubernetes 容器編排入門](/posts/2018/12/31/kubernetes-container-orchestration-intro/)），成功部署了第一個應用。但遇到一個問題：如何管理應用的設定檔和敏感資訊？

這週深入研究 ConfigMap、Secret 和 Volume，學習如何在 Kubernetes 中管理設定和資料。

## 問題場景

我們的 Spring Boot 應用有以下需求：

1. **應用設定**：`application.properties` 在不同環境有不同的值
2. **敏感資訊**：資料庫密碼、API key 不能寫在程式碼或 YAML 裡
3. **日誌儲存**：容器重啟後日誌會消失
4. **資料持久化**：資料庫資料不能因為 Pod 重啟而遺失

直接把設定寫在 Docker image 裡是很糟的做法：
- 每次改設定都要重新 build image
- 密碼會被 commit 到 Git
- Dev、Staging、Prod 環境要維護不同的 image

## ConfigMap - 管理設定

ConfigMap 用來儲存非敏感的設定資料。

### 從字面量建立

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=DB_POOL_SIZE=20 \
  --from-literal=CACHE_ENABLED=true
```

查看：
```bash
kubectl get configmap app-config -o yaml

# apiVersion: v1
# kind: ConfigMap
# metadata:
#   name: app-config
# data:
#   LOG_LEVEL: INFO
#   DB_POOL_SIZE: "20"
#   CACHE_ENABLED: "true"
```

### 從檔案建立

有個 `application.properties`：
```properties
server.port=8080
logging.level.root=INFO
logging.level.com.mycompany=DEBUG
spring.jpa.show-sql=false
spring.jpa.hibernate.ddl-auto=validate
```

建立 ConfigMap：
```bash
kubectl create configmap app-properties \
  --from-file=application.properties
```

### 從 YAML 建立

`configmap.yaml`：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-shop-config
data:
  # 單一值
  log.level: "INFO"
  cache.ttl: "3600"
  
  # 整個檔案內容
  application.properties: |
    server.port=8080
    logging.level.root=INFO
    spring.datasource.hikari.maximum-pool-size=20
    
  logback.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="info">
            <appender-ref ref="CONSOLE" />
        </root>
    </configuration>
```

部署：
```bash
kubectl apply -f configmap.yaml
```

### 在 Pod 中使用 ConfigMap

#### 方式一：作為環境變數

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-shop-api:1.0.0
    env:
    # 單一值
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    
    # 全部值
    envFrom:
    - configMapRef:
        name: app-config
```

這樣在容器內就能用 `System.getenv("LOG_LEVEL")` 讀取。

#### 方式二：掛載為檔案

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-shop-api:1.0.0
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
      
  volumes:
  - name: config-volume
    configMap:
      name: my-shop-config
```

容器內會有：
- `/app/config/application.properties`
- `/app/config/logback.xml`

Spring Boot 會自動讀取 `/app/config/application.properties`。

#### 方式三：掛載特定檔案

只掛載 ConfigMap 中的特定項目：

```yaml
volumes:
- name: config-volume
  configMap:
    name: my-shop-config
    items:
    - key: application.properties
      path: application.properties
```

### ConfigMap 熱更新

修改 ConfigMap 後，掛載為 volume 的檔案會自動更新（約 1 分鐘延遲）。

```bash
# 修改 ConfigMap
kubectl edit configmap my-shop-config

# 或使用 kubectl replace
kubectl create configmap my-shop-config --from-file=application.properties -o yaml --dry-run | kubectl replace -f -
```

但環境變數不會自動更新，需要重啟 Pod：
```bash
kubectl rollout restart deployment my-shop-api
```

## Secret - 管理敏感資訊

Secret 用來儲存密碼、token、SSH key 等敏感資訊。

### 建立 Secret

```bash
kubectl create secret generic mysql-secret \
  --from-literal=username=shopuser \
  --from-literal=password=MySecureP@ssw0rd
```

查看：
```bash
kubectl get secret mysql-secret -o yaml

# apiVersion: v1
# kind: Secret
# metadata:
#   name: mysql-secret
# type: Opaque
# data:
#   username: c2hvcHVzZXI=
#   password: TXlTZWN1cmVQQHNzdzByZA==
```

注意：Secret 的值是 base64 編碼，**不是加密**！

解碼：
```bash
echo "c2hvcHVzZXI=" | base64 -d
# shopuser
```

### 從檔案建立 Secret

```bash
# SSH key
kubectl create secret generic ssh-key-secret \
  --from-file=ssh-privatekey=~/.ssh/id_rsa \
  --from-file=ssh-publickey=~/.ssh/id_rsa.pub

# TLS 憑證
kubectl create secret tls tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key
```

### 使用 YAML 建立 Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  # 要先 base64 編碼
  username: c2hvcHVzZXI=
  password: TXlTZWN1cmVQQHNzdzByZA==
```

也可以用 `stringData`（不需要編碼）：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  username: shopuser
  password: MySecureP@ssw0rd
```

**注意**：不要把 Secret YAML 提交到 Git！

### 在 Pod 中使用 Secret

#### 作為環境變數

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-shop-api:1.0.0
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: password
```

#### 掛載為檔案

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-shop-api:1.0.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
      
  volumes:
  - name: secret-volume
    secret:
      secretName: mysql-secret
```

容器內會有：
- `/etc/secrets/username`
- `/etc/secrets/password`

讀取：
```bash
cat /etc/secrets/username
# shopuser
```

### Docker Registry Secret

從私有 Registry 拉取映像檔需要認證：

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.mycompany.com \
  --docker-username=developer \
  --docker-password=dev123456 \
  --docker-email=developer@mycompany.com
```

使用：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: my-app
    image: registry.mycompany.com/myshop/my-shop-api:1.0.0
```

## Volume - 資料持久化

預設情況下，容器的檔案系統是暫時的，Pod 重啟後資料會消失。

### emptyDir - 暫時儲存

Pod 內的容器共享儲存，Pod 刪除後資料也刪除。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: my-shop-api:1.0.0
    volumeMounts:
    - name: cache-volume
      mountPath: /app/cache
      
  - name: log-collector
    image: fluentd:1.3
    volumeMounts:
    - name: cache-volume
      mountPath: /var/log/app
      
  volumes:
  - name: cache-volume
    emptyDir: {}
```

兩個容器可以透過 `/app/cache` 和 `/var/log/app` 共享檔案。

### hostPath - 掛載主機目錄

將 Node 的目錄掛載到容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-shop-api:1.0.0
    volumeMounts:
    - name: log-volume
      mountPath: /app/logs
      
  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/my-app
      type: DirectoryOrCreate
```

注意：Pod 被排程到不同 Node 時，會看到不同的目錄。

### PersistentVolume (PV) 和 PersistentVolumeClaim (PVC)

#### 概念

- **PersistentVolume (PV)**：實際的儲存資源（管理員建立）
- **PersistentVolumeClaim (PVC)**：對儲存的請求（開發者建立）

類比：PV 是硬碟，PVC 是申請單。

#### 建立 PV

`pv.yaml`：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce  # 單一 Node 讀寫
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/mysql
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
# NAME       CAPACITY   ACCESS MODES   STATUS      CLAIM   
# mysql-pv   10Gi       RWO            Available
```

#### 建立 PVC

`pvc.yaml`：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
# NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES
# mysql-pvc   Bound    mysql-pv   10Gi       RWO
```

PVC 會自動綁定到符合條件的 PV。

#### 在 Pod 中使用 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.7
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
      
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-pvc
```

現在 MySQL 的資料會持久化，Pod 重啟後資料還在！

### AccessMode

- **ReadWriteOnce (RWO)**：單一 Node 可讀寫
- **ReadOnlyMany (ROX)**：多個 Node 唯讀
- **ReadWriteMany (RWX)**：多個 Node 可讀寫

本機的 hostPath 只支援 RWO。RWX 需要網路儲存（NFS、GlusterFS）。

### StorageClass - 動態配置

不想手動建立 PV？使用 StorageClass 自動配置。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard  # 使用預設的 StorageClass
  resources:
    requests:
      storage: 10Gi
```

Kubernetes 會自動建立 PV 並綁定。

查看可用的 StorageClass：
```bash
kubectl get storageclass

# NAME                 PROVISIONER
# standard (default)   k8s.io/minikube-hostpath
```

## 完整範例：部署 MySQL

結合 ConfigMap、Secret、PVC：

`mysql-deployment.yaml`：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci
    max_connections=200
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  root-password: MyRootP@ss123
  user: shopuser
  password: ShopP@ss456
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: myshop
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d
          
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
      - name: mysql-config
        configMap:
          name: mysql-config
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  clusterIP: None  # Headless Service
```

部署：
```bash
kubectl apply -f mysql-deployment.yaml
```

測試連接：
```bash
kubectl run -it --rm mysql-client --image=mysql:5.7 --restart=Never -- \
  mysql -h mysql-service -u shopuser -pShopP@ss456 myshop
```

## 遇到的問題

### 問題一：ConfigMap 更新但 Pod 沒重載

修改了 ConfigMap，但應用還是用舊設定。

原因：
- 環境變數不會自動更新
- Volume 雖然會更新，但應用不會重新讀取檔案

解決方法：
```bash
# 重啟 Deployment
kubectl rollout restart deployment my-shop-api
```

或使用 ConfigMap 的 checksum 觸發更新：
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

### 問題二：Secret 在 Git 歷史中洩漏

不小心把包含 Secret 的 YAML commit 到 Git。

解決方法：
1. 使用 `.gitignore` 排除 secret.yaml
2. 使用外部密鑰管理（Sealed Secrets、Vault）
3. Git history 清理（但已經 push 的很難完全刪除）

**最佳實踐**：用環境變數或 CI/CD 工具注入 Secret，不要放在 Git。

### 問題三：PVC 刪除後資料消失

刪除 PVC 時，PV 也被刪除了。

原因：PV 的 `persistentVolumeReclaimPolicy` 設為 `Delete`。

解決方法：設為 `Retain`
```yaml
spec:
  persistentVolumeReclaimPolicy: Retain
```

這樣 PVC 刪除後，PV 會保留，資料不會遺失。

## 心得

ConfigMap、Secret、Volume 是 Kubernetes 中管理設定和資料的核心機制。

以前用 Docker Compose 時，設定都寫在 `.env` 檔案或 `docker-compose.yml`，比較直覺。Kubernetes 的方式雖然複雜一點，但更靈活：
- 同一個 Deployment 可以在不同環境使用不同的 ConfigMap
- Secret 和應用程式完全分離
- 資料持久化有多種選擇

下週要研究 Kubernetes 的 Ingress，實現更靈活的路由和負載平衡。
