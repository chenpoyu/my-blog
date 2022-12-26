---
layout: post
title: "AWS S3 與 CloudFront：打造高效能 CDN 架構"
date: 2021-01-13 16:55:00 +0800
categories: [AWS, CDN]
tags: [AWS, S3, CloudFront, CDN, Static Content]
---

## 前言

隨著 IoT 專案的發展,我們開始需要儲存和提供大量的靜態資源：
- 裝置韌體更新檔案
- 設定檔案（JSON、XML）
- 裝置照片和縮圖
- 前端靜態資源（Vue.js SPA）
- 使用者手冊 PDF

一開始把這些檔案放在 EC2 的檔案系統，但很快遇到問題：
- 儲存空間有限
- 備份麻煩
- 無法有效快取
- 跨區域存取速度慢

評估後決定採用 **S3 (Simple Storage Service)** 作為物件儲存，搭配 **CloudFront** CDN 加速全球存取。

## S3 核心概念

### Bucket（儲存貯體）
S3 的最上層容器，類似資料夾。每個 bucket 有：
- 全域唯一的名稱
- 所屬區域
- 存取權限設定

### Object（物件）
儲存在 bucket 中的檔案，每個物件包含：
- **Key**：物件的唯一識別（類似檔案路徑）
- **Value**：實際資料（最大 5 TB）
- **Metadata**：描述資料（Content-Type、自訂標籤等）
- **Version ID**：版本控制識別碼（啟用版本控制時）

### Storage Classes（儲存類別）

| 類別 | 說明 | 適用場景 | 成本 |
|------|------|---------|------|
| Standard | 高頻存取 | 經常使用的資料 | 最高 |
| Intelligent-Tiering | 自動分層 | 存取模式不確定 | 中 |
| Standard-IA | 低頻存取 | 備份、舊資料 | 低 |
| Glacier | 歸檔 | 合規、長期保存 | 很低 |
| Glacier Deep Archive | 深度歸檔 | 幾乎不存取 | 最低 |

## 建立 S3 Bucket

### 使用 AWS CLI

```bash
# 建立 bucket
aws s3 mb s3://my-iot-storage --region us-east-1

# 啟用版本控制
aws s3api put-bucket-versioning \
  --bucket my-iot-storage \
  --versioning-configuration Status=Enabled

# 設定生命週期規則（30 天後移到 IA）
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-iot-storage \
  --lifecycle-configuration file://lifecycle.json
```

`lifecycle.json`：

```json
{
  "Rules": [
    {
      "Id": "Move old objects to IA",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionExpirations": [
        {
          "NoncurrentDays": 90
        }
      ]
    }
  ]
}
```

## 上傳檔案

### 使用 AWS CLI

```bash
# 上傳單一檔案
aws s3 cp firmware-v1.2.bin s3://my-iot-storage/firmware/

# 上傳整個目錄
aws s3 cp ./dist s3://my-iot-storage/web/ --recursive

# 同步目錄（只上傳變更的檔案）
aws s3 sync ./dist s3://my-iot-storage/web/

# 設定 Content-Type 和快取控制
aws s3 cp index.html s3://my-iot-storage/web/ \
  --content-type "text/html" \
  --cache-control "max-age=3600" \
  --metadata-directive REPLACE
```

### 使用 Java SDK

```java
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.ObjectMetadata;
import com.amazonaws.services.s3.model.PutObjectRequest;

@Service
public class S3Service {
    
    private final AmazonS3 s3Client;
    private final String bucketName = "my-iot-storage";
    
    public S3Service() {
        this.s3Client = AmazonS3ClientBuilder.standard()
            .withRegion("us-east-1")
            .build();
    }
    
    public String uploadFile(String key, File file, String contentType) {
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentType(contentType);
        metadata.setContentLength(file.length());
        
        // 自訂 metadata
        metadata.addUserMetadata("uploaded-by", "api");
        metadata.addUserMetadata("upload-time", 
            LocalDateTime.now().toString());
        
        PutObjectRequest request = new PutObjectRequest(
            bucketName, key, file)
            .withMetadata(metadata);
        
        s3Client.putObject(request);
        
        // 回傳檔案 URL
        return s3Client.getUrl(bucketName, key).toString();
    }
    
    public void uploadInputStream(String key, InputStream inputStream, 
                                   long contentLength, String contentType) {
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentType(contentType);
        metadata.setContentLength(contentLength);
        
        PutObjectRequest request = new PutObjectRequest(
            bucketName, key, inputStream, metadata);
        
        s3Client.putObject(request);
    }
}
```

