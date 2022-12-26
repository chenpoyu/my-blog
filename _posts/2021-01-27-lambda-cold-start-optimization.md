---
layout: post
title: "AWS Lambda 冷啟動優化與最佳實踐"
date: 2021-01-27 13:10:00 +0800
categories: [AWS, Serverless]
tags: [AWS, Lambda, Performance, Optimization, Serverless]
---

## 前言

自從去年 12 月開始使用 AWS Lambda 以來，已經部署了十幾個函式，處理各種任務：webhook 接收、資料轉換、定時任務等。Lambda 確實方便，但也遇到一個老生常談的問題：**冷啟動**（Cold Start）。

這篇文章整理了我在優化 Lambda 效能時的經驗和最佳實踐。

## 什麼是冷啟動？

Lambda 函式閒置一段時間後，AWS 會回收執行環境。下次呼叫時需要：

1. **下載函式程式碼**
2. **初始化執行環境**（建立容器）
3. **載入 runtime**（Node.js、Python、Java 等）
4. **執行初始化程式碼**（handler 外部的程式碼）
5. **執行 handler 函式**

前 4 步就是冷啟動，只有第 5 步是實際的業務邏輯。

### 冷啟動時間實測

我測試了不同語言的冷啟動時間（256 MB 記憶體）：

| 語言 | 最小程式碼 | 有依賴套件 | 備註 |
|------|-----------|-----------|------|
| Node.js 14 | 150-250 ms | 300-500 ms | 最快 |
| Python 3.9 | 200-300 ms | 400-700 ms | 快 |
| Go 1.x | 100-200 ms | 200-400 ms | 編譯型，很快 |
| Java 11 | 2-4 秒 | 4-8 秒 | JVM 啟動慢 |
| .NET Core 3.1 | 1-2 秒 | 2-4 秒 | AOT 可改善 |

**結論**：對於注重冷啟動的場景，優先選擇 Node.js、Python 或 Go。

## 優化策略 1：減少套件大小

### Node.js 範例

**優化前**（使用完整的 AWS SDK）：

```javascript
const AWS = require('aws-sdk');
const s3 = new AWS.S3();

exports.handler = async (event) => {
    await s3.putObject({
        Bucket: 'my-bucket',
        Key: 'file.txt',
        Body: 'Hello World'
    }).promise();
};
```

部署套件大小：50 MB
冷啟動時間：500 ms

**優化後**（只引入需要的服務）：

```javascript
const S3 = require('aws-sdk/clients/s3');
const s3 = new S3();

exports.handler = async (event) => {
    await s3.putObject({
        Bucket: 'my-bucket',
        Key: 'file.txt',
        Body: 'Hello World'
    }).promise();
};
```

部署套件大小：8 MB
冷啟動時間：320 ms

**進一步優化**（使用 AWS SDK v3）：

```javascript
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const client = new S3Client({ region: 'us-east-1' });

exports.handler = async (event) => {
    const command = new PutObjectCommand({
        Bucket: 'my-bucket',
        Key: 'file.txt',
        Body: 'Hello World'
    });
    
    await client.send(command);
};
```

部署套件大小：3 MB
冷啟動時間：280 ms

### Java 範例

**優化前**：

```xml
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk</artifactId>
    <version>1.12.x</version>
</dependency>
```

**優化後**（只引入需要的模組）：

```xml
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-s3</artifactId>
    <version>1.12.x</version>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-rds</artifactId>
    <version>1.12.x</version>
</dependency>
```

### 使用 Webpack 打包

對於 Node.js，使用 Webpack 可以進一步減小體積：

```javascript
// webpack.config.js
module.exports = {
    entry: './src/handler.js',
    target: 'node',
    mode: 'production',
    output: {
        filename: 'index.js',
        path: path.resolve(__dirname, 'dist'),
        libraryTarget: 'commonjs2'
    },
    externals: {
        'aws-sdk': 'aws-sdk'  // AWS SDK 已內建在 Lambda 環境
    },
    optimization: {
        minimize: true
    }
};
```

打包後體積可減少 50-70%。

