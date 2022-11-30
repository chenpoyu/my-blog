---
layout: post
title: "備份與災難恢復"
date: 2019-05-13 10:00:00 +0800
categories: [DevOps, Reliability]
tags: [Backup, Disaster Recovery, RTO, RPO, Velero]
---

上週完成了資料庫遷移（參考 [資料庫遷移策略](/posts/2019/05/06/database-migration-strategy/)），這週來談一個重要但常被忽視的主題：備份與災難恢復 (Disaster Recovery, DR)。

為什麼重要？上個月的慘痛教訓：

週五晚上，工程師誤執行了 `kubectl delete namespace production`，整個生產環境被刪除。我們的備份策略？幾乎沒有。

結果：
- 停機 8 小時
- 遺失 4 小時的資料
- 客戶投訴爆炸
- 損失慘重

這次事件讓我們認真看待備份和 DR。

> 使用工具：Velero、pgBackRest、Restic、AWS S3

## 核心概念

### RTO (Recovery Time Objective)

系統可容忍的最長停機時間。

例如：
- 電商網站：RTO = 1 小時（超過 1 小時損失太大）
- 內部工具：RTO = 24 小時（可接受）

### RPO (Recovery Point Objective)

可容忍的最大資料遺失時間。

例如：
- 金融系統：RPO = 0（不能遺失任何資料）
- 日誌系統：RPO = 1 小時（可接受）

RTO 和 RPO 決定了備份策略和成本：

| RTO | RPO | 策略 | 成本 |
|-----|-----|------|------|
| 24小時 | 24小時 | 每日備份 | 低 |
| 4小時 | 1小時 | 每小時備份 + 快速恢復 | 中 |
| 1小時 | 15分鐘 | 持續複寫 + 自動 failover | 高 |
| < 5分鐘 | 0 | 多區域同步複寫 | 非常高 |

我們的目標：
- 應用：RTO = 2 小時，RPO = 30 分鐘
- 資料庫：RTO = 1 小時，RPO = 5 分鐘

## Kubernetes 備份（Velero）

Velero 是 CNCF 專案，用來備份 Kubernetes 資源和 Persistent Volumes。

### 安裝 Velero

安裝 CLI：
```bash
# macOS
brew install velero

# Linux
wget https://github.com/vmware-tanzu/velero/releases/download/v1.0.0/velero-v1.0.0-linux-amd64.tar.gz
tar -xvf velero-v1.0.0-linux-amd64.tar.gz
sudo mv velero-v1.0.0-linux-amd64/velero /usr/local/bin/
```

安裝到 Kubernetes（使用 AWS S3）：
```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.0.0 \
  --bucket my-velero-backup \
  --backup-location-config region=ap-southeast-1 \
  --snapshot-location-config region=ap-southeast-1 \
  --secret-file ./credentials-velero
```

`credentials-velero`：
```ini
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

確認安裝：
```bash
kubectl get pods -n velero
# NAME                      READY   STATUS    RESTARTS   AGE
# velero-7d9c8b5c4d-x7k2m   1/1     Running   0          1m
```

### 手動備份

備份整個 namespace：
```bash
velero backup create production-backup --include-namespaces production
```

查看備份狀態：
```bash
velero backup describe production-backup

# 輸出
Name:         production-backup
Namespace:    velero
Status:       Completed
Started:      2019-05-13 10:00:00 +0800 CST
Completed:    2019-05-13 10:05:00 +0800 CST
Expiration:   2019-06-12 10:00:00 +0800 CST
Resource List:
  - deployments
  - services
  - configmaps
  - secrets
  - persistentvolumeclaims
  - persistentvolumes
Persistent Volumes: 5/5 backed up successfully
```

備份特定資源：
```bash
# 只備份 Deployment
velero backup create deployments-backup \
  --include-namespaces production \
  --include-resources deployments

# 備份有特定 label 的資源
velero backup create app-backup \
  --selector app=order-service
```

### 恢復

恢復整個備份：
```bash
velero restore create --from-backup production-backup
```

查看恢復狀態：
```bash
velero restore describe production-backup-20190513100500

# 驗證 Pod 是否恢復
kubectl get pods -n production
```

恢復到不同 namespace：
```bash
velero restore create --from-backup production-backup \
  --namespace-mappings production:production-restore
