---
layout: post
title: "AWS Step Functions：編排複雜的 Serverless 工作流程"
date: 2020-12-28 14:32:00 +0800
categories: [AWS, Serverless]
tags: [AWS, Step Functions, Lambda, Workflow, State Machine]
---

## 前言

在使用 Lambda 一段時間後，遇到了一個問題：如何協調多個 Lambda 函式來完成複雜的業務流程？

舉個實際例子，我們的 IoT 系統需要處理裝置註冊流程：

1. 驗證裝置資訊
2. 檢查裝置是否已存在
3. 寫入資料庫
4. 產生憑證
5. 發送通知給管理員
6. 記錄稽核日誌

如果用多個 Lambda 函式實作，會遇到：
- 如何協調執行順序？
- 如何處理錯誤和重試？
- 如何追蹤整個流程的狀態？
- 如何實作條件分支和並行處理？

AWS Step Functions 就是為了解決這些問題而生的。

## 什麼是 Step Functions？

Step Functions 是一個 **狀態機** (State Machine) 服務，用來協調分散式應用程式和微服務的執行流程。

核心概念：
- **State Machine**：工作流程定義
- **State**：流程中的每個步驟
- **Execution**：工作流程的一次執行
- **Task**：呼叫 AWS 服務（如 Lambda、RDS、SNS）

### 支援的 State 類型

| State 類型 | 說明 | 使用場景 |
|-----------|------|---------|
| Task | 執行工作（呼叫 Lambda、API 等） | 業務邏輯處理 |
| Choice | 條件分支 | if-else 邏輯 |
| Parallel | 並行執行多個分支 | 同時執行多個任務 |
| Wait | 等待一段時間 | 延遲處理 |
| Succeed | 成功結束 | 流程完成 |
| Fail | 失敗結束 | 錯誤處理 |
| Pass | 傳遞資料（不執行任何動作） | 測試或資料轉換 |
| Map | 迭代處理陣列 | 批次處理 |

## 建立第一個 State Machine

### 場景：裝置註冊流程

用 Amazon States Language (ASL) 定義狀態機：

```json
{
  "Comment": "Device Registration Workflow",
  "StartAt": "ValidateDevice",
  "States": {
    "ValidateDevice": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateDevice",
      "ResultPath": "$.validationResult",
      "Next": "CheckDeviceExists",
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "ResultPath": "$.error",
          "Next": "ValidationFailed"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ]
    },
    "CheckDeviceExists": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CheckDevice",
      "ResultPath": "$.deviceExists",
      "Next": "DeviceExistsChoice"
    },
    "DeviceExistsChoice": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.deviceExists",
          "BooleanEquals": true,
          "Next": "DeviceAlreadyExists"
        }
      ],
      "Default": "RegisterDevice"
    },
    "DeviceAlreadyExists": {
      "Type": "Fail",
      "Error": "DeviceExistsError",
      "Cause": "Device already registered"
    },
    "RegisterDevice": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "SaveToDatabase",
          "States": {
            "SaveToDatabase": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:SaveDevice",
              "End": true
            }
          }
        },
        {
          "StartAt": "GenerateCertificate",
          "States": {
            "GenerateCertificate": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:GenerateCert",
              "End": true
            }
          }
        }
      ],
      "ResultPath": "$.registrationResults",
      "Next": "SendNotification"
    },
    "SendNotification": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:SendNotification",
      "ResultPath": "$.notificationResult",
      "Next": "LogAudit"
    },
    "LogAudit": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:LogAudit",
      "End": true
    },
    "ValidationFailed": {
      "Type": "Fail",
      "Error": "ValidationError",
      "Cause": "Device validation failed"
    }
  }
}
```

### 流程說明

1. **ValidateDevice**：驗證裝置資訊
   - 設定重試策略：失敗時重試 3 次，間隔遞增
   - 捕捉 ValidationError 錯誤

2. **CheckDeviceExists**：檢查裝置是否已存在

3. **DeviceExistsChoice**：條件判斷
   - 如果裝置已存在 → 流程失敗
   - 否則 → 繼續註冊

4. **RegisterDevice**：並行執行
   - 分支 1：寫入資料庫
   - 分支 2：產生憑證

5. **SendNotification**：發送通知

6. **LogAudit**：記錄稽核日誌

## Lambda 函式實作

### ValidateDevice Lambda