## 優化策略 2：提升記憶體配置

Lambda 的 CPU 效能與記憶體成正比。提升記憶體不僅增加 RAM，也會分配更多 CPU。

### 實測資料

測試函式：解析 1 MB 的 JSON 檔案

| 記憶體 | 執行時間 | 成本 | 備註 |
|--------|---------|------|------|
| 128 MB | 800 ms | $0.0000002 | 最便宜但最慢 |
| 256 MB | 450 ms | $0.0000002 | 甜蜜點 |
| 512 MB | 280 ms | $0.0000003 | 速度提升明顯 |
| 1024 MB | 200 ms | $0.0000004 | 性價比下降 |
| 3008 MB | 150 ms | $0.0000010 | 最快但昂貴 |

**結論**：對於大多數應用，256-512 MB 是最佳選擇。

### 使用 Lambda Power Tuning

AWS 提供了一個開源工具來找出最佳記憶體配置：

```bash
# 部署 Power Tuning
git clone https://github.com/alexcasalboni/aws-lambda-power-tuning.git
cd aws-lambda-power-tuning
npm install
sam deploy --guided

# 執行調校
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:powerTuningStateMachine \
  --input '{
    "lambdaARN": "arn:aws:lambda:us-east-1:123456789012:function:my-function",
    "powerValues": [128, 256, 512, 1024, 1536, 3008],
    "num": 10,
    "payload": {}
  }'
```

工具會測試不同記憶體配置，並產生視覺化報告。

## 優化策略 3：全域變數與連線重用

### 錯誤做法（每次建立連線）

```javascript
exports.handler = async (event) => {
    const mysql = require('mysql2/promise');
    
    // 每次都建立新連線！
    const connection = await mysql.createConnection({
        host: process.env.DB_HOST,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME
    });
    
    const [rows] = await connection.execute('SELECT * FROM devices');
    await connection.end();
    
    return { statusCode: 200, body: JSON.stringify(rows) };
};
```

### 正確做法（重用連線）

```javascript
const mysql = require('mysql2/promise');

// 在 handler 外部建立連線（全域變數）
let connection;

async function getConnection() {
    if (!connection) {
        connection = await mysql.createConnection({
            host: process.env.DB_HOST,
            user: process.env.DB_USER,
            password: process.env.DB_PASSWORD,
            database: process.env.DB_NAME
        });
    }
    return connection;
}

exports.handler = async (event) => {
    const conn = await getConnection();
    const [rows] = await conn.execute('SELECT * FROM devices');
    
    return { statusCode: 200, body: JSON.stringify(rows) };
};
```

**效能改善**：
- 首次呼叫（冷啟動）：500 ms
- 後續呼叫（溫啟動）：50 ms（省下連線建立時間）

### Java 範例

```java
public class DeviceHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    // 靜態變數，在 Lambda 容器生命週期內重用
    private static AmazonS3 s3Client;
    private static Connection dbConnection;
    
    static {
        s3Client = AmazonS3ClientBuilder.standard()
            .withRegion("us-east-1")
            .build();
        
        try {
            dbConnection = DriverManager.getConnection(
                System.getenv("DB_URL"),
                System.getenv("DB_USER"),
                System.getenv("DB_PASSWORD")
            );
        } catch (SQLException e) {
            throw new RuntimeException("Failed to connect to database", e);
        }
    }
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent request, Context context) {
        // 使用已初始化的 client
        GetObjectRequest getObjectRequest = new GetObjectRequest("my-bucket", "file.txt");
        S3Object s3Object = s3Client.getObject(getObjectRequest);
        
        // ...
    }
}
```

## 優化策略 4：Provisioned Concurrency

如果無法接受冷啟動延遲，可以使用 Provisioned Concurrency（預配並發）。

### 啟用 Provisioned Concurrency

```bash
# 發布版本
aws lambda publish-version --function-name my-function

# 設定預配並發
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier 1 \
  --provisioned-concurrent-executions 5
```

### 成本考量

Provisioned Concurrency 會持續計費，即使沒有請求：

