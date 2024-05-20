---
layout: post
title: "n8n 正式環境部署與維運"
date: 2024-05-20 11:05:00 +0800
categories: [自動化, 工作流程]
tags: [n8n, Production, DevOps]
---

前面幾週都在測試環境玩 n8n，這週來研究如何部署到正式環境。測試環境用 Docker 單機跑沒問題，但正式環境要考慮的東西更多：資料持久化、高可用性、安全性、監控等等。

## 正式環境的架構考量

測試環境和正式環境的差異：

**資料庫**：測試環境用內建的 SQLite，資料存在 Docker volume。正式環境要用 PostgreSQL 或 MySQL，做好備份。

**Queue Mode**：n8n 可以用 queue 模式，把 workflow 執行分散到多個 worker，提高吞吐量和可靠性。

**反向代理**：用 Nginx 或 Traefik 做 HTTPS、load balancing、rate limiting。

**監控**：整合 Prometheus、Grafana 監控執行狀況。

**備份**：定期備份資料庫和 workflow 設定。

## PostgreSQL 設定

首先準備 PostgreSQL。n8n 需要一個資料庫來存放 workflow、execution history、credentials 等。

建立資料庫：

```sql
CREATE DATABASE n8n;
CREATE USER n8n_user WITH PASSWORD 'strong_password';
GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n_user;
```

## Docker Compose 部署

用 Docker Compose 管理多個容器。建立 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n_user
      POSTGRES_PASSWORD: strong_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-network

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n_user
      - DB_POSTGRESDB_PASSWORD=strong_password
      - N8N_PROTOCOL=https
      - N8N_HOST=n8n.company.com
      - NODE_ENV=production
      - WEBHOOK_URL=https://n8n.company.com/
      - GENERIC_TIMEZONE=Asia/Taipei
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres
    networks:
      - n8n-network

volumes:
  postgres_data:
  n8n_data:

networks:
  n8n-network:
```

重點設定：

- **N8N_ENCRYPTION_KEY**：用來加密 credentials，一定要設定，而且要保密。可以用 `openssl rand -hex 32` 產生。
- **WEBHOOK_URL**：外部系統呼叫 webhook 的網址。
- **GENERIC_TIMEZONE**：時區設定，影響 Schedule Trigger。

啟動：

```bash
export N8N_ENCRYPTION_KEY="你的加密金鑰"
docker-compose up -d
```

## Queue Mode 設定

預設 n8n 是單一 process，所有 workflow 都在同一個 process 執行。如果有個 workflow 卡住，會影響其他 workflow。

Queue mode 把 workflow 執行丟到 queue，由多個 worker 處理，提高穩定性和吞吐量。

需要 Redis 做 queue：

修改 `docker-compose.yml`：

```yaml
services:
  redis:
    image: redis:7-alpine
    networks:
      - n8n-network

  n8n-main:
    image: docker.n8n.io/n8nio/n8n:latest
    environment:
      # ... 前面的設定
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
    depends_on:
      - postgres
      - redis
    networks:
      - n8n-network

  n8n-worker-1:
    image: docker.n8n.io/n8nio/n8n:latest
    command: worker
    environment:
      # ... 相同的資料庫和 Redis 設定
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
    depends_on:
      - redis
    networks:
      - n8n-network

  n8n-worker-2:
    image: docker.n8n.io/n8nio/n8n:latest
    command: worker
    environment:
      # ... 相同設定
    depends_on:
      - redis
    networks:
      - n8n-network
```

這樣架構是：
- **n8n-main**：處理 UI、API、觸發 workflow
- **n8n-worker-1/2**：執行 workflow

可以根據負載調整 worker 數量。

## Nginx 反向代理

用 Nginx 處理 HTTPS 和 WebSocket：

```nginx
upstream n8n {
    server n8n:5678;
}

server {
    listen 443 ssl http2;
    server_name n8n.company.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://n8n;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 80;
    server_name n8n.company.com;
    return 301 https://$server_name$request_uri;
}
```

n8n 的 UI 用到 WebSocket，所以要設定 `Upgrade` header。

## 環境變數管理

正式環境的環境變數不要直接寫在 docker-compose.yml，用 `.env` 檔案：

```bash
# .env
N8N_ENCRYPTION_KEY=你的加密金鑰
DB_PASSWORD=strong_password
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=admin_password
```

在 docker-compose.yml 引用：

```yaml
environment:
  - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
  - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
```

`.env` 檔案不要放到 git，加到 `.gitignore`。

## 備份策略

**資料庫備份**：

用 cron job 定期備份 PostgreSQL：

```bash
#!/bin/bash
# backup-n8n.sh

BACKUP_DIR="/backup/n8n"
DATE=$(date +%Y%m%d_%H%M%S)
FILENAME="n8n_backup_${DATE}.sql"