```

### 定期備份

建立排程備份（每 6 小時）：
```bash
velero schedule create production-schedule \
  --schedule="0 */6 * * *" \
  --include-namespaces production \
  --ttl 720h  # 保留 30 天
```

查看排程：
```bash
velero schedule get

# NAME                  STATUS    CREATED                         SCHEDULE    BACKUP TTL   LAST BACKUP
# production-schedule   Enabled   2019-05-13 10:00:00 +0800 CST   0 */6 * * * 720h0m0s     2m ago
```

查看自動建立的備份：
```bash
velero backup get

# NAME                              STATUS      CREATED                         EXPIRES
# production-schedule-20190513100000 Completed   2019-05-13 10:00:00 +0800 CST   29d
# production-schedule-20190513160000 Completed   2019-05-13 16:00:00 +0800 CST   29d
```

## 資料庫備份

### PostgreSQL 備份（pgBackRest）

pgBackRest 是企業級 PostgreSQL 備份工具。

安裝：
```bash
# Ubuntu
sudo apt-get install pgbackrest

# 配置
sudo vi /etc/pgbackrest.conf
```

`/etc/pgbackrest.conf`：
```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=7
repo1-retention-diff=7

# S3 儲存
repo1-type=s3
repo1-s3-bucket=my-pg-backups
repo1-s3-region=ap-southeast-1
repo1-s3-key=AKIAIOSFODNN7EXAMPLE
repo1-s3-key-secret=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[main]
pg1-path=/var/lib/postgresql/11/main
pg1-port=5432
```

PostgreSQL 配置：
```ini
# /etc/postgresql/11/main/postgresql.conf
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

初始化：
```bash
# 建立 stanza
sudo -u postgres pgbackrest --stanza=main stanza-create

# 驗證
sudo -u postgres pgbackrest --stanza=main check
```

### 全量備份

```bash
sudo -u postgres pgbackrest --stanza=main backup --type=full
```

時間：視資料量而定（100GB 約 30 分鐘）。

### 增量備份

```bash
# 差異備份（基於最後一次全量備份）
sudo -u postgres pgbackrest --stanza=main backup --type=diff

# 增量備份（基於最後一次任何備份）
sudo -u postgres pgbackrest --stanza=main backup --type=incr
```

增量備份很快（1-5 分鐘），因為只備份變更的資料。

### 定期備份

使用 cron：
```bash
# /etc/cron.d/pgbackrest
# 每天 2:00 AM 全量備份
0 2 * * * postgres pgbackrest --stanza=main backup --type=full

# 每小時增量備份
0 * * * * postgres pgbackrest --stanza=main backup --type=incr
```

### 恢復資料庫

完整恢復：
```bash
# 停止 PostgreSQL
sudo systemctl stop postgresql

# 恢復
sudo -u postgres pgbackrest --stanza=main restore

# 啟動 PostgreSQL
sudo systemctl start postgresql
```

時間點恢復 (PITR)：
```bash
# 恢復到特定時間點
sudo -u postgres pgbackrest --stanza=main restore \
  --type=time \
  --target="2019-05-13 14:30:00"
```

例如：誤刪資料在 14:35，恢復到 14:30。

### 驗證備份

定期測試恢復：
```bash
# 恢復到測試環境
sudo -u postgres pgbackrest --stanza=main restore \
  --pg1-path=/var/lib/postgresql/test \
  --log-level-console=info

# 啟動測試 instance
pg_ctl -D /var/lib/postgresql/test start

# 驗證資料
psql -h localhost -p 5433 -c "SELECT COUNT(*) FROM orders"
```

**重要**：如果沒測試過恢復,備份形同虛設！

## 應用程式資料備份

### 檔案備份（Restic）

Restic 是現代化的備份工具，支援加密和去重。

安裝：
```bash
# macOS
brew install restic

# Linux
sudo apt-get install restic
```

初始化 repository：
```bash
# 使用 S3
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

restic -r s3:s3.amazonaws.com/my-restic-backup init
# 輸入密碼（用來加密備份）
```

備份：
```bash
# 備份整個目錄
restic -r s3:s3.amazonaws.com/my-restic-backup backup /data

# 輸出
Files:        12345 new,     0 changed,     0 unmodified
Dirs:          456 new,     0 changed,     0 unmodified
Added to the repo: 5.123 GiB
```

