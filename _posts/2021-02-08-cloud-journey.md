---
layout: post
title: "從 .NET Core 轉戰 Java + AWS：三個月的雲端開發心路歷程"
date: 2021-02-08 14:45:00 +0800
categories: [Architecture, AWS]
tags: [AWS, Java, Spring Boot, IoT, Cloud Architecture]
---

## 前言

去年底才剛完成一個 .NET Core + Vue.js 的專案，本來以為可以喘口氣，沒想到馬上就接到新的挑戰。這次的專案是物聯網平台，客戶指定要用 Java 開發，而且所有的基礎設施都要跑在 AWS 上。

說實話，我心裡有點掙扎。雖然以前寫過 Java，但已經好一陣子沒碰了，更別說 AWS 了——之前的專案都是在地端機房或者公司自己的虛擬機器上跑。不過既然是工作，還是硬著頭皮接下來。

回頭看這三個月（2020/12 - 2021/2），真的是蠻刺激的一段經歷。從對 AWS 一無所知，到現在能夠獨立規劃和部署整個系統，這篇文章想記錄一下這段學習過程中的心得、踩過的坑，還有一些想法。

## 專案需求

這次的物聯網平台要處理的東西還不少：
- 大約 8,000 多個裝置會定期回傳 heartbeat（大概每 30 秒一次）
- 裝置的各種資料要收集起來存到資料庫
- 韌體更新檔要能分發給裝置下載
- 裝置如果離線或異常要能即時告警
- 還要做一個管理後台讓客戶可以查看和管理裝置

客戶那邊很明確要用 AWS，所以我得從頭學。

## 重拾 Java 的感覺

第一個禮拜光是把開發環境建起來就搞了好久。從 .NET Core 的世界回到 Java，很多東西都要重新適應：

- IDE 從 Visual Studio / Rider 換回 IntelliJ IDEA
- 依賴管理從 NuGet 變成 Maven（還好概念類似）
- Web 框架從 ASP.NET Core 變成 Spring Boot
- ORM 從 Entity Framework 換成 JPA/Hibernate

不過說實在的，Spring Boot 用起來還蠻順手的，很多設定都是自動完成，比我之前用 Spring MVC 的時候方便太多了。而且文件也寫得很清楚，遇到問題 Google 一下基本上都能找到答案。

```java
// 第一個 Spring Boot Controller，懷念的感覺又回來了
@RestController
@RequestMapping("/api/devices")
public class DeviceController {
    
    @Autowired
    private DeviceService deviceService;
    
    @GetMapping
    public ResponseEntity<List<Device>> getDevices() {
        return ResponseEntity.ok(deviceService.findAll());
    }
}
```

寫起來跟 .NET Core 的 Controller 其實概念差不多，只是 Annotation 不一樣而已。

## 架構演進：從簡單到複雜

### 一開始的做法（2020/12 初）

第一版我就是按照最傳統的方式做：租了一台 EC2，把 Spring Boot 應用和 MySQL 都裝在上面。

```
┌─────────────────┐
│   EC2 Instance  │
│  ┌────────────┐ │
│  │Spring Boot │ │
│  │Application │ │
│  └────────────┘ │
│  ┌────────────┐ │
│  │   MySQL    │ │
│  └────────────┘ │
└─────────────────┘
```

這樣做最快，而且一開始裝置數量不多，也還跑得動。但我心裡知道這不是長久之計：
- 如果這台機器掛了，整個系統就死了
- 備份要自己搞
- 更新的時候會停機
- 無法應付突然增加的流量

不過當時的想法是：先跑起來再說，反正是測試階段。

### 遇到第一個瓶頸（2020/12 中）

系統上線測試大概兩個禮拜後，裝置數量從一開始的幾百台快速增長到三千多台。問題就來了：

API 開始變慢，原本幾十毫秒的請求現在要好幾秒。我用 JProfiler 抓了一下，發現瓶頸在資料庫寫入。那時候的設計是每次收到 heartbeat 就直接寫進 MySQL，8,000 台裝置每 30 秒打一次，等於每秒要處理將近 300 次寫入。

資料庫 CPU 直接爆表。

主管看到監控數據也傻眼，問我有沒有辦法優化。我想了幾個方案：
1. 升級資料庫規格（治標不治本，而且貴）
2. 批次寫入（要改很多程式碼）
3. 加個 message queue 緩衝（感覺比較正規）

最後決定引入 RabbitMQ。

### 引入 RabbitMQ 解決燃眉之急（2020/12 下旬）

我在 EC2 上裝了 RabbitMQ，調整架構變成這樣：

