---
layout: post
title: "Multi-Cloud Observability：跨雲端的統一監控"
date: 2023-12-28 09:00:00 +0800
categories: [Observability, Multi-Cloud]
tags: [Multi-Cloud, AWS, Azure, GCP, Hybrid Cloud, Observability]
---

「我們的服務同時跑在 AWS、Azure 和 GCP 上，該怎麼監控？」

歡迎來到 **Multi-Cloud Observability** 的世界。

## 為什麼需要 Multi-Cloud？

### 常見場景

1. **避免供應商鎖定（Vendor Lock-in）**
   - 不依賴單一雲端服務商
   - 可以隨時切換到成本更低的方案

2. **就近服務（Geo-Distribution）**
   - AWS 在美國
   - Azure 在歐洲
   - GCP 在亞洲

3. **災難復原（Disaster Recovery）**
   - 主站在 AWS
   - 備援在 Azure

4. **合併後的系統（M&A）**
   - 公司 A 使用 AWS
   - 公司 B 使用 GCP
   - 合併後需要整合監控

5. **特定服務需求**
   - AI/ML 使用 GCP（TensorFlow）
   - 大數據使用 AWS（EMR）
   - DevOps 使用 Azure（Azure DevOps）

## Multi-Cloud Observability 挑戰

### 挑戰 1：不同的 Metrics 格式

```
AWS CloudWatch:
{
  "Namespace": "AWS/EC2",
  "MetricName": "CPUUtilization",
  "Dimensions": [{"Name": "InstanceId", "Value": "i-1234567890"}],
  "Value": 75.5
}

Azure Monitor:
{
  "name": "Percentage CPU",
  "resourceId": "/subscriptions/.../Microsoft.Compute/virtualMachines/vm1",
  "value": 75.5
}

GCP Cloud Monitoring:
{
  "metric": {
    "type": "compute.googleapis.com/instance/cpu/utilization",
    "labels": {"instance_id": "1234567890"}
  },
  "points": [{"value": {"doubleValue": 0.755}}]
}
```

### 挑戰 2：不同的 Log 格式

```
AWS CloudWatch Logs:
{
  "timestamp": 1703239200000,
  "message": "Request processed",
  "logGroup": "/aws/lambda/my-function",
  "logStream": "2023/12/22/[$LATEST]abcdef"
}

Azure Log Analytics:
{
  "TimeGenerated": "2023-12-22T12:00:00Z",
  "Message": "Request processed",
  "Computer": "vm1",
  "ResourceId": "/subscriptions/..."
}

GCP Cloud Logging:
{
  "timestamp": "2023-12-22T12:00:00Z",
  "textPayload": "Request processed",
  "resource": {"type": "gce_instance", "labels": {"instance_id": "1234"}}
}
```

### 挑戰 3：分散的監控工具

```
AWS:
- CloudWatch (Metrics, Logs, Alarms)
- X-Ray (Distributed Tracing)
- CloudTrail (Audit Logs)

Azure:
- Azure Monitor (Metrics, Logs)
- Application Insights (APM)
- Log Analytics

GCP:
- Cloud Monitoring (Metrics)
- Cloud Logging (Logs)
- Cloud Trace (Distributed Tracing)
```

## Multi-Cloud Observability 架構

### 架構 1：集中式 Observability Platform

```
┌─────────────────────────────────────────────┐
│              Application Layer               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │AWS Apps │  │Azure Apps│  │GCP Apps │     │
│  └────┬────┘  └────┬─────┘  └────┬────┘     │
└───────┼────────────┼─────────────┼──────────┘
        │            │             │
        │ OpenTelemetry            │
        │            │             │
        ▼            ▼             ▼
┌─────────────────────────────────────────────┐
│       Centralized Observability Platform     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │Prometheus│  │  Loki    │  │  Jaeger  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │          Grafana (統一介面)          │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 架構 2：混合式監控

```
┌────────────┐    ┌────────────┐    ┌────────────┐
│  AWS       │    │  Azure     │    │  GCP       │
│  CloudWatch│    │  Monitor   │    │  Monitoring│
└──────┬─────┘    └──────┬─────┘    └──────┬─────┘
       │                 │                  │
       │ Export          │ Export           │ Export
       │                 │                  │
       ▼                 ▼                  ▼
┌─────────────────────────────────────────────────┐
│         Data Aggregation Layer                   │
│  ┌─────────────────────────────────────────┐    │
│  │  Prometheus Remote Write                │    │
│  │  (收集各雲端 Metrics)                    │    │
│  └─────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────┐    │
│  │  Loki / Elasticsearch                   │    │
│  │  (收集各雲端 Logs)                       │    │
│  └─────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────┐    │
│  │  Jaeger                                 │    │
│  │  (收集各雲端 Traces)                     │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
                      │
                      ▼
              ┌──────────────┐
              │   Grafana    │
              │ (統一監控面板) │
              └──────────────┘