### Multipart Upload（大檔案）

對於超過 100 MB 的檔案，建議使用分段上傳：

```java
public void uploadLargeFile(String key, File file) {
    TransferManager transferManager = TransferManagerBuilder.standard()
        .withS3Client(s3Client)
        .build();
    
    try {
        Upload upload = transferManager.upload(bucketName, key, file);
        
        // 等待上傳完成
        upload.waitForCompletion();
        
        System.out.println("Upload completed");
        
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new RuntimeException("Upload interrupted", e);
    } finally {
        transferManager.shutdownNow();
    }
}
```

## 存取權限管理

### Bucket Policy

設定 bucket 層級的權限：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-iot-storage/public/*"
    },
    {
      "Sid": "AllowEC2Access",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/EC2-S3-Access"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-iot-storage/firmware/*"
    }
  ]
}
```

### IAM Policy

應用程式使用的 IAM 角色需要適當權限：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-iot-storage/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::my-iot-storage"
    }
  ]
}
```

### Pre-signed URLs（預簽名 URL）

讓使用者臨時存取私有檔案，不需要 AWS 憑證：

```java
public String generatePresignedUrl(String key, int expirationMinutes) {
    Date expiration = new Date();
    long expTimeMillis = expiration.getTime();
    expTimeMillis += 1000 * 60 * expirationMinutes;
    expiration.setTime(expTimeMillis);
    
    GeneratePresignedUrlRequest request = 
        new GeneratePresignedUrlRequest(bucketName, key)
            .withMethod(HttpMethod.GET)
            .withExpiration(expiration);
    
    URL url = s3Client.generatePresignedUrl(request);
    return url.toString();
}

// 使用範例
String downloadUrl = s3Service.generatePresignedUrl(
    "firmware/v1.2.bin", 60);  // 60 分鐘有效
```

上傳用的預簽名 URL：

```java
public String generatePresignedUploadUrl(String key, int expirationMinutes) {
    Date expiration = new Date();
    expiration.setTime(expiration.getTime() + 1000 * 60 * expirationMinutes);
    
    GeneratePresignedUrlRequest request = 
        new GeneratePresignedUrlRequest(bucketName, key)
            .withMethod(HttpMethod.PUT)
            .withExpiration(expiration);
    
    URL url = s3Client.generatePresignedUrl(request);
    return url.toString();
}
```

前端直接上傳到 S3：

```javascript
async function uploadFile(file) {
    // 向後端請求預簽名 URL
    const response = await fetch('/api/upload-url', {
        method: 'POST',
        body: JSON.stringify({ filename: file.name })
    });
    const { uploadUrl } = await response.json();
    
    // 直接上傳到 S3
    await fetch(uploadUrl, {
        method: 'PUT',
        body: file
    });
    
    console.log('Upload completed');
}
```

## CloudFront CDN 整合

S3 雖然可靠，但存取速度受地理位置影響。CloudFront 是 AWS 的 CDN 服務，可以：
- 快取內容到全球邊緣節點
- 減少延遲
- 降低 S3 請求成本
- 支援 HTTPS
- 自訂網域

### 建立 CloudFront Distribution

1. **Origin 設定**：
   - Origin Domain：`my-iot-storage.s3.amazonaws.com`
   - Origin Path：（選填，如 `/production`）
   - Origin Access Identity：建立新的 OAI（讓 CloudFront 存取私有 bucket）

2. **Default Cache Behavior**：
   - Viewer Protocol Policy：Redirect HTTP to HTTPS
   - Allowed HTTP Methods：GET, HEAD, OPTIONS
   - Cache Policy：使用 CachingOptimized

3. **Distribution 設定**：
   - Price Class：Use All Edge Locations
   - Alternate Domain Names：`cdn.mycompany.com`
   - SSL Certificate：使用 ACM 憑證

### 更新 Bucket Policy

允許 CloudFront OAI 存取：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAI",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity E1234567890ABC"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-iot-storage/*"
    }
  ]
}
```

### 設定自訂網域

1. 在 Route 53 或 DNS 提供商建立 CNAME：
   ```
   cdn.mycompany.com  CNAME  d1234567890.cloudfront.net
   ```

2. 在 ACM（Certificate Manager）申請 SSL 憑證

3. 在 CloudFront 設定中綁定網域和憑證

## 快取策略

### Cache-Control Headers

在上傳檔案時設定：

```bash
# 長期快取（不常變更的資源）
aws s3 cp logo.png s3://my-iot-storage/images/ \
  --cache-control "public, max-age=31536000, immutable"

# 短期快取（經常更新的內容）
aws s3 cp index.html s3://my-iot-storage/web/ \
  --cache-control "public, max-age=3600, must-revalidate"

# 不快取
aws s3 cp api-response.json s3://my-iot-storage/data/ \
  --cache-control "no-cache, no-store, must-revalidate"
```

### Cache Key 設計

使用版本號或雜湊值來管理快取：

```
# 不好的做法（快取更新困難）
/js/app.js

# 好的做法（版本化）
/js/app.v1.2.3.js
/js/app.abc123def.js
```

Webpack 自動產生 hash 檔名：

```javascript
// webpack.config.js
module.exports = {
  output: {
    filename: '[name].[contenthash].js',
    path: path.resolve(__dirname, 'dist')
  }
};
```

### 手動清除快取

```bash
# 建立 invalidation
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/images/*" "/css/*"

# 清除所有快取（不建議，成本高）
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"
```

**注意**：每月前 1000 個 invalidation 路徑免費，超過後每個路徑 $0.005。

## 託管靜態網站

S3 可以直接託管靜態網站：

### 啟用靜態網站託管

```bash
aws s3 website s3://my-iot-storage/ \
  --index-document index.html \
  --error-document error.html
```

### Vue.js SPA 部署

處理 Vue Router 的 history 模式：

在 CloudFront 建立 Custom Error Response：
- HTTP Error Code：403, 404
- Response Page Path：`/index.html`
- HTTP Response Code：200

這樣所有路由都會由 Vue Router 處理。

### 自動化部署腳本

```bash
#!/bin/bash

# 建置 Vue.js 應用
npm run build

# 同步到 S3
aws s3 sync ./dist s3://my-iot-storage/web/ \
  --delete \
  --cache-control "public, max-age=31536000" \
  --exclude "index.html" \
  --exclude "service-worker.js"

# index.html 使用較短的快取時間
aws s3 cp ./dist/index.html s3://my-iot-storage/web/ \
  --cache-control "public, max-age=300, must-revalidate"

# 清除 CloudFront 快取
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/index.html" "/service-worker.js"

echo "Deployment completed"
```

## S3 事件通知

當檔案上傳到 S3 時觸發 Lambda：

```bash
aws s3api put-bucket-notification-configuration \
  --bucket my-iot-storage \
  --notification-configuration file://notification.json
```

`notification.json`：

```json
{
  "LambdaFunctionConfigurations": [
    {
      "Id": "ImageUploadNotification",
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:ProcessImage",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "images/"
            },
            {
              "Name": "suffix",
              "Value": ".jpg"
            }
          ]
        }
      }
    }
  ]
}
```

Lambda 函式範例（產生縮圖）：

```javascript
const AWS = require('aws-sdk');
const sharp = require('sharp');

const s3 = new AWS.S3();

exports.handler = async (event) => {
    const bucket = event.Records[0].s3.bucket.name;
    const key = decodeURIComponent(
        event.Records[0].s3.object.key.replace(/\+/g, ' ')
    );
    
    console.log(`Processing ${bucket}/${key}`);
    
    try {
        // 下載原圖
        const original = await s3.getObject({ Bucket: bucket, Key: key }).promise();
        
        // 產生縮圖
        const thumbnail = await sharp(original.Body)
            .resize(200, 200, { fit: 'cover' })
            .toBuffer();
        
        // 上傳縮圖
        const thumbnailKey = key.replace('images/', 'thumbnails/');
        await s3.putObject({
            Bucket: bucket,
            Key: thumbnailKey,
            Body: thumbnail,
            ContentType: 'image/jpeg'
        }).promise();
        
        console.log(`Thumbnail created: ${thumbnailKey}`);
        
        return { statusCode: 200, body: 'Success' };
        
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
};
```

## 安全性設定

### 封鎖公開存取

預設應該封鎖公開存取，只透過 CloudFront 提供內容：

```bash
aws s3api put-public-access-block \
  --bucket my-iot-storage \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### 啟用伺服器端加密

```bash
aws s3api put-bucket-encryption \
  --bucket my-iot-storage \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

或使用 KMS 管理的金鑰：

```json
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abcd-1234"
    }
  }]
}
```

### 啟用存取日誌

```bash
aws s3api put-bucket-logging \
  --bucket my-iot-storage \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "my-logs-bucket",
      "TargetPrefix": "s3-access-logs/"
    }
  }'
```

## 成本優化

### 實際成本分析（以 us-east-1 為例）

**S3 儲存成本**：
- Standard：$0.023/GB/月
- Standard-IA：$0.0125/GB/月
- Glacier：$0.004/GB/月

**請求成本**：
- PUT/POST：$0.005/1000 請求
- GET：$0.0004/1000 請求

**CloudFront 成本**：
- 資料傳輸（前 10 TB）：$0.085/GB

**範例計算**：

假設每月：
- 儲存 500 GB（Standard）
- 100 萬次 GET 請求
- CloudFront 傳輸 1 TB

```
S3 儲存：500 * $0.023 = $11.50
S3 GET：1000 * $0.0004 = $0.40
CloudFront：1000 * $0.085 = $85.00
總計：$96.90/月
```

### 優化建議

1. **使用 Lifecycle Policy**：自動轉移舊資料到低成本儲存
2. **啟用 Intelligent-Tiering**：讓 AWS 自動優化儲存類別
3. **壓縮檔案**：上傳前壓縮，減少儲存和傳輸成本
4. **設定 CloudFront 快取**：減少回源請求

## 監控與告警

### S3 指標

在 CloudWatch 可以看到：
- BucketSizeBytes：Bucket 大小
- NumberOfObjects：物件數量
- AllRequests：總請求數
- 4xxErrors / 5xxErrors：錯誤率

### CloudFront 指標

- Requests：請求數
- BytesDownloaded：下載流量
- ErrorRate：錯誤率
- CacheHitRate：快取命中率

### 設定告警

```bash
# S3 錯誤率告警
aws cloudwatch put-metric-alarm \
  --alarm-name s3-high-error-rate \
  --alarm-description "S3 4xx error rate above 5%" \
  --metric-name 4xxErrors \
  --namespace AWS/S3 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold

# CloudFront 快取命中率告警
aws cloudwatch put-metric-alarm \
  --alarm-name cloudfront-low-cache-hit \
  --alarm-description "CloudFront cache hit rate below 80%" \
  --metric-name CacheHitRate \
  --namespace AWS/CloudFront \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator LessThanThreshold
```

## 遇到的問題

### 問題 1：CORS 錯誤

**現象**：前端無法直接從 S3/CloudFront 下載檔案

**解決**：設定 CORS 規則

```bash
aws s3api put-bucket-cors \
  --bucket my-iot-storage \
  --cors-configuration file://cors.json
```

`cors.json`：

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://myapp.com"],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

### 問題 2：CloudFront 快取舊內容

**解決**：
1. 使用版本化的檔名
2. 建立 invalidation（成本較高）
3. 設定較短的 TTL

### 問題 3：上傳大檔案逾時

**解決**：使用 Multipart Upload 或預簽名 URL 讓前端直接上傳

## 總結

S3 + CloudFront 的組合提供了：
- **高可用性**：99.99% SLA
- **無限擴展**：不需擔心容量
- **全球加速**：CloudFront 邊緣節點
- **成本效益**：按使用量付費

對於靜態資源和檔案儲存，這是非常成熟的解決方案。相比自建檔案伺服器，省下大量維護時間。

經過這三個月的學習，已經建立了一個完整的 AWS IoT 架構：
- EC2：應用程式主機
- Lambda + API Gateway：Serverless API
- RabbitMQ：訊息佇列
- Step Functions：工作流程編排
- RDS：關聯式資料庫
- S3 + CloudFront：物件儲存與 CDN

下一步計畫深入研究 AWS 的其他服務，例如 ECS/EKS（容器服務）和 Kinesis（串流處理）。

## 參考資料

- [AWS S3 使用者指南](https://docs.aws.amazon.com/s3/)
- [CloudFront 開發人員指南](https://docs.aws.amazon.com/cloudfront/)
- [S3 最佳實踐](https://docs.aws.amazon.com/AmazonS3/latest/userguide/best-practices.html)