查看 snapshots：
```bash
restic -r s3:s3.amazonaws.com/my-restic-backup snapshots

# ID        Time                 Host        Tags        Paths
# -----------------------------------------------------------------------
# abc12345  2019-05-13 10:00:00  server-1                /data
# def67890  2019-05-13 16:00:00  server-1                /data
```

恢復：
```bash
# 恢復最新 snapshot
restic -r s3:s3.amazonaws.com/my-restic-backup restore latest --target /restore

# 恢復特定 snapshot
restic -r s3:s3.amazonaws.com/my-restic-backup restore abc12345 --target /restore

# 只恢復特定檔案
restic -r s3:s3.amazonaws.com/my-restic-backup restore latest \
  --target /restore \
  --include /data/orders/*
```

### 自動化備份

`backup.sh`：
```bash
#!/bin/bash
set -e

REPO="s3:s3.amazonaws.com/my-restic-backup"
SOURCE="/data"
RETENTION_DAYS=30

# 備份
restic -r $REPO backup $SOURCE \
  --exclude-file=/etc/restic/excludes.txt

# 清理舊備份（保留 30 天）
restic -r $REPO forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --prune

# 驗證 repository 完整性
restic -r $REPO check

echo "Backup completed at $(date)"
```

`/etc/restic/excludes.txt`：
```
*.tmp
*.log
*.cache
```

定期執行：
```bash
# /etc/cron.d/restic-backup
0 3 * * * root /usr/local/bin/backup.sh >> /var/log/restic-backup.log 2>&1
```

## 災難恢復演練

定期演練 DR 流程，確保能快速恢復。

### DR 演練計畫

**演練頻率**：每季一次

**演練場景**：
1. 單一 Pod 故障（RTO: 5 分鐘）
2. 整個 Deployment 被刪除（RTO: 30 分鐘）
3. Namespace 被刪除（RTO: 2 小時）
4. 資料庫損毀（RTO: 1 小時）
5. 整個 Kubernetes 叢集故障（RTO: 4 小時）
6. 整個區域故障（RTO: 8 小時）

### 演練流程

**場景 3：Namespace 被刪除**

1. **準備階段（10 分鐘）**：
```bash
# 記錄當前狀態
kubectl get all -n production > before-disaster.txt
kubectl get pvc -n production >> before-disaster.txt

# 確認最新備份
velero backup get | grep production
```

2. **模擬災難（1 分鐘）**：
```bash
# 刪除 namespace
kubectl delete namespace production

# 驗證刪除
kubectl get namespace production
# Error: namespace "production" not found
```

3. **災難響應（5 分鐘）**：
```bash
# 發送告警
curl -X POST https://slack.com/api/chat.postMessage \
  -d "text=🚨 DISASTER: Production namespace deleted!"

# 召集團隊
# 啟動 DR 流程
```

4. **執行恢復（60 分鐘）**：
```bash
# 找到最新的備份
BACKUP=$(velero backup get | grep production | head -1 | awk '{print $1}')

# 恢復
velero restore create production-restore-$(date +%Y%m%d-%H%M%S) \
  --from-backup $BACKUP

# 監控恢復進度
watch kubectl get pods -n production
```

5. **驗證恢復（15 分鐘）**：
```bash
# 驗證 Pod 數量
kubectl get pods -n production | wc -l
# 應該與 before-disaster.txt 相同

# 驗證服務可用
curl http://api.example.com/health
# {"status": "UP"}

# 驗證資料
curl http://api.example.com/orders/12345
# 應該回傳正確資料

# 驗證 PVC
kubectl get pvc -n production
# 所有 PVC 應該都是 Bound 狀態
```

6. **記錄結果（10 分鐘）**：
```
災難恢復演練報告
================
日期：2019-05-13
場景：Namespace 被刪除
目標 RTO：2 小時
實際 RTO：1 小時 20 分鐘 ✅

時間線：
10:00 - 模擬災難（刪除 namespace）
10:05 - 發現問題並啟動 DR
10:10 - 開始恢復
11:05 - 恢復完成
11:20 - 驗證完成

遇到的問題：
1. 第一次恢復失敗（PVC 未正確恢復）
   解決：手動建立 PVC，然後重新恢復

改進建議：
1. 優化 Velero 配置，確保 PVC 正確恢復
2. 建立自動化驗證腳本
3. 文件化恢復流程
```

## 多區域備份

將備份儲存在多個區域，避免區域故障。

### S3 跨區域複寫

```bash
# 建立複寫規則
aws s3api put-bucket-replication \
  --bucket my-velero-backup \
  --replication-configuration file://replication.json
```