```

## 實作 1：使用 OpenTelemetry 統一資料收集

### OpenTelemetry Collector 設定

```yaml
# otel-collector-config.yaml
receivers:
  # 接收應用程式 Metrics
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  
  # 接收 Prometheus Metrics
  prometheus:
    config:
      scrape_configs:
        - job_name: 'aws-apps'
          static_configs:
            - targets: ['aws-app-1:8080', 'aws-app-2:8080']
        - job_name: 'azure-apps'
          static_configs:
            - targets: ['azure-app-1:8080']
        - job_name: 'gcp-apps'
          static_configs:
            - targets: ['gcp-app-1:8080']

processors:
  # 批次處理
  batch:
    timeout: 10s
    send_batch_size: 1024
  
  # 加入雲端標籤
  resource:
    attributes:
      - key: cloud.provider
        from_attribute: cloud.provider
        action: insert
      - key: cloud.region
        from_attribute: cloud.region
        action: insert
  
  # 正規化標籤名稱
  attributes:
    actions:
      # AWS
      - key: aws.instance.id
        pattern: ^i-.*
        action: update
        new_name: instance.id
      # Azure
      - key: azure.vm.id
        pattern: ^/subscriptions/.*
        action: update
        new_name: instance.id
      # GCP
      - key: gcp.instance.id
        pattern: ^[0-9]+$
        action: update
        new_name: instance.id

exporters:
  # 匯出到 Prometheus
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"
    resource_to_telemetry_conversion:
      enabled: true
  
  # 匯出到 Loki
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
  
  # 匯出到 Jaeger
  jaeger:
    endpoint: "jaeger:14250"
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch, resource, attributes]
      exporters: [prometheusremotewrite]
    
    logs:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [loki]
    
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [jaeger]
```

### 應用程式整合（Java）

```java
// Application.java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import io.opentelemetry.exporter.otlp.metrics.OtlpGrpcMetricExporter;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;

public class ObservabilityConfig {
    
    public static OpenTelemetry initOpenTelemetry(String cloudProvider) {
        // 設定 Resource Attributes
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.builder()
                .put(ResourceAttributes.SERVICE_NAME, "my-service")
                .put(ResourceAttributes.SERVICE_VERSION, "1.0.0")
                .put("cloud.provider", cloudProvider)  // aws, azure, gcp
                .put("cloud.region", getCloudRegion())
                .put("deployment.environment", getEnvironment())
                .build()));
        
        // 設定 OTLP Exporter
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build();
        
        OtlpGrpcMetricExporter metricExporter = OtlpGrpcMetricExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build();
        
        return OpenTelemetrySdk.builder()
            .setTracerProvider(SdkTracerProvider.builder()
                .setResource(resource)
                .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
                .build())
            .setMeterProvider(SdkMeterProvider.builder()
                .setResource(resource)
                .registerMetricReader(PeriodicMetricReader.builder(metricExporter)
                    .setInterval(Duration.ofSeconds(60))
                    .build())
                .build())
            .buildAndRegisterGlobal();
    }
    
    private static String getCloudRegion() {
        // 自動偵測雲端 Region
        if (isAWS()) return System.getenv("AWS_REGION");
        if (isAzure()) return System.getenv("AZURE_REGION");
        if (isGCP()) return System.getenv("GCP_REGION");
        return "unknown";
    }
    
    private static boolean isAWS() {
        return System.getenv("AWS_EXECUTION_ENV") != null;
    }
    
    private static boolean isAzure() {
        return System.getenv("WEBSITE_SITE_NAME") != null;
    }
    
    private static boolean isGCP() {
        return System.getenv("GOOGLE_CLOUD_PROJECT") != null;
    }
}
```

## 實作 2：整合雲端原生 Metrics

### AWS CloudWatch → Prometheus

```yaml
# cloudwatch-exporter.yml
region: us-west-2
metrics:
  - aws_namespace: AWS/EC2
    aws_metric_name: CPUUtilization
    aws_dimensions: [InstanceId]
    aws_statistics: [Average]
    period_seconds: 300
    range_seconds: 600
    set_timestamp: true
    
  - aws_namespace: AWS/RDS
    aws_metric_name: DatabaseConnections
    aws_dimensions: [DBInstanceIdentifier]
    aws_statistics: [Average]
    
  - aws_namespace: AWS/ELB
    aws_metric_name: HealthyHostCount
    aws_dimensions: [LoadBalancerName, AvailabilityZone]
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'aws-cloudwatch'
    static_configs:
      - targets: ['cloudwatch-exporter:9106']
    relabel_configs:
      # 加入 cloud.provider 標籤
      - source_labels: []
        target_label: cloud_provider
        replacement: aws
