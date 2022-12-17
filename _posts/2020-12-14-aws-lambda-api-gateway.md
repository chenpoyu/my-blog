---
layout: post
title: "AWS Lambda + API Gateway 實戰：打造 Serverless REST API"
date: 2020-12-14 15:47:00 +0800
categories: [AWS, Serverless]
tags: [AWS, Lambda, API Gateway, Serverless, Node.js]
---

## 前言

上週在 EC2 上部署了傳統的應用程式後，我開始思考一個問題：有些功能其實不需要 24/7 運行，例如定期報表生成、webhook 處理、簡單的 CRUD API 等。這時候 AWS Lambda 的 Serverless 架構就顯得很有吸引力。

今天要分享的是如何用 Lambda + API Gateway 建立一個 REST API，完全不需要管理伺服器。

## 什麼是 AWS Lambda？

Lambda 是一種「函式即服務」(FaaS) 的運算服務：

- **事件驅動**：由事件觸發執行（HTTP 請求、檔案上傳、定時任務等）
- **自動擴展**：流量增加時自動擴展，不需手動設定
- **按需計費**：只為實際執行時間付費（以 100ms 為單位）
- **免維護**：不需要管理作業系統、更新、修補程式

## Lambda 支援的程式語言

目前 Lambda 支援：
- Node.js 14.x, 12.x, 10.x
- Python 3.9, 3.8, 3.7, 3.6
- Java 11, Java 8
- .NET Core 3.1
- Go 1.x
- Ruby 2.7

作為 Java 和 .NET 背景的工程師，理論上可以繼續用熟悉的語言。但考慮到冷啟動時間，我選擇了 Node.js，因為它的啟動速度比 JVM 和 .NET 快很多。

## 第一個 Lambda 函式

### 建立 Lambda 函式

在 AWS Console 中建立 Lambda 函式：

1. 選擇「從頭開始編寫」
2. 函式名稱：`user-api-handler`
3. 執行階段：Node.js 14.x
4. 架構：x86_64

預設的程式碼範例：

```javascript
exports.handler = async (event) => {
    const response = {
        statusCode: 200,
        body: JSON.stringify('Hello from Lambda!'),
    };
    return response;
};
```

### Lambda 函式結構

Lambda 的進入點是 `handler` 函式，它接收兩個主要參數：

- **event**：觸發事件的資料（例如 API Gateway 傳來的 HTTP 請求）
- **context**：執行環境的資訊（請求 ID、剩餘執行時間等）

## 整合 API Gateway

Lambda 本身無法直接接收 HTTP 請求，需要透過 API Gateway 作為前端。

### 建立 REST API

1. 在 API Gateway 控制台建立新的 REST API
2. 建立資源：`/users`
3. 建立方法：GET、POST、PUT、DELETE
4. 整合類型：選擇 Lambda 函式
5. 選擇剛才建立的 `user-api-handler`

### 啟用 Lambda Proxy 整合

這是一個重要的設定，啟用後：
- API Gateway 會將完整的 HTTP 請求資訊傳給 Lambda
- Lambda 需要回傳符合格式的 HTTP 回應

```javascript
exports.handler = async (event) => {
    console.log('Event:', JSON.stringify(event, null, 2));
    
    const { httpMethod, path, queryStringParameters, body } = event;
    
    try {
        if (httpMethod === 'GET' && path === '/users') {
            return {
                statusCode: 200,
                headers: {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                body: JSON.stringify({
                    users: [
                        { id: 1, name: 'John Doe', email: 'john@example.com' },
                        { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
                    ]
                })
            };
        }
        
        if (httpMethod === 'POST' && path === '/users') {
            const userData = JSON.parse(body);
            // 這裡應該要寫入資料庫，目前先回傳建立成功
            return {
                statusCode: 201,
                headers: {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                body: JSON.stringify({
                    message: 'User created successfully',
                    user: userData
                })
            };
        }
        
        return {
            statusCode: 404,
            body: JSON.stringify({ error: 'Not Found' })
        };
        
    } catch (error) {
        console.error('Error:', error);
        return {
            statusCode: 500,
            body: JSON.stringify({ error: 'Internal Server Error' })
        };
    }
};
```

### 部署 API

在 API Gateway 中：
1. 點擊「動作」→「部署 API」
2. 選擇或建立一個階段（例如 `dev` 或 `prod`）
3. 取得 API 端點 URL

範例：`https://abc123.execute-api.us-east-1.amazonaws.com/dev/users`

## 測試 API

使用 curl 測試：

```bash
# GET 請求
curl https://abc123.execute-api.us-east-1.amazonaws.com/dev/users

# POST 請求
curl -X POST https://abc123.execute-api.us-east-1.amazonaws.com/dev/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'
```

## 環境變數管理

Lambda 支援環境變數，適合存放設定：

```javascript
const DB_HOST = process.env.DB_HOST;
const DB_NAME = process.env.DB_NAME;
const API_KEY = process.env.API_KEY;
```

在 Lambda 控制台的「組態」→「環境變數」中設定。

**注意**：敏感資訊（如密碼、API 金鑰）應該使用 AWS Secrets Manager 或 Systems Manager Parameter Store。

## 處理 CORS

前端呼叫 API 時常遇到 CORS 問題。有兩種解決方式：

### 方式 1：在 Lambda 中設定回應標頭

```javascript
return {
    statusCode: 200,
    headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Content-Type,Authorization',
        'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS'
    },
    body: JSON.stringify(data)
};
```

### 方式 2：在 API Gateway 啟用 CORS