```
IoT Devices → Spring Boot API → RabbitMQ → Consumer → MySQL
```

API 收到 heartbeat 後不直接寫資料庫，而是丟到 RabbitMQ，然後立刻回應 202 Accepted。後端有個 Consumer 不斷從 queue 拿訊息出來處理。

效果立竿見影，API 回應時間從幾秒降到 50ms 以下。資料庫的壓力也緩解了，因為 Consumer 可以用批次寫入的方式處理。

```java
@RabbitListener(queues = "heartbeat.queue")
public void processHeartbeat(HeartbeatMessage message) {
    try {
        deviceService.updateStatus(message);
        log.debug("Processed heartbeat: {}", message.getDeviceId());
    } catch (Exception e) {
        log.error("Failed to process heartbeat", e);
        // 這裡應該要有 retry 機制，但當時還沒做
    }
}
```

不過這樣還是有個問題：所有東西都在同一台 EC2 上，還是單點故障。我跟主管提了這個風險，他說先這樣，等穩定一點再說。

### 客戶要求高可用（2021/1 初）

果然好景不長。有一次那台 EC2 不知道為什麼突然重開機（後來查是 AWS 在做維護），系統就掛了大概 10 分鐘。客戶那邊馬上打電話來問，我只能尷尬地解釋是主機重開機。

客戶不太高興，說這樣不行，要求我們提供一個高可用的方案。這下不做不行了。

我開始認真研究 AWS 的各種服務。首先把資料庫搬到 RDS，至少備份和容錯移轉 AWS 會幫忙處理。然後把 RabbitMQ 改成 3 node cluster，跨不同的 availability zone。API 的部分也多開了幾台 EC2，前面放個 Application Load Balancer 做分流。

```
                    Internet
                       ↓
              ┌────────────────┐
              │      ALB       │
              └────────────────┘
                 ↓    ↓    ↓
          ┌─────┴─────┴─────┐
          │   EC2 Cluster   │
          │  (Spring Boot)  │
          └─────────────────┘
                   ↓
          ┌─────────────────┐
          │  RabbitMQ (3 nodes) │
          └─────────────────┘
                   ↓
          ┌─────────────────┐
          │   RDS MySQL     │
          │   (Multi-AZ)    │
          └─────────────────┘
```

這樣改完以後，穩定性確實提升很多。就算某台 EC2 掛了，ALB 會自動把流量導到其他正常的機器。RDS 的 Multi-AZ 也讓我安心不少，至少資料庫不會因為單一 AZ 的問題就死掉。

不過成本也跟著上來了。從原本一個月 40 美金左右，變成將近 250 美金。主管看到帳單有點皺眉，不過考慮到系統穩定性，還是批准了。

### 繼續優化（2021/1 中）

系統跑了一段時間，我發現還有幾個地方可以改善：

**問題 1：Lambda 的嘗試**

有些功能其實不需要一直跑著，像是定期產生報表、處理 webhook 這種偶爾才會觸發的任務。我試著把這些改成 Lambda，省掉一些 EC2 的成本。

一開始寫 Lambda 還蠻不習慣的，尤其是 cold start 的問題。用 Java 寫的 Lambda，冷啟動要好幾秒，根本不能用在需要即時回應的地方。後來看了一些文章，發現大家都推薦用 Node.js 或 Python 寫 Lambda，啟動快很多。

我就把一些簡單的 webhook 用 Node.js 重寫，配合 API Gateway，效果還不錯。

```javascript
// 第一次寫 Lambda，語法跟 Java 差很多
exports.handler = async (event) => {
    const body = JSON.parse(event.body);
    // 做一些簡單的處理
    return {
        statusCode: 200,
        body: JSON.stringify({ message: 'OK' })
    };
};
```

**問題 2：檔案儲存**

韌體更新檔一開始是放在 EC2 的硬碟上，後來發現這樣不太好管理，而且如果要做 CDN 加速也麻煩。就把檔案全部搬到 S3，前面加上 CloudFront。

設定 CloudFront 的時候踩了個坑：一開始不知道要設定 Origin Access Identity，結果 S3 bucket 的檔案都是 public 的，差點被資安部門罵。後來乖乖照著文件設定，只讓 CloudFront 可以存取 S3。

**問題 3：監控和告警**

這個是我一開始最忽略的部分。系統有問題的時候，都是客戶先發現然後通知我們，實在太糗了。

後來認真把 CloudWatch 搞起來，設定了一堆 metric 和 alarm：
- CPU 使用率超過 80%
- RabbitMQ queue 積壓超過 10,000 條
- API 錯誤率超過 1%
- RDS 連線數過多