```javascript
exports.handler = async (event) => {
    console.log('Validating device:', JSON.stringify(event));
    
    const { deviceId, deviceType, serialNumber } = event;
    
    // 驗證必填欄位
    if (!deviceId || !deviceType || !serialNumber) {
        const error = new Error('Missing required fields');
        error.name = 'ValidationError';
        throw error;
    }
    
    // 驗證格式
    if (!/^[A-Z0-9]{8}$/.test(deviceId)) {
        const error = new Error('Invalid device ID format');
        error.name = 'ValidationError';
        throw error;
    }
    
    return {
        valid: true,
        deviceId,
        deviceType,
        serialNumber
    };
};
```

### CheckDevice Lambda

```javascript
const mysql = require('mysql2/promise');

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
    console.log('Checking device:', event.deviceId);
    
    try {
        const conn = await getConnection();
        const [rows] = await conn.execute(
            'SELECT * FROM devices WHERE device_id = ?',
            [event.deviceId]
        );
        return rows.length > 0;
        
    } catch (error) {
        console.error('Database error:', error);
        throw error;
    }
};
```

### SaveDevice Lambda

```javascript
const mysql = require('mysql2/promise');

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
    console.log('Saving device:', JSON.stringify(event));
    
    try {
        const conn = await getConnection();
        await conn.execute(
            'INSERT INTO devices (device_id, device_type, serial_number, status, registered_at) VALUES (?, ?, ?, ?, ?)',
            [event.deviceId, event.deviceType, event.serialNumber, 'active', new Date()]
        );
        return { success: true, deviceId: event.deviceId };
        
    } catch (error) {
        console.error('Failed to save device:', error);
        throw error;
    }
};
```

## 資料傳遞與轉換

Step Functions 在 State 之間傳遞資料的機制：

### InputPath、ResultPath、OutputPath

```json
{
  "ValidateDevice": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:...",
    "InputPath": "$.device",
    "ResultPath": "$.validationResult",
    "OutputPath": "$",
    "Next": "CheckDeviceExists"
  }
}
```

- **InputPath**：選擇輸入資料的哪個部分傳給 Task
- **ResultPath**：將 Task 結果存到哪個欄位
- **OutputPath**：選擇輸出資料的哪個部分傳給下一個 State

### Parameters 參數轉換

```json
{
  "SendNotification": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:...",
    "Parameters": {
      "message.$": "$.device.deviceId",
      "recipient": "admin@example.com",
      "timestamp.$": "$$.Execution.StartTime"
    },
    "Next": "LogAudit"
  }
}
```

`$` 表示輸入資料，`$$` 表示 Step Functions 的內建變數。

## 錯誤處理與重試

### Retry（重試）

```json
{
  "Retry": [
    {
      "ErrorEquals": ["States.TaskFailed"],
      "IntervalSeconds": 2,
      "MaxAttempts": 3,
      "BackoffRate": 2.0
    },
    {
      "ErrorEquals": ["States.Timeout"],
      "IntervalSeconds": 1,
      "MaxAttempts": 2,
      "BackoffRate": 1.5
    }
  ]
}
```

- **IntervalSeconds**：第一次重試前等待的秒數
- **MaxAttempts**：最大重試次數
- **BackoffRate**：每次重試的等待時間倍數

### Catch（捕捉錯誤）

```json
{
  "Catch": [
    {
      "ErrorEquals": ["ValidationError"],
      "ResultPath": "$.error",
      "Next": "ValidationFailed"
    },
    {
      "ErrorEquals": ["States.ALL"],
      "ResultPath": "$.error",
      "Next": "HandleError"
    }
  ]
}
```

預定義的錯誤類型：
- `States.ALL`：所有錯誤
- `States.TaskFailed`：Task 執行失敗
- `States.Timeout`：逾時
- `States.Permissions`：權限錯誤

## 執行 State Machine

### 透過 AWS Console

1. 進入 Step Functions 控制台
2. 選擇 State Machine
3. 點擊「開始執行」
4. 輸入執行輸入：

```json
{
  "deviceId": "ABC12345",
  "deviceType": "sensor",
  "serialNumber": "SN-2021-12345"
}
```

### 透過 AWS SDK

