---
layout: post
title: "多租戶環境的可觀測性與資料隔離"
date: 2023-09-01 09:00:00 +0800
categories: [Observability, Multi-Tenancy]
tags: [Multi-Tenancy, Security, Data Isolation, SaaS]
---

當你在建立 SaaS 產品時，會遇到一個挑戰：**多個租戶共享同一套系統，但可觀測性資料必須隔離**。

今天我們來談談如何在多租戶環境中實作可觀測性。

## 多租戶的挑戰

### 問題 1：資料混在一起

所有租戶的 Logs、Metrics、Traces 都在同一個系統中：

```
Elasticsearch:
  tenant_a: 10,000 logs
  tenant_b: 100,000 logs  ← B 的資料淹沒了 A
  tenant_c: 1,000 logs
```

### 問題 2：成本分攤

租戶 B 產生了 90% 的資料，但成本是平均分攤的嗎？

### 問題 3：資料安全

租戶 A 不應該看到租戶 B 的資料。

## 多租戶的設計模式

### 模式 1：Database per Tenant

每個租戶有自己的資料庫（或 Elasticsearch Index）。

#### Elasticsearch

```
jaeger-tenant-a-2023-09-01
jaeger-tenant-b-2023-09-01
jaeger-tenant-c-2023-09-01
```

#### 優點
- 資料完全隔離
- 容易刪除租戶資料（合規要求）

#### 缺點
- 資源浪費（小租戶也要一個完整的 Index）
- 管理複雜（數百個 Index）

### 模式 2：Shared Database with tenant_id

所有租戶共享一個資料庫，但每筆資料都有 `tenant_id`。

#### Elasticsearch

```json
{
  "tenant_id": "tenant-a",
  "trace_id": "abc123",
  "service": "order-service",
  "duration": 120
}
```

#### 優點
- 資源利用率高
- 管理簡單

#### 缺點
- 需要在查詢時過濾 `tenant_id`
- 有資料洩漏的風險

### 模式 3：Hybrid（混合模式）

- 大租戶：獨立的 Elasticsearch Index
- 小租戶：共享一個 Index

## 實作：在 Trace 中加入 tenant_id

### Java + OpenTelemetry

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.context.Context;

@Component
public class TenantInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 從 Header 或 JWT 中提取 tenant_id
        String tenantId = extractTenantId(request);
        
        // 加入到當前的 Span
        Span span = Span.current();
        span.setAttribute("tenant.id", tenantId);
        span.setAttribute("tenant.name", getTenantName(tenantId));
        
        // 也可以加入到 MDC（用於 Logs）
        MDC.put("tenant_id", tenantId);
        
        return true;
    }
    
    private String extractTenantId(HttpServletRequest request) {
        // 從 Header 提取
        String tenantId = request.getHeader("X-Tenant-ID");
        if (tenantId != null) {
            return tenantId;
        }
        
        // 從 JWT 提取
        String jwt = request.getHeader("Authorization");
        if (jwt != null) {
            return extractTenantFromJWT(jwt);
        }
        
        // 從子網域提取（例如：tenant-a.example.com）
        String host = request.getServerName();
        if (host.contains(".")) {
            return host.split("\\.")[0];
        }
        
        return "unknown";
    }
}
```

### .NET + OpenTelemetry

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Http;

public class TenantMiddleware
{
    private readonly RequestDelegate _next;

    public TenantMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var tenantId = ExtractTenantId(context);
        
        var activity = Activity.Current;
        if (activity != null)
        {
            activity.SetTag("tenant.id", tenantId);
            activity.SetTag("tenant.name", GetTenantName(tenantId));
        }
        
        // 也可以加入到 LogContext
        using (Serilog.Context.LogContext.PushProperty("tenant_id", tenantId))
        {
            await _next(context);
        }
    }
    
    private string ExtractTenantId(HttpContext context)
    {
        // 從 Header 提取
        if (context.Request.Headers.TryGetValue("X-Tenant-ID", out var tenantId))
        {
            return tenantId.ToString();
        }
        
        // 從 JWT 提取
        if (context.Request.Headers.TryGetValue("Authorization", out var jwt))
        {
            return ExtractTenantFromJWT(jwt.ToString());
        }
        
        // 從子網域提取
        var host = context.Request.Host.Host;
        if (host.Contains("."))
        {
            return host.Split('.')[0];
        }
        
        return "unknown";
    }
}

// 註冊 Middleware
app.UseMiddleware<TenantMiddleware>();
```

## 實作：在 Logs 中加入 tenant_id

### Logback 配置

```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>tenant_id</includeMdcKeyName>
            <includeMdcKeyName>trace_id</includeMdcKeyName>
        </encoder>
    </appender>
</configuration>
```

輸出的 Log：

```json
{
  "timestamp": "2023-09-01T10:15:23.123Z",
  "message": "Order created",
  "tenant_id": "tenant-a",
  "trace_id": "abc123"
}
```

## 實作：在 Metrics 中加入 tenant_id

### Prometheus

```java
import io.prometheus.client.Counter;

Counter orderCounter = Counter.build()
    .name("orders_total")
    .help("Total number of orders")
    .labelNames("tenant_id", "status")
    .register();

orderCounter.labels(tenantId, "created").inc();
```

查詢時，可以過濾特定租戶：

```promql
rate(orders_total{tenant_id="tenant-a"}[5m])
```

## 查詢時的資料隔離

### Jaeger

在 Jaeger 中，用 `tenant.id` 標籤過濾：

```
service=order-service tenant.id=tenant-a
```

但是，這依賴使用者**手動輸入**，容易出錯。

### 解決方案：Jaeger UI 的 Reverse Proxy

在 Jaeger UI 前面加一個 Reverse Proxy，自動加入 `tenant.id` 過濾條件。