- **預配成本**：$0.0000041667/GB-秒
- **執行成本**：與一般 Lambda 相同

範例：5 個 512 MB 實例，24/7 運行

```
預配成本：5 * 0.5 GB * 86400 秒/天 * 30 天 * $0.0000041667 = $27/月
```

相比隨需模式，成本增加顯著。只對關鍵路徑使用。

### 自動擴展 Provisioned Concurrency

使用 Application Auto Scaling：

```bash
# 註冊擴展目標
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:my-function:1 \
  --scalable-dimension lambda:function:ProvisionedConcurrentExecutions \
  --min-capacity 2 \
  --max-capacity 10

# 設定擴展政策
aws application-autoscaling put-scaling-policy \
  --service-namespace lambda \
  --resource-id function:my-function:1 \
  --scalable-dimension lambda:function:ProvisionedConcurrentExecutions \
  --policy-name my-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 0.70,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "LambdaProvisionedConcurrencyUtilization"
    }
  }'
```

## 優化策略 5：Lambda Layers

將共用的程式庫提取到 Layer，減少部署套件大小。

### 建立 Layer

```bash
# 準備 Layer 內容
mkdir -p layer/nodejs/node_modules
cd layer/nodejs
npm install aws-sdk lodash moment

# 打包
cd ..
zip -r layer.zip nodejs/

# 發布 Layer
aws lambda publish-layer-version \
  --layer-name common-dependencies \
  --zip-file fileb://layer.zip \
  --compatible-runtimes nodejs14.x
```

### 使用 Layer

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:us-east-1:123456789012:layer:common-dependencies:1
```

Lambda 函式：

```javascript
const AWS = require('aws-sdk');  // 來自 Layer
const lodash = require('lodash');  // 來自 Layer

exports.handler = async (event) => {
    // 業務邏輯
};
```

**好處**：
- 部署套件更小
- Layer 可在多個函式間共用
- 更新 Layer 不影響函式程式碼

## 優化策略 6：VPC 設定

Lambda 在 VPC 內執行時，冷啟動時間會顯著增加（曾經是 10-30 秒）。

### 2019 年前的問題

Lambda 需要為每個執行環境建立 ENI（Elastic Network Interface），非常慢。

### 2019 年後的改善

AWS 改善了 VPC 連線機制，現在冷啟動只增加 1-2 秒。

### 最佳實踐

如果不需要存取 VPC 內的資源（如 RDS、ElastiCache），不要把 Lambda 放在 VPC 內。

如果需要，確保：
1. **子網路有足夠的 IP**：每個執行環境需要一個 IP
2. **使用 NAT Gateway**：如果需要存取網際網路
3. **Security Group 正確設定**：允許必要的出站流量

## 優化策略 7：X-Ray 追蹤

使用 X-Ray 了解函式的效能瓶頸。

### 啟用 X-Ray

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --tracing-config Mode=Active
```

### 在程式碼中使用

```javascript
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

exports.handler = async (event) => {
    const segment = AWSXRay.getSegment();
    
    // 建立子區段
    const subsegment = segment.addNewSubsegment('database-query');
    try {
        const result = await queryDatabase();
        subsegment.close();
        return result;
    } catch (error) {
        subsegment.addError(error);
        subsegment.close();
        throw error;
    }
};
```

### 分析結果

X-Ray 會顯示：
- 冷啟動 vs 溫啟動比例
- 初始化時間
- 各個操作的耗時
- 外部服務呼叫延遲

## 實戰案例：優化 IoT Webhook

### 優化前

```javascript
// 500 行程式碼，包含大量 npm 套件
const express = require('express');
const bodyParser = require('body-parser');
const AWS = require('aws-sdk');
const mysql = require('mysql2/promise');
const moment = require('moment');
const lodash = require('lodash');
// ... 更多套件

exports.handler = async (event) => {
    // 每次建立資料庫連線
    const connection = await mysql.createConnection({...});
    
    // 處理邏輯
    const result = await processWebhook(event, connection);
    
    await connection.end();
    return result;
};
```