```java
import com.amazonaws.services.stepfunctions.AWSStepFunctions;
import com.amazonaws.services.stepfunctions.AWSStepFunctionsClientBuilder;
import com.amazonaws.services.stepfunctions.model.StartExecutionRequest;
import com.amazonaws.services.stepfunctions.model.StartExecutionResult;

@Service
public class StepFunctionsService {
    
    private final AWSStepFunctions stepFunctions;
    private final String stateMachineArn;
    
    public StepFunctionsService() {
        this.stepFunctions = AWSStepFunctionsClientBuilder.defaultClient();
        this.stateMachineArn = "arn:aws:states:us-east-1:123456789012:stateMachine:DeviceRegistration";
    }
    
    public String startDeviceRegistration(String deviceId, String deviceType, String serialNumber) {
        String input = String.format(
            "{\"deviceId\":\"%s\",\"deviceType\":\"%s\",\"serialNumber\":\"%s\"}",
            deviceId, deviceType, serialNumber
        );
        
        StartExecutionRequest request = new StartExecutionRequest()
            .withStateMachineArn(stateMachineArn)
            .withInput(input);
        
        StartExecutionResult result = stepFunctions.startExecution(request);
        
        return result.getExecutionArn();
    }
}
```

### 透過 API Gateway + Lambda

```javascript
const AWS = require('aws-sdk');
const stepfunctions = new AWS.StepFunctions();

exports.handler = async (event) => {
    const body = JSON.parse(event.body);
    
    const params = {
        stateMachineArn: process.env.STATE_MACHINE_ARN,
        input: JSON.stringify(body)
    };
    
    try {
        const result = await stepfunctions.startExecution(params).promise();
        
        return {
            statusCode: 202,
            body: JSON.stringify({
                executionArn: result.executionArn,
                message: 'Workflow started'
            })
        };
    } catch (error) {
        console.error('Failed to start workflow:', error);
        return {
            statusCode: 500,
            body: JSON.stringify({ error: 'Failed to start workflow' })
        };
    }
};
```

## Map State：批次處理

處理多個裝置的場景：

```json
{
  "ProcessDevices": {
    "Type": "Map",
    "ItemsPath": "$.devices",
    "MaxConcurrency": 5,
    "Iterator": {
      "StartAt": "ProcessDevice",
      "States": {
        "ProcessDevice": {
          "Type": "Task",
          "Resource": "arn:aws:lambda:...:function:ProcessSingleDevice",
          "End": true
        }
      }
    },
    "ResultPath": "$.processResults",
    "Next": "GenerateSummary"
  }
}
```

輸入範例：

```json
{
  "devices": [
    {"deviceId": "DEV001", "action": "update"},
    {"deviceId": "DEV002", "action": "delete"},
    {"deviceId": "DEV003", "action": "update"}
  ]
}
```

## Wait State：延遲執行

```json
{
  "WaitForVerification": {
    "Type": "Wait",
    "Seconds": 300,
    "Next": "CheckVerificationStatus"
  }
}
```

或使用動態等待時間：

```json
{
  "WaitUntil": {
    "Type": "Wait",
    "TimestampPath": "$.scheduledTime",
    "Next": "ProcessScheduledTask"
  }
}
```

## 整合其他 AWS 服務

Step Functions 可以直接呼叫多種 AWS 服務，不需要透過 Lambda：

### 發送 SNS 通知

```json
{
  "SendNotification": {
    "Type": "Task",
    "Resource": "arn:aws:states:::sns:publish",
    "Parameters": {
      "TopicArn": "arn:aws:sns:us-east-1:123456789012:DeviceAlerts",
      "Message.$": "$.message"
      }
    },
    "Next": "SendSNS"
  }
}
```

### 直接發送 SNS 通知

```json
{
  "SendSNS": {
    "Type": "Task",
    "Resource": "arn:aws:states:::sns:publish",
    "Parameters": {
      "TopicArn": "arn:aws:sns:us-east-1:123456789012:DeviceNotifications",
      "Message.$": "$.message"
    },
    "End": true
  }
}
```

### 啟動另一個 Step Functions

```json
{
  "StartChildWorkflow": {
    "Type": "Task",
    "Resource": "arn:aws:states:::states:startExecution.sync",
    "Parameters": {
      "StateMachineArn": "arn:aws:states:us-east-1:123456789012:stateMachine:ChildWorkflow",
      "Input.$": "$"
    },
    "Next": "ProcessResult"
  }
}
```

## 監控與除錯

### CloudWatch Logs

在 State Machine 設定中啟用日誌記錄：

```json
{
  "loggingConfiguration": {
    "level": "ALL",
    "includeExecutionData": true,
    "destinations": [
      {
        "cloudWatchLogsLogGroup": {
          "logGroupArn": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/stepfunctions/DeviceRegistration"
        }
      }
    ]
  }
}
```