這些告警會發到 SNS，然後 SNS 再發 email 或簡訊給我們。至少現在有問題我們可以第一時間知道，不用等客戶來抱怨。

### 系統拆分的嘗試（2021/1 下旬）

隨著系統越來越複雜，我開始覺得把所有邏輯都塞在同一個 Spring Boot 應用裡不是辦法。code base 越來越大，改一個小地方要重新部署整個應用。而且不同功能的資源需求也不一樣，全部綁在一起很難調配。

我開始嘗試把系統拆開：
- **API Service**：處理前端和裝置的請求
- **Data Service**：從 RabbitMQ 消費訊息，寫入資料庫
- **Alert Service**：監控裝置狀態，發送告警
- **Report Service**：產生各種報表

每個 service 都是獨立的 Spring Boot 應用，跑在不同的 EC2 上。彼此之間透過 RabbitMQ 或 REST API 溝通。

```
┌──────────────┐     ┌──────────────┐
│ API Service  │     │ Data Service │
│   (EC2 x2)   │     │   (EC2 x3)   │
└──────────────┘     └──────────────┘
        ↓                    ↓
    ┌────────────────────────────┐
    │      RabbitMQ Cluster      │
    └────────────────────────────┘
        ↓                    ↓
┌──────────────┐     ┌──────────────┐
│Alert Service │     │Report Service│
│   (EC2 x1)   │     │   (Lambda)   │
└──────────────┘     └──────────────┘
                           ↓
                    ┌──────────────┐
                    │   RDS MySQL  │
                    └──────────────┘
```

這樣做的好處是：
- 可以針對不同 service 獨立擴展（Data Service 需要很多台，Report Service 一台就夠）
- 部署更靈活，改 Alert Service 不會影響到 API Service
- 出問題的時候比較容易定位是哪個 service 的問題

缺點是：
- 管理變複雜了，要維護更多台機器
- 網路通訊的 overhead 增加
- 要小心處理分散式系統的問題（例如某個 service 掛了怎麼辦）

主管聽到我在做這個，說這不就是所謂的「微服務」嗎？我說概念上差不多，不過我們還沒做到那麼細，應該說是把系統「模組化」，讓不同功能可以獨立部署和擴展。真正的微服務還要考慮很多東西，像是 service discovery、circuit breaker、distributed tracing 這些，我們還沒走到那一步。

不過這也埋下了一個伏筆，未來如果系統繼續成長，往微服務架構發展應該是遲早的事。

## 技術選擇的一些考量

回頭看這三個月做的決策，有些是對的，有些可能還有改進空間。

### EC2 vs Lambda

一開始我以為所有東西都可以用 Lambda，後來發現不是這樣。

**Lambda 適合的場景**：
- 偶爾才會執行的任務（定時 job、webhook）
- 執行時間短（幾秒鐘內）
- 不需要維護狀態

**EC2 適合的場景**：
- 需要長時間運行（像我們的 API server）
- 有複雜的業務邏輯
- 需要大量的記憶體或 CPU

我們的核心 API 還是放在 EC2 上，只有一些輔助功能用 Lambda。

### RDS vs 自建資料庫

一開始為了省錢，資料庫是自己裝在 EC2 上的。後來還是乖乖搬到 RDS：

**RDS 的好處**：
- 自動備份（而且可以 point-in-time recovery）
- Multi-AZ 容錯移轉
- 自動更新 patch
- 監控和告警整合得很好

**成本**：
- 比自建貴大概 30-50%
- 但省下的維護時間絕對值得

### RabbitMQ vs Kafka vs AWS 原生服務

這個我考慮了很久。AWS 有 SQS、SNS、Kinesis 這些 message queue 相關的服務，為什麼還要自己架 RabbitMQ？

主要原因是：
1. **熟悉度**：之前有用過 RabbitMQ，上手比較快
2. **功能**：RabbitMQ 的 routing 機制比較靈活，可以用 topic exchange 做複雜的訊息分派
3. **成本**：自建 RabbitMQ 的成本其實比較便宜（如果訊息量大的話）

如果重來一次，我可能會考慮：
- 訊息量不大的話，用 SQS 比較省事
- 需要處理 streaming data，用 Kinesis
- 需要複雜的 routing，還是 RabbitMQ 比較好

### 監控工具

CloudWatch 其實很夠用了，不過有時候我還是會懷念 Application Insights（.NET 那邊用的）。CloudWatch 的 dashboard 設定起來沒那麼直覺，query log 也沒有很好用。