docker exec postgres pg_dump -U n8n_user n8n > "${BACKUP_DIR}/${FILENAME}"

# 壓縮
gzip "${BACKUP_DIR}/${FILENAME}"

# 保留最近 30 天的備份
find ${BACKUP_DIR} -name "n8n_backup_*.sql.gz" -mtime +30 -delete
```

加到 crontab：

```bash
0 2 * * * /path/to/backup-n8n.sh
```

**Workflow 匯出**：

雖然 workflow 存在資料庫，但也可以定期匯出成 JSON 放到 Git：

用 n8n API 匯出所有 workflow：

```bash
#!/bin/bash
API_KEY="你的API金鑰"
N8N_URL="https://n8n.company.com"

curl -H "X-N8N-API-KEY: ${API_KEY}" \
     "${N8N_URL}/api/v1/workflows" | \
     jq '.' > workflows_backup.json

git add workflows_backup.json
git commit -m "Backup workflows $(date)"
git push
```

## 監控整合

用 Prometheus 監控 n8n。

n8n 沒有內建 Prometheus exporter，但可以透過 API 抓 metrics：

建立一個自訂的 exporter：

```python
# n8n_exporter.py
from prometheus_client import start_http_server, Gauge
import requests
import time

N8N_URL = "https://n8n.company.com"
API_KEY = "你的API金鑰"

active_workflows = Gauge('n8n_active_workflows', 'Number of active workflows')
failed_executions = Gauge('n8n_failed_executions', 'Number of failed executions in last hour')

def collect_metrics():
    headers = {"X-N8N-API-KEY": API_KEY}
    
    # 查詢 workflow 數量
    workflows = requests.get(f"{N8N_URL}/api/v1/workflows", headers=headers).json()
    active = len([w for w in workflows['data'] if w['active']])
    active_workflows.set(active)
    
    # 查詢失敗的 execution
    executions = requests.get(f"{N8N_URL}/api/v1/executions", headers=headers).json()
    failed = len([e for e in executions['data'] if not e['finished']])
    failed_executions.set(failed)

if __name__ == '__main__':
    start_http_server(8000)
    while True:
        collect_metrics()
        time.sleep(60)
```

在 Prometheus 設定檔加上：

```yaml
scrape_configs:
  - job_name: 'n8n'
    static_configs:
      - targets: ['n8n-exporter:8000']
```

然後在 Grafana 建立 dashboard 顯示 metrics。

## 日誌管理

n8n 的日誌預設輸出到 stdout，用 Docker 的日誌機制收集：

```yaml
# docker-compose.yml
services:
  n8n:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

如果要整合到 ELK 或其他日誌系統，用 Fluentd 或 Filebeat 收集。

## 安全性加固

**Basic Auth**：

如果 n8n 只有內部使用，可以加上 Basic Auth：

```yaml
environment:
  - N8N_BASIC_AUTH_ACTIVE=true
  - N8N_BASIC_AUTH_USER=admin
  - N8N_BASIC_AUTH_PASSWORD=strong_password
```

**防火牆**：

只允許特定 IP 存取管理介面，webhook 可以對外開放。

**Rate Limiting**：

在 Nginx 加上 rate limit，防止 abuse：

```nginx
limit_req_zone $binary_remote_addr zone=webhook:10m rate=10r/s;

location /webhook {
    limit_req zone=webhook burst=20 nodelay;
    proxy_pass http://n8n;
}
```

**Credentials 管理**：

n8n 的 credentials 用 N8N_ENCRYPTION_KEY 加密存在資料庫。這個 key 一定要保密，不要放到 git。

可以用 Vault 或 AWS Secrets Manager 管理這些敏感資料。

## 升級策略

n8n 更新蠻頻繁的，要有升級計畫：

1. 在測試環境先升級，測試 workflow 是否正常
2. 備份正式環境的資料庫
3. 用 rolling update 升級（如果是 queue mode）
4. 觀察日誌，確認沒有錯誤
5. 如果有問題，回滾到舊版本

Docker Compose 升級：

```bash
# 拉取新版本
docker-compose pull

# 重啟服務
docker-compose up -d

# 檢查日誌
docker-compose logs -f n8n
```

## 實務心得

正式環境部署比想像中複雜，但做好這些設定後，系統會穩定很多：

**Queue mode 很重要**：單一 process 容易因為某個 workflow 卡住而影響全部。

**監控不能少**：要知道系統的狀態，執行了多少 workflow、失敗率多少。

**備份是保命符**：曾經不小心刪掉 workflow，還好有備份。

**日誌要保留**：出問題時可以回頭看發生什麼事。

**資源要夠**：n8n 執行複雜的 workflow 會用不少記憶體和 CPU，要預留足夠資源。

經過這幾週的研究，n8n 從入門到部署都實際操作過了。接下來就是實際應用到業務流程上，看能自動化哪些工作，提升效率。