```

### Azure Monitor → Prometheus

```yaml
# azure-exporter.yml
subscriptions:
  - subscription_id: "your-subscription-id"
    resource_groups:
      - resource_group: "production-rg"
        resources:
          - resource: "vm1"
            metrics:
              - name: "Percentage CPU"
                aggregation: "Average"
              - name: "Network In Total"
                aggregation: "Total"
```

### GCP Cloud Monitoring → Prometheus

```yaml
# stackdriver-exporter.yml
project_id: "your-project-id"
metrics:
  - name: compute.googleapis.com/instance/cpu/utilization
    metric_kind: GAUGE
    value_type: DOUBLE
    
  - name: compute.googleapis.com/instance/disk/read_bytes_count
    metric_kind: DELTA
    value_type: INT64
```

## 實作 3：統一 Log 格式

### Fluentd 正規化設定

```ruby
# fluentd.conf
<source>
  @type forward
  port 24224
</source>

# AWS CloudWatch Logs
<filter aws.**>
  @type record_transformer
  <record>
    cloud_provider aws
    cloud_region ${record["awsRegion"]}
    log_group ${record["logGroup"]}
    # 正規化欄位名稱
    timestamp ${record["timestamp"]}
    message ${record["message"]}
    level ${record["level"] || "INFO"}
  </record>
  remove_keys awsRegion,logGroup
</filter>

# Azure Log Analytics
<filter azure.**>
  @type record_transformer
  <record>
    cloud_provider azure
    cloud_region ${record["Location"]}
    resource_group ${record["ResourceGroup"]}
    # 正規化欄位名稱
    timestamp ${record["TimeGenerated"]}
    message ${record["Message"]}
    level ${record["Level"] || "INFO"}
  </record>
  remove_keys Location,ResourceGroup,TimeGenerated,Message,Level
</filter>

# GCP Cloud Logging
<filter gcp.**>
  @type record_transformer
  <record>
    cloud_provider gcp
    cloud_region ${record["resource"]["labels"]["zone"]}
    project_id ${record["resource"]["labels"]["project_id"]}
    # 正規化欄位名稱
    timestamp ${record["timestamp"]}
    message ${record["textPayload"] || record["jsonPayload"]["message"]}
    level ${record["severity"] || "INFO"}
  </record>
</filter>

# 統一輸出
<match **>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix unified-logs
  
  <buffer>
    @type file
    path /var/log/fluentd-buffers/unified
    flush_interval 5s
  </buffer>
</match>
```

## 實作 4：統一 Dashboard

### Grafana Multi-Cloud Dashboard

```json
{
  "dashboard": {
    "title": "Multi-Cloud Overview",
    "panels": [
      {
        "title": "CPU Usage by Cloud Provider",
        "targets": [
          {
            "expr": "avg(cpu_usage{cloud_provider=\"aws\"}) by (instance)",
            "legendFormat": "AWS - {{instance}}"
          },
          {
            "expr": "avg(cpu_usage{cloud_provider=\"azure\"}) by (instance)",
            "legendFormat": "Azure - {{instance}}"
          },
          {
            "expr": "avg(cpu_usage{cloud_provider=\"gcp\"}) by (instance)",
            "legendFormat": "GCP - {{instance}}"
          }
        ]
      },
      {
        "title": "Request Rate by Region",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (cloud_provider, cloud_region)",
            "legendFormat": "{{cloud_provider}} - {{cloud_region}}"
          }
        ]
      },
      {
        "title": "Error Rate Comparison",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (cloud_provider) / sum(rate(http_requests_total[5m])) by (cloud_provider)",
            "legendFormat": "{{cloud_provider}}"
          }
        ]
      },
      {
        "title": "Cost per Cloud Provider",
        "targets": [
          {
            "expr": "sum(cloud_cost_hourly) by (cloud_provider)",
            "legendFormat": "{{cloud_provider}}"
          }
        ]
      }
    ]
  }
}
```

## 實作 5：跨雲端 Trace Propagation

### OpenTelemetry Context Propagation

```java
// AWS Lambda → Azure Function → GCP Cloud Run

// 1. AWS Lambda (發起請求)
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.propagation.TextMapSetter;

public class AWSLambdaHandler {
    
    public void handleRequest(Map<String, String> input) {
        Span span = tracer.spanBuilder("aws-lambda-handler")
            .setAttribute("cloud.provider", "aws")
            .setAttribute("faas.name", context.getFunctionName())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 注入 Trace Context 到 HTTP Headers
            Map<String, String> headers = new HashMap<>();
            openTelemetry.getPropagators().getTextMapPropagator()
                .inject(Context.current(), headers, new MapSetter());
            
            // 呼叫 Azure Function
            callAzureFunction(headers);
        } finally {
            span.end();
        }
    }
}