後來發現可以用 CloudWatch Insights 寫 query 來分析 log，功能還算強大。也試著用了 X-Ray 做 distributed tracing，可以看到 request 經過哪些 service、每個環節花了多少時間，這個蠻有用的。

## 成本的真相

這個是主管最關心的部分。從一開始的 40 美金到後來穩定在 180 美金左右，成長了 4.5 倍。

### 成本分解（2021/2）

- **EC2**（6 台 t3.small/medium）：約 $90
- **RDS**（db.t3.medium, Multi-AZ）：約 $80
- **RabbitMQ**（3 台 t3.micro）：約 $25（其實可以跟其他 service 共用機器）
- **Lambda + API Gateway**：約 $5
- **S3 + CloudFront**：約 $8
- **Data Transfer + 其他**：約 $12

總計：約 $220/月（實際上會有點波動）

### 優化的努力

後來主管要求我想辦法降低成本，我做了幾件事：

1. **Reserved Instances**：對於確定會長期使用的 EC2，買 1 年的 RI，省了大概 30%
2. **Right-sizing**：發現有些 EC2 的 CPU 使用率一直很低，降低規格
3. **清理資源**：刪掉一堆測試用的 snapshot、不用的 EBS
4. **調整 CloudWatch Logs retention**：從 never expire 改成 30 天

最後成本降到 $180 左右，主管勉強接受了。

## 踩過的坑

### 1. RabbitMQ 佇列塞住了（2020/12 下旬）

第一次遇到嚴重問題是在上線測試大約一個禮拜後。某天早上進辦公室，收到一堆 CloudWatch 告警，RabbitMQ 的 memory 使用率飆到 90%，queue 裡面積壓了快三萬條訊息。

我趕快登進去看，發現是 consumer 處理速度跟不上。當時只有一個 consumer 在跑，每收到一條訊息就寫一次資料庫，根本吃不消。

**解決方式**：
1. 緊急加開兩個 consumer instance
2. 改成批次寫入（一次處理 50 筆）
3. 設定 Lazy Queue 和 TTL

### 2. Lambda 冷啟動太慢（2021/1 中）

用 Java 寫的 Lambda，第一次執行要等 3-5 秒。把 webhook 改成 Lambda 後，客戶那邊說 timeout。

試了幾種方法，最後改用 Node.js，冷啟動降到 500ms 以下。

### 3. RDS 連線數爆掉（2021/1 下旬）

Lambda 越寫越多，RDS 連線數突然暴增，超過 max_connections 限制。每個 Lambda instance 都會建立自己的資料庫連線，Lambda 可能同時跑幾十個 instance。

用 RDS Proxy 解決。

### 4. 服務拆開後反而變慢了（2021/2 初）

拆成多個 service 後，response time 從 50ms 變成 150-200ms。因為一個請求要經過多個 service，每次都有網路延遲。

主管問我值得嗎？我說如果只是為了效能，不值得。但考慮到維護性和可擴展性，還是值得的。

## 學到的經驗

### 技術面

1. **雲端原生設計**：從一開始就考慮分散式架構
2. **自動化優先**：Infrastructure as Code（正在學習 Terraform）
3. **監控至上**：沒有監控就是盲目飛行
4. **成本意識**：每個決策都要考慮成本影響

### 團隊面

1. **文件化**：架構圖、API 文件、部署流程都要記錄
2. **知識分享**：定期技術分享會
3. **權責分明**：明確誰負責哪些服務

### 工具面

1. **AWS CLI**：比 Console 更有效率
2. **CloudWatch Insights**：強大的日誌查詢
3. **AWS Cost Explorer**：了解花費在哪裡

## 未來計畫

### 短期（Q1 2022）

1. **容器化**：將服務遷移到 ECS/Fargate
2. **CI/CD**：建立自動化部署流程（CodePipeline）
3. **Infrastructure as Code**：使用 Terraform 管理基礎設施

### 中期（Q2-Q3 2022）

1. **Kubernetes**：評估 EKS
2. **服務網格**：考慮 AWS App Mesh
3. **機器學習**：裝置異常偵測（SageMaker）

### 長期

1. **多區域部署**：提供全球服務
2. **邊緣運算**：使用 AWS IoT Greengrass
3. **無伺服器優先**：能用 Serverless 就不用伺服器

## 給同樣想學 AWS 的工程師建議

### 1. 從小項目開始

不要一開始就想建立複雜架構，從簡單的開始：
- Week 1-2：熟悉 EC2、Security Groups
- Week 3-4：學習 RDS、S3
- Week 5-6：嘗試 Lambda、API Gateway
- Week 7-8：整合多個服務