**Nginx 配置**：

```nginx
location /jaeger/ {
    set $tenant_id '';
    
    # 從 Cookie 或 JWT 提取 tenant_id
    if ($http_cookie ~* "tenant_id=([^;]+)") {
        set $tenant_id $1;
    }
    
    # 重寫 Query String，加入 tenant.id 過濾
    set $args "${args}&tags={\"tenant.id\":\"${tenant_id}\"}";
    
    proxy_pass http://jaeger-query:16686;
}
```

這樣使用者就**無法**看到其他租戶的 Trace。

### Grafana

在 Grafana 中，用 **Data Source Variable** 過濾。

**配置**：

```yaml
datasources:
  - name: Prometheus-TenantA
    type: prometheus
    url: http://prometheus:9090
    jsonData:
      httpHeaderName1: X-Tenant-ID
    secureJsonData:
      httpHeaderValue1: tenant-a
```

或者用 Grafana 的 **Organization** 功能：
- 每個租戶一個 Organization
- 每個 Organization 有自己的 Dashboard 和 Data Source

## 成本分攤

### 計算每個租戶的資料量

#### Logs（Elasticsearch）

```json
GET logs-*/_search
{
  "size": 0,
  "aggs": {
    "per_tenant": {
      "terms": {
        "field": "tenant_id",
        "size": 1000
      },
      "aggs": {
        "total_size": {
          "sum": {
            "field": "_size"
          }
        }
      }
    }
  }
}
```

#### Traces（Jaeger）

```promql
sum by (tenant_id) (increase(spans_total[30d]))
```

#### Metrics（Prometheus）

```promql
sum by (tenant_id) (increase(samples_total[30d]))
```

### 計費

```
tenant-a:
  Logs: 10 GB × $0.10/GB = $1.00
  Traces: 1,000,000 spans × $0.01/1000 = $10.00
  Metrics: 100,000,000 samples × $0.01/1M = $1.00
  Total: $12.00

tenant-b:
  Logs: 100 GB × $0.10/GB = $10.00
  Traces: 10,000,000 spans × $0.01/1000 = $100.00
  Metrics: 1,000,000,000 samples × $0.01/1M = $10.00
  Total: $120.00
```

## 資料保留策略

不同的租戶可能有不同的資料保留需求。

### Elasticsearch ILM

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "delete": {
        "min_age": "30d",  // 預設保留 30 天
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

但是，如果租戶 A 要求保留 90 天，租戶 B 只要 7 天呢？

### 解決方案：Index per Tenant

```
logs-tenant-a-2023-09-01  (保留 90 天)
logs-tenant-b-2023-09-01  (保留 7 天)
```

每個 Index 有自己的 ILM Policy。

## 效能隔離

### 問題

租戶 B 的流量很大，拖慢了租戶 A 的查詢。

### 解決方案 1：Elasticsearch 的 Search Throttling

```json
PUT logs-tenant-b/_settings
{
  "index.search.throttled": true
}
```

### 解決方案 2：獨立的 Elasticsearch Cluster

- 大租戶：獨立的 Cluster
- 小租戶：共享一個 Cluster

### 解決方案 3：Kubernetes Resource Limits

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger-query-tenant-a
spec:
  template:
    spec:
      containers:
      - name: jaeger-query
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
```

## 合規要求

### GDPR：資料刪除

當租戶要求刪除資料時，必須確保所有資料都被刪除。

#### Elasticsearch

```bash
# 刪除特定租戶的所有資料
POST logs-*/_delete_by_query
{
  "query": {
    "term": {
      "tenant_id": "tenant-a"
    }
  }
}

POST traces-*/_delete_by_query
{
  "query": {
    "term": {
      "tenant.id": "tenant-a"
    }
  }
}
```

#### Prometheus

Prometheus 不支援刪除單筆資料，只能刪除整個 Time Series：

```bash
curl -X POST http://prometheus:9090/api/v1/admin/tsdb/delete_series -d 'match[]={tenant_id="tenant-a"}'
```

但是，這只是標記為刪除，實際刪除要等到下次 Compaction。

### 稽核日誌

記錄誰查詢了哪個租戶的資料。

```java
@Aspect
@Component
public class AuditAspect {
    @Around("execution(* com.example.JaegerService.query(..))")
    public Object audit(ProceedingJoinPoint joinPoint) throws Throwable {
        String user = getCurrentUser();
        String tenantId = (String) joinPoint.getArgs()[0];
        
        auditLog.info("User {} queried tenant {}", user, tenantId);
        
        return joinPoint.proceed();
    }
}
```

## 實戰案例

### 案例：SaaS 平台

**租戶數量**：1,000

**分類**：
- Enterprise（10 個）：每個租戶有獨立的 Elasticsearch Cluster
- Business（100 個）：共享 3 個 Elasticsearch Cluster
- Startup（890 個）：共享 1 個 Elasticsearch Cluster

**資料保留**：
- Enterprise：90 天
- Business：30 天
- Startup：7 天

**成本**：
- Enterprise：每個租戶 $1,000/month
- Business：每個租戶 $100/month
- Startup：每個租戶 $10/month

## 建議

### 1. 一開始就加入 tenant_id

不要等到有多租戶需求才加，從一開始就加入 `tenant_id`。

### 2. 定期檢查資料洩漏

寫一個腳本，檢查是否有查詢沒有加入 `tenant_id` 過濾條件。

### 3. 用 Namespace 隔離 Kubernetes 資源

```
namespace: tenant-a
namespace: tenant-b
```

這樣可以用 Kubernetes 的 RBAC 控制存取權限。

---

**多租戶環境的可觀測性，不只是技術問題，還是成本和合規問題。**

好的設計可以讓你在保持資料隔離的同時，降低營運成本。