`replication.json`：
```json
{
  "Role": "arn:aws:iam::123456789012:role/s3-replication-role",
  "Rules": [
    {
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::my-velero-backup-replica",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {
            "Minutes": 15
          }
        }
      }
    }
  ]
}
```

備份會自動複寫到另一個區域（15 分鐘內）。

### 多叢集架構

主區域故障時,切換到備援區域：

```
主區域 (ap-southeast-1)
├─ K8s Cluster A (主)
├─ PostgreSQL Primary
└─ 備份 → S3 (ap-southeast-1)
         └─ 複寫 → S3 (ap-northeast-1)

備援區域 (ap-northeast-1)
├─ K8s Cluster B (待命)
└─ PostgreSQL Standby (串流複寫)
```

切換流程：
1. 檢測主區域故障
2. 提升備援 PostgreSQL 為主
3. 從 S3 恢復 Kubernetes 資源到 Cluster B
4. 更新 DNS（Route53）指向 Cluster B
5. 驗證服務

RTO：30 分鐘，RPO：0（即時複寫）

## 備份監控

監控備份狀態，確保及時發現問題。

### Prometheus 監控

Velero 提供 Prometheus metrics：
```bash
# 啟用 metrics
kubectl patch deployment velero -n velero \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--metrics-address=:8085"}]'
```

監控指標：
```promql
# 最後一次備份距今時間
time() - velero_backup_last_successful_timestamp{schedule="production-schedule"}

# 備份失敗次數
velero_backup_failure_total{schedule="production-schedule"}

# 備份大小
velero_backup_tarball_size_bytes{schedule="production-schedule"}
```

告警規則：
```yaml
groups:
- name: velero
  rules:
  - alert: BackupFailed
    expr: velero_backup_failure_total > 0
    for: 5m
    annotations:
      summary: "Velero backup failed"
      description: "Backup {{ $labels.schedule }} has failed"
  
  - alert: BackupTooOld
    expr: (time() - velero_backup_last_successful_timestamp) > 21600
    annotations:
      summary: "Backup is too old"
      description: "Last successful backup was {{ $value }}s ago (> 6 hours)"
```

### 備份驗證

定期驗證備份完整性：

```bash
#!/bin/bash
# verify-backup.sh

# 恢復到測試環境
velero restore create test-restore-$(date +%Y%m%d-%H%M%S) \
  --from-backup production-schedule-latest \
  --namespace-mappings production:test-restore

# 等待恢復完成
while true; do
  STATUS=$(velero restore describe test-restore-latest --details | grep "Phase:" | awk '{print $2}')
  if [ "$STATUS" = "Completed" ]; then
    break
  fi
  sleep 10
done

# 驗證 Pod 數量
EXPECTED_PODS=10
ACTUAL_PODS=$(kubectl get pods -n test-restore | grep Running | wc -l)

if [ $ACTUAL_PODS -eq $EXPECTED_PODS ]; then
  echo "Backup verification passed"
  # 清理
  kubectl delete namespace test-restore
  exit 0
else
  echo "Backup verification failed: expected $EXPECTED_PODS pods, got $ACTUAL_PODS"
  exit 1
fi
```

每天自動執行：
```bash
# /etc/cron.d/verify-backup
0 4 * * * root /usr/local/bin/verify-backup.sh >> /var/log/verify-backup.log 2>&1
```

## 心得

這次誤刪 namespace 的事件是個慘痛教訓，但也讓我們建立了完善的備份和 DR 機制。

Velero 真的很好用，安裝簡單，自動化備份，恢復快速。現在每 6 小時自動備份，保留 30 天，很安心。

定期演練很重要！第一次演練時,發現很多問題：
- Velero 配置不正確（PVC 沒有備份）
- 沒有文件化恢復流程（大家不知道怎麼做）
- 驗證不完整（只看 Pod 數量，沒驗證資料）

經過幾次演練後，現在團隊都知道怎麼應對災難，RTO 從 8 小時降到 1.5 小時。

建議：
1. **自動化備份**：手動備份會忘記
2. **定期演練**：沒演練過的 DR 計畫等於沒有
3. **監控備份**：備份失敗要立即告警
4. **多區域儲存**：單一區域故障就完了
5. **驗證恢復**：定期測試恢復流程

下週要研究效能測試，確保系統能承受預期的負載。