在 API Gateway 控制台：
1. 選擇資源
2. 動作 → 啟用 CORS
3. 設定允許的來源、標頭、方法

## 連接資料庫

後來需要讓 Lambda 連接資料庫，我們用的是 RDS MySQL：

```javascript
const mysql = require('mysql2/promise');

let connection;

const getConnection = async () => {
    if (connection) {
        return connection;
    }
    
    connection = await mysql.createConnection({
        host: process.env.DB_HOST,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME
    });
    
    return connection;
};

exports.handler = async (event) => {
    const conn = await getConnection();
    const [rows] = await conn.execute('SELECT * FROM users');
    
    return {
        statusCode: 200,
        body: JSON.stringify(rows)
    };
};
```

**重要優化**：將資料庫連線放在 handler 外部，可以重用連線，減少冷啟動時間。

## Lambda 層 (Layers)

如果多個 Lambda 函式需要共用相同的程式庫，可以使用 Lambda 層：

1. 將 `node_modules` 打包成 ZIP
2. 在 Lambda 控制台建立層
3. 在函式中附加該層

這樣可以：
- 減少部署套件大小
- 程式碼與相依性分離
- 多個函式共用程式庫

## 權限管理 (IAM Role)

Lambda 執行時需要適當的權限。預設會建立一個執行角色，包含基本的 CloudWatch Logs 權限。

如果需要存取其他 AWS 服務，需要附加對應的政策：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "*"
        }
    ]
}
```

## 冷啟動問題

Lambda 的一個已知問題是「冷啟動」：當函式閒置一段時間後，再次呼叫會需要較長的初始化時間。

### 冷啟動時間比較（個人測試）

- Node.js: 100-300ms
- Python: 150-400ms
- Java: 2-5 秒
- .NET Core: 1-3 秒

### 減少冷啟動的方法

1. **選擇輕量級語言**：Node.js 或 Python
2. **減少套件大小**：只打包必要的相依性
3. **保持函式溫熱**：用 CloudWatch Events 定期觸發（但會增加成本）
4. **使用 Provisioned Concurrency**：預先初始化實例（2019 年底推出的功能）

## 成本分析

Lambda 的計費方式：

- **請求次數**：每 100 萬次請求 $0.20
- **執行時間**：每 GB-秒 $0.00001667

假設一個 API 函式：
- 記憶體：256 MB
- 平均執行時間：100 ms
- 每月請求：100 萬次

計算：
```
請求費用：1M * $0.20/M = $0.20
執行費用：1M * 0.1s * 0.25GB * $0.00001667 = $0.42
總計：$0.62/月
```

相比 EC2 t2.micro（約 $8/月），成本優勢明顯。

## 遇到的坑

### 問題 1：timeout 逾時

預設 timeout 是 3 秒，如果函式執行時間較長會被中斷。

**解決**：在函式設定中調整 timeout（最長 15 分鐘）

### 問題 2：記憶體不足

預設記憶體是 128 MB，可能不夠用。

**解決**：調整記憶體配置（同時會增加 CPU 效能）

### 問題 3：VPC 連線導致冷啟動變慢

如果 Lambda 需要存取 VPC 內的資源（如 RDS），冷啟動時間會大幅增加（10-30 秒）。

**解決**：AWS 在 2019 年改善了 VPC 連線機制，現在的冷啟動已經快很多。

## 與傳統 API 的比較

| 特性 | EC2 + Spring Boot | Lambda + API Gateway |
|------|-------------------|---------------------|
| 啟動時間 | 快（已常駐） | 冷啟動較慢 |
| 擴展性 | 需要手動設定 Auto Scaling | 自動擴展 |
| 成本 | 固定（即使無流量） | 按使用量計費 |
| 維護 | 需要管理 OS、更新 | 免維護 |
| 適用場景 | 高流量、複雜應用 | 間歇性、輕量級 API |

## 實際應用場景

我計畫將以下功能遷移到 Lambda：

1. **Webhook 處理**：接收第三方服務的通知
2. **定時報表生成**：每天凌晨執行
3. **圖片處理**：上傳圖片後自動產生縮圖
4. **簡單的 CRUD API**：低流量的管理介面

而核心的業務邏輯 API 仍然保留在 EC2，因為：
- 流量穩定且較高
- 有複雜的業務邏輯
- 需要長時間執行的任務

## 下一步探索

API Gateway 還有很多進階功能：
- **API Key 管理**：控制 API 存取
- **使用量計畫**：限制請求速率
- **快取**：減少 Lambda 呼叫次數
- **自訂網域**：使用自己的網域名稱

另外，我也想試試 **Step Functions**，用來編排多個 Lambda 函式的工作流程。

## 總結

Lambda + API Gateway 提供了一個快速建立 API 的方式，特別適合：
- 流量不穩定或低流量的服務
- 事件驅動的應用
- 微服務架構

雖然有冷啟動的問題，但對於大部分的應用場景來說，這個延遲是可接受的。而且隨著 AWS 持續優化，冷啟動時間也在逐步改善。

對於習慣寫 Java 和 .NET 的工程師來說，學習 Node.js 可能需要一些時間，但這個投資是值得的。Node.js 的生態系統非常豐富，而且非常適合處理 I/O 密集的任務。

下一篇計畫分享最近專案中用到的 RabbitMQ，特別是如何處理 IoT 裝置的 heartbeat 訊息。

## 參考資料

- [AWS Lambda 開發人員指南](https://docs.aws.amazon.com/lambda/)
- [API Gateway REST API 開發](https://docs.aws.amazon.com/apigateway/)
- [Lambda 最佳實踐](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