// 2. Azure Function (中間服務)
public class AzureFunctionHandler {
    
    public HttpResponseMessage run(HttpRequestMessage<String> request) {
        // 提取 Trace Context
        Context extractedContext = openTelemetry.getPropagators()
            .getTextMapPropagator()
            .extract(Context.current(), request.getHeaders(), new MapGetter());
        
        Span span = tracer.spanBuilder("azure-function-handler")
            .setParent(extractedContext)
            .setAttribute("cloud.provider", "azure")
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 呼叫 GCP Cloud Run
            callGCPCloudRun();
        } finally {
            span.end();
        }
    }
}

// 3. GCP Cloud Run (最終服務)
@RestController
public class GCPCloudRunController {
    
    @GetMapping("/process")
    public String process(@RequestHeader Map<String, String> headers) {
        // 提取 Trace Context
        Context extractedContext = openTelemetry.getPropagators()
            .getTextMapPropagator()
            .extract(Context.current(), headers, new MapGetter());
        
        Span span = tracer.spanBuilder("gcp-cloudrun-process")
            .setParent(extractedContext)
            .setAttribute("cloud.provider", "gcp")
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            return processRequest();
        } finally {
            span.end();
        }
    }
}
```

### Jaeger 完整 Trace 視圖

```
Trace ID: abc123
Duration: 850ms

┌─────────────────────────────────────────────────┐
│ aws-lambda-handler (AWS)              850ms     │
│ ┌─────────────────────────────────────────────┐ │
│ │ azure-function-handler (Azure)      650ms   │ │
│ │ ┌─────────────────────────────────────────┐ │ │
│ │ │ gcp-cloudrun-process (GCP)    450ms     │ │ │
│ │ └─────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘

Tags:
- cloud.provider: aws → azure → gcp
- cloud.region: us-west-2 → westeurope → asia-east1
```

## 成本比較與優化

### 雲端原生 vs. 自建 Observability

| 方案 | 成本（每月） | 優點 | 缺點 |
|------|-------------|------|------|
| **AWS CloudWatch + Azure Monitor + GCP Cloud Monitoring** | $5,000 - $15,000 | 原生整合，易用 | 成本高，數據分散 |
| **Datadog / New Relic（SaaS）** | $3,000 - $10,000 | 統一平台，功能完整 | 成本高，供應商鎖定 |
| **自建（Prometheus + Loki + Jaeger）** | $1,000 - $3,000 | 成本低，完全掌控 | 需要維運團隊 |

### 混合式方案（建議）

```
關鍵 Metrics → 雲端原生（高可用性）
應用 Metrics → 自建 Prometheus（成本優化）
Logs → 短期雲端原生 + 長期 S3/GCS（成本優化）
Traces → 自建 Jaeger（成本優化）
```

## 最佳實踐

### 1. 標準化標籤

```yaml
# 所有服務必須包含的標籤
required_labels:
  - cloud.provider      # aws, azure, gcp
  - cloud.region        # us-west-2, westeurope, asia-east1
  - environment         # dev, staging, production
  - service.name        # order-service, payment-service
  - service.version     # 1.0.0
  - team                # backend, frontend, platform
```

### 2. 統一 SLO

```yaml
# SLO 定義（跨所有雲端）
slos:
  availability:
    target: 99.9%
    measurement: |
      sum(rate(http_requests_total{status=~"2..|3.."}[30d]))
      /
      sum(rate(http_requests_total[30d]))
  
  latency:
    target: 95% < 500ms
    measurement: |
      histogram_quantile(0.95,
        sum(rate(http_request_duration_seconds_bucket[30d])) by (le)
      ) < 0.5
```

### 3. 災難復原驗證

```bash
# 定期測試跨雲端 Failover
./scripts/test-failover.sh

# 步驟：
# 1. 模擬 AWS 故障
# 2. 流量自動切換到 Azure
# 3. 驗證監控數據完整性
# 4. 驗證 Trace 連續性
```

## 總結

Multi-Cloud Observability 的關鍵：

1. **統一資料格式**：使用 OpenTelemetry 標準化
2. **集中式平台**：Prometheus + Loki + Jaeger
3. **標籤標準化**：cloud.provider, cloud.region 必備
4. **成本優化**：混合式方案（雲端原生 + 自建）
5. **自動化**：使用 IaC 管理所有設定

**Multi-Cloud 不是終點，而是起點。統一的 Observability 才能讓你真正掌控分散式系統。**

---

**相關文章：**
- 《Week 23: OpenTelemetry 簡介》
- 《Week 50: Observability 成本優化》
- 《Week 51: Observability as Code》