### 2. 善用免費方案

AWS 免費方案很慷慨：
- EC2 t2.micro：750 小時/月（12 個月）
- RDS db.t2.micro：750 小時/月（12 個月）
- Lambda：每月 100 萬次請求免費
- S3：5 GB 儲存免費

### 3. 重視基礎

在學習高級服務前，先打好基礎：
- VPC 和網路概念
- IAM 權限管理
- CloudWatch 監控

### 4. 實戰學習

光看文件不夠，要動手做：
- 建立測試環境
- 模擬故障情境
- 測試容錯移轉

### 5. 社群與資源

- AWS Documentation（最權威）
- AWS Blog（最新功能）
- AWS re:Invent 影片（深入技術）
- Reddit r/aws（實務經驗）

## 總結

這三個月（2020/12 - 2021/2）真的學到很多。從完全不懂 AWS，到現在可以建立一個相對穩定的系統，充滿挑戰但也很有成就感。

關鍵的收穫：

1. **架構演進**：從單體到分散式，了解什麼時候該拆、什麼時候不該拆
2. **服務選擇**：知道何時該用 EC2、Lambda、RDS
3. **高可用性**：Multi-AZ、Cluster、容錯移轉的實務經驗
4. **效能優化**：Lambda 冷啟動、資料庫優化、批次處理
5. **成本控制**：學會看帳單、Right-sizing、Reserved Instances
6. **監控告警**：CloudWatch、X-Ray 的重要性

對於同樣是從傳統架構轉向雲端的工程師，我的建議是：**不要怕嘗試，但要小心成本**。AWS 提供了大量的服務和彈性，關鍵是選擇適合自己需求的方案。

雲端技術還在快速發展，我也會持續學習和實踐。接下來想深入研究容器化、Kubernetes 和 DevOps 實踐，期待在未來的文章中繼續分享。

感謝這三個月來的學習機會，也感謝所有提供幫助和建議的同事。技術的學習永無止境，我們一起努力！

## 參考架構

最終的系統架構：

```
                          Internet
                             │
                    ┌────────┴────────┐
                    │  Route 53 (DNS) │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │   CloudFront    │
                    │      (CDN)      │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────┴────┐         ┌────┴────┐         ┌────┴────┐
   │   ALB   │         │   ALB   │         │   S3    │
   │(API-1)  │         │(API-2)  │         │(Static) │
   └────┬────┘         └────┬────┘         └─────────┘
        │                   │
   ┌────┴────┐         ┌────┴────┐
   │   EC2   │         │  Lambda │
   │(Core API)│        │(Webhook)│
   └────┬────┘         └────┬────┘
        │                   │
        └────────┬──────────┘
                 │
         ┌───────┴───────┐
         │   RabbitMQ    │
         │    Cluster    │
         │  (3 nodes)    │
         └───────┬───────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───┴───┐   ┌───┴───┐   ┌───┴───┐
│Lambda │   │Lambda │   │Lambda │
│Process│   │Alert  │   │Archive│
└───┬───┘   └───┬───┘   └───┬───┘
    │           │           │
    └───────────┼───────────┘
                │
          ┌─────┴─────┐
          │    RDS    │
          │  (Multi-  │
          │    AZ)    │
          └───────────┘
                │
          ┌─────┴─────┐
          │CloudWatch │
          │ Monitoring│
          └───────────┘
```

這就是我們三個月來建立的系統，從簡單到現在這個樣子。不算完美，但夠穩定、夠用。

## 相關文章

- [AWS EC2 入門：從零開始部署應用服務]({% post_url 2020-12-06-aws-ec2-getting-started %})
- [AWS Lambda + API Gateway 實戰：打造 Serverless REST API]({% post_url 2020-12-14-aws-lambda-api-gateway %})
- [RabbitMQ 實戰：處理 IoT 裝置 Heartbeat]({% post_url 2020-12-21-rabbitmq-iot-heartbeat %})
- [AWS Step Functions：編排複雜的 Serverless 工作流程]({% post_url 2020-12-28-aws-step-functions %})
- [AWS RDS 與 CloudWatch：資料庫管理與監控實戰]({% post_url 2021-01-06-aws-rds-cloudwatch %})
- [AWS S3 與 CloudFront：打造高效能 CDN 架構]({% post_url 2021-01-13-aws-s3-cloudfront %})
- [RabbitMQ 進階：高可用與效能優化實戰]({% post_url 2021-01-20-rabbitmq-ha-performance %})
- [AWS Lambda 冷啟動優化與最佳實踐]({% post_url 2021-01-27-lambda-cold-start-optimization %})