### X-Ray 整合

啟用 X-Ray 追蹤：

```json
{
  "tracingConfiguration": {
    "enabled": true
  }
}
```

### 視覺化執行狀態

Step Functions 控制台提供了直觀的視覺化介面：
- 即時顯示執行進度
- 每個 State 的輸入/輸出
- 錯誤訊息和堆疊追蹤
- 執行時間統計

## 成本分析

Step Functions 的計費方式：

### Standard Workflows（標準工作流程）
- 每 1,000 次狀態轉換：$0.025
- 適合長時間運行的工作流程（最長 1 年）

### Express Workflows（快速工作流程）
- 每 100 萬次請求：$1.00
- 按執行時間計費：每 GB-秒 $0.00001667
- 適合高流量、短時間的工作流程（最長 5 分鐘）

### 範例計算

假設每月執行 10 萬次工作流程，每次 5 個狀態轉換：

```
狀態轉換次數：100,000 * 5 = 500,000
費用：500 * $0.025 = $12.50/月
```

相比用 Lambda 協調多個函式，Step Functions 的成本合理，而且大幅簡化了開發複雜度。

## Express vs Standard 的選擇

| 特性 | Standard | Express |
|------|----------|---------|
| 最長執行時間 | 1 年 | 5 分鐘 |
| 執行歷史 | 保留 90 天 | 僅 CloudWatch Logs |
| 執行語義 | Exactly-once | At-least-once |
| 計費模式 | 按狀態轉換 | 按請求數和執行時間 |
| 適用場景 | 複雜、長時間流程 | 高頻、短時間流程 |

對於 IoT heartbeat 處理，我會選 Express；對於裝置註冊，選 Standard。

## 實際應用案例

除了裝置註冊，我們還用 Step Functions 處理：

### 1. 資料處理管線

```
S3 上傳 → Lambda (驗證) → Lambda (轉換) → Lambda (儲存) → SNS (通知)
```

### 2. 訂單處理流程

```
建立訂單 → 檢查庫存 → 預留庫存 → 處理付款 → 確認訂單 → 發送通知
```

### 3. 批次報表生成

```
Map 處理多個資料源 → 並行查詢 → 合併結果 → 生成 PDF → 上傳 S3 → 發送郵件
```

## 遇到的問題

### 問題 1：狀態機定義太複雜

**解決**：將大型狀態機拆分成多個小型狀態機，用 `startExecution` 組合

### 問題 2：Lambda 冷啟動影響整體執行時間

**解決**：使用 Provisioned Concurrency 或選擇 Express 模式

### 問題 3：錯誤訊息不清楚

**解決**：在 Lambda 中記錄詳細日誌，啟用 CloudWatch Logs

## 與其他方案比較

### vs 直接用 Lambda 呼叫 Lambda

| Step Functions | Lambda 呼叫 Lambda |
|----------------|-------------------|
| 視覺化流程 | 程式碼中隱藏 |
| 內建重試和錯誤處理 | 需自行實作 |
| 狀態追蹤 | 需自行記錄 |
| 較高成本 | 較低成本 |

### vs SQS + Lambda

| Step Functions | SQS + Lambda |
|----------------|--------------|
| 複雜流程編排 | 簡單佇列處理 |
| 支援條件分支 | 無法條件路由 |
| 內建並行處理 | 需多個佇列 |
| 視覺化監控 | 需自建 |

## 總結

Step Functions 是協調 Serverless 應用的強大工具，特別適合：
- 需要編排多個 Lambda 函式
- 有複雜的條件邏輯和並行處理
- 需要可靠的錯誤處理和重試機制
- 想要視覺化追蹤執行狀態

對於熟悉 Java 和 .NET 的工程師，Step Functions 的狀態機概念可能需要一些時間適應，但一旦掌握，就能大幅簡化複雜流程的開發。

Amazon States Language (ASL) 的學習曲線不算陡峭，搭配 AWS Console 的視覺化編輯器，可以快速上手。

下一篇計畫分享 AWS RDS 和 CloudWatch 的使用經驗，看看如何建立可靠的資料庫監控機制。

## 參考資料

- [AWS Step Functions 開發人員指南](https://docs.aws.amazon.com/step-functions/)
- [Amazon States Language 規範](https://states-language.net/spec.html)
- [Step Functions 最佳實踐](https://docs.aws.amazon.com/step-functions/latest/dg/best-practices.html)