- 部署套件：45 MB
- 冷啟動時間：800 ms
- 溫啟動時間：150 ms

### 優化後

```javascript
// 只引入必要的套件
const mysql = require('mysql2/promise');

// 全域連線，重複使用
let connection;

const getConnection = async () => {
    if (!connection) {
        connection = await mysql.createConnection({
            host: process.env.DB_HOST,
            user: process.env.DB_USER,
            password: process.env.DB_PASSWORD,
            database: process.env.DB_NAME
        });
    }
    return connection;
};

exports.handler = async (event) => {
    const body = JSON.parse(event.body);
    
    // 簡單驗證
    if (!body.deviceId || !body.data) {
        return { statusCode: 400, body: 'Invalid request' };
    }
    
    // 寫入 RDS
    const conn = await getConnection();
    await conn.execute(
        'INSERT INTO device_data (device_id, data, created_at) VALUES (?, ?, ?)',
        [body.deviceId, JSON.stringify(body.data), new Date()]
    );
            timestamp: { N: Date.now().toString() },
            data: { S: JSON.stringify(body.data) }
        }
    });
    
    await client.send(command);
    
    return { statusCode: 202, body: 'Accepted' };
};
```

- 部署套件：2 MB
- 冷啟動時間：280 ms
- 溫啟動時間：40 ms

**改善**：
- 套件大小減少 95%
- 冷啟動時間減少 65%
- 溫啟動時間減少 73%

## 監控與告警

### CloudWatch Metrics

關鍵指標：
- **Duration**：執行時間
- **Errors**：錯誤次數
- **Throttles**：限流次數
- **ConcurrentExecutions**：並發執行數
- **InitDuration**：初始化時間（冷啟動）

### 設定告警

```bash
# 冷啟動時間過長
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-slow-cold-start \
  --alarm-description "Lambda cold start > 1s" \
  --metric-name InitDuration \
  --namespace AWS/Lambda \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=my-function
```

### 使用 CloudWatch Insights

查詢冷啟動比例：

```
filter @type = "REPORT"
| fields @initDuration, @duration
| stats count(@initDuration > 0) as coldStarts, count(*) as totalInvocations
| put coldStartPercentage = coldStarts / totalInvocations * 100
```

## 成本優化

### 實際成本分析

假設每月 1000 萬次呼叫，每次 200 ms，512 MB 記憶體：

```
請求成本：10M * $0.20/M = $2.00
運算成本：10M * 0.2s * 0.5GB * $0.0000166667 = $16.67
總計：$18.67/月
```

### 優化建議

1. **減少記憶體**：如果不需要高效能，降低記憶體配置
2. **減少執行時間**：優化程式碼，減少外部呼叫
3. **使用 Graviton2**：ARM 架構，成本降低 20%
4. **批次處理**：合併多個請求，減少呼叫次數

### Graviton2 範例

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --architectures arm64
```

## 最佳實踐總結

1. **選擇合適的語言**：Node.js、Python、Go 冷啟動快
2. **減少套件大小**：只引入需要的模組
3. **提升記憶體**：256-512 MB 通常是甜蜜點
4. **重用連線**：使用全域變數
5. **避免 VPC**（如果可能）：除非必要
6. **使用 Layers**：共用程式庫
7. **Provisioned Concurrency**：關鍵路徑使用
8. **監控與優化**：持續追蹤指標

## 結語

Lambda 的冷啟動問題雖然存在，但透過系統性的優化，可以將影響降到最低。對於大多數應用場景，優化後的冷啟動時間（200-400 ms）是可以接受的。

如果真的無法接受，還有其他選擇：
- **ECS Fargate**：容器化部署，無冷啟動
- **EKS**：Kubernetes，更複雜但更靈活
- **EC2**：傳統方式，完全控制

但對於事件驅動、間歇性的工作負載，Lambda 仍然是最佳選擇。

經過兩個月的深入使用，我已經能夠熟練地運用 Lambda 解決各種問題。下一步計畫探索如何將這些經驗整合成一個完整的微服務架構。

## 參考資料

- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Lambda Performance Optimization](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/)
- [Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)
