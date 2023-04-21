---
layout: post
title: "Logstash Filters：解析、轉換、豐富你的日誌"
date: 2023-04-21 09:00:00 +0800
categories: [Observability, Logging]
tags: [Logstash, Grok, Filters, ETL, Data Processing]
---

上週我們談了結構化日誌的重要性。但現實是：**你的系統裡有 80% 的日誌都是非結構化的純文字。**

今天我們來看 Logstash 如何把這些「垃圾」變成「寶藏」。

## Logstash 的角色：ETL Pipeline

> **基礎知識**：我們在《Week 14: ELK Stack 入門》已經介紹過 Logstash 的基本概念。這裡專注於 Filter 的進階應用。

Logstash 的核心功能就是 **Extract, Transform, Load**：

```
輸入 → 過濾 → 輸出
Input → Filter → Output
```

### 基本架構（快速回顧）

```ruby
input {
  # 從哪裡收集日誌
  file { path => "/var/log/nginx/access.log" }
  tcp { port => 5000 }
}

filter {
  # 如何處理日誌
  grok { ... }
  mutate { ... }
  date { ... }
}

output {
  # 輸出到哪裡
  elasticsearch { ... }
  stdout { codec => rubydebug }
}
```

今天我們專注在 **Filter** 部分。

## Grok：強大的文字解析工具

Grok 是 Logstash 最重要的過濾器，用來解析非結構化文字。

### Grok 語法基礎

Grok 的語法是：`%{PATTERN:field_name}`

例如：

```ruby
filter {
  grok {
    match => { "message" => "%{IP:client_ip} %{WORD:method} %{URIPATHPARAM:request}" }
  }
}
```

輸入：
```
192.168.1.100 GET /api/orders?id=123
```

輸出：
```json
{
  "client_ip": "192.168.1.100",
  "method": "GET",
  "request": "/api/orders?id=123"
}
```

### 內建的 Grok Patterns

Logstash 內建了 120+ 個常用 Pattern：

| Pattern | 說明 | 範例 |
|---------|------|------|
| `%{IP}` | IP 位址 | `192.168.1.100` |
| `%{HOSTNAME}` | 主機名稱 | `server-01.example.com` |
| `%{TIMESTAMP_ISO8601}` | ISO 時間 | `2023-04-21T10:15:23Z` |
| `%{NUMBER}` | 數字（字串） | `123` |
| `%{INT}` | 整數（字串） | `123` |
| `%{WORD}` | 單字 | `ERROR` |
| `%{DATA}` | 任意文字（非貪婪） | `any text` |
| `%{GREEDYDATA}` | 任意文字（貪婪） | `any text...` |

完整列表：https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns

## 實戰案例 1：解析 Java Exception Stack Trace

### 原始日誌

```
2023-04-21 10:15:23 ERROR [order-service] Failed to process order
java.lang.NullPointerException: Cannot invoke "User.getId()" because "user" is null
	at com.example.OrderService.createOrder(OrderService.java:45)
	at com.example.OrderController.handleRequest(OrderController.java:23)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
```

### Grok 設定

```ruby
filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:service}\] %{GREEDYDATA:error_message}"
    }
  }
  
  # 解析 stack trace（多行日誌）
  if [level] == "ERROR" {
    grok {
      match => {
        "message" => "(?<exception_class>[\w\.]+Exception): %{GREEDYDATA:exception_message}"
      }
    }
  }
  
  # 提取第一行的檔案名稱和行號
  grok {
    match => {
      "message" => "at %{DATA:exception_method}\(%{DATA:exception_file}:%{INT:exception_line:int}\)"
    }
  }
}
```

### 輸出

```json
{
  "timestamp": "2023-04-21 10:15:23",
  "level": "ERROR",
  "service": "order-service",
  "error_message": "Failed to process order",
  "exception_class": "java.lang.NullPointerException",
  "exception_message": "Cannot invoke \"User.getId()\" because \"user\" is null",
  "exception_method": "com.example.OrderService.createOrder",
  "exception_file": "OrderService.java",
  "exception_line": 45
}
```

現在你可以在 Kibana 查詢：
```
exception_file:"OrderService.java" AND exception_line:45
```

## 實戰案例 2：解析 Nginx Access Log

### 原始日誌（Combined Log Format）

```
192.168.1.100 - - [21/Apr/2023:10:15:23 +0000] "GET /api/orders?page=1 HTTP/1.1" 200 1234 "https://example.com" "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
```

### Logstash 設定

```ruby
filter {
  grok {
    match => {
      "message" => '%{IPORHOST:client_ip} - - \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}" %{INT:status_code:int} %{INT:bytes:int} "%{DATA:referrer}" "%{DATA:user_agent}"'
    }
  }
  
  # 轉換時間格式
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }
  
  # 解析 URL 參數
  kv {
    source => "request"
    field_split => "&?"
    value_split => "="
  }
  
  # 解析 User Agent
  useragent {
    source => "user_agent"
    target => "ua"
  }
  
  # GeoIP 解析
  geoip {
    source => "client_ip"
    target => "geoip"
  }
  
  # 判斷是否為爬蟲
  if [ua][name] =~ /bot|crawler|spider/i {
    mutate {
      add_field => { "is_bot" => true }
    }
  } else {
    mutate {
      add_field => { "is_bot" => false }
    }
  }
  
  # 判斷 HTTP 狀態類型
  if [status_code] >= 500 {
    mutate { add_field => { "status_type" => "server_error" } }
  } else if [status_code] >= 400 {
    mutate { add_field => { "status_type" => "client_error" } }
  } else if [status_code] >= 300 {
    mutate { add_field => { "status_type" => "redirect" } }
  } else if [status_code] >= 200 {
    mutate { add_field => { "status_type" => "success" } }
  }
}
```

### 輸出

```json
{
  "@timestamp": "2023-04-21T10:15:23.000Z",
  "client_ip": "192.168.1.100",
  "method": "GET",
  "request": "/api/orders",
  "http_version": "1.1",
  "status_code": 200,
  "status_type": "success",
  "bytes": 1234,
  "referrer": "https://example.com",
  "user_agent": "Mozilla/5.0 ...",
  "page": "1",
  "ua": {
    "name": "Chrome",
    "version": "112.0",
    "os": "Windows 10",
    "device": "Desktop"
  },
  "geoip": {
    "country_name": "Taiwan",
    "city_name": "Taipei",
    "location": {
      "lat": 25.0330,
      "lon": 121.5654
    }
  },
  "is_bot": false
}
```

## Mutate：欄位轉換與處理

Mutate 是 Logstash 的「瑞士刀」，用來修改、新增、刪除欄位。

### 1. 重新命名欄位

```ruby
filter {
  mutate {
    rename => {
      "old_field_name" => "new_field_name"
    }
  }
}
```

### 2. 新增欄位

```ruby
filter {
  mutate {
    add_field => {
      "environment" => "production"
      "version" => "1.2.3"
    }
  }
}
```

### 3. 刪除欄位

```ruby
filter {
  mutate {
    remove_field => ["unnecessary_field", "temp_field"]
  }
}
```

### 4. 轉換類型

```ruby
filter {
  mutate {
    convert => {
      "status_code" => "integer"
      "duration" => "float"
      "is_active" => "boolean"
    }
  }
}
```

### 5. 字串處理

```ruby
filter {
  mutate {
    # 轉大寫
    uppercase => ["method"]
    
    # 轉小寫
    lowercase => ["user_id"]
    
    # 移除空白
    strip => ["message"]
    
    # 分割字串成陣列
    split => { "tags" => "," }
    
    # 取代字串
    gsub => [
      "message", "/", "_"  # 把 / 取代成 _
    ]
  }
}
```

## Date：時間解析

Logstash 預設會自動加上 `@timestamp` 欄位（收到日誌的時間），但這不是日誌真正產生的時間。

我們需要用 `date` 過濾器來解析日誌中的時間戳記。

### 範例

```ruby
filter {
  date {
    match => [
      "timestamp",
      "yyyy-MM-dd HH:mm:ss",
      "ISO8601",
      "UNIX",
      "dd/MMM/yyyy:HH:mm:ss Z"
    ]
    target => "@timestamp"  # 覆寫預設的 @timestamp
    timezone => "Asia/Taipei"
  }
}
```

### 時間格式對照表

| 格式 | 範例 | Logstash Pattern |
|------|------|------------------|
| ISO 8601 | `2023-04-21T10:15:23Z` | `ISO8601` |
| Unix timestamp | `1682070923` | `UNIX` |
| Java SimpleDateFormat | `2023-04-21 10:15:23` | `yyyy-MM-dd HH:mm:ss` |
| Nginx | `21/Apr/2023:10:15:23 +0000` | `dd/MMM/yyyy:HH:mm:ss Z` |

## KV：Key-Value 解析

用來解析 `key=value` 格式的日誌。

### 範例

原始日誌：
```
userId=12345 action=login ip=192.168.1.100 duration=1234
```

設定：
```ruby
filter {
  kv {
    field_split => " "
    value_split => "="
  }
}
```

輸出：
```json
{
  "userId": "12345",
  "action": "login",
  "ip": "192.168.1.100",
  "duration": "1234"
}
```

## Ruby：自訂邏輯

如果內建過濾器不夠用，可以用 Ruby 寫自訂邏輯。

### 範例：計算請求的執行時間百分位

```ruby
filter {
  ruby {
    code => '
      duration = event.get("duration")
      if duration
        if duration > 5000
          event.set("duration_level", "critical")
        elsif duration > 1000
          event.set("duration_level", "slow")
        else
          event.set("duration_level", "normal")
        end
      end
    '
  }
}
```

## 條件判斷：If / Else

Logstash 支援條件判斷。

### 語法

```ruby
filter {
  if [level] == "ERROR" {
    # 只處理錯誤日誌
    mutate {
      add_tag => ["error"]
    }
  } else if [level] == "WARN" {
    mutate {
      add_tag => ["warning"]
    }
  } else {
    # 其他情況
    drop { }  # 丟棄日誌
  }
}
```

### 運算子

```ruby
# 相等
if [level] == "ERROR" { }

# 不相等
if [level] != "DEBUG" { }

# 比較
if [status_code] >= 500 { }

# 正則表達式
if [user_agent] =~ /bot/ { }

# 欄位存在
if [error_message] { }

# 欄位不存在
if ![optional_field] { }

# AND / OR
if [level] == "ERROR" and [service] == "order-service" { }
if [status_code] == 500 or [status_code] == 503 { }

# IN
if [level] in ["ERROR", "FATAL"] { }
```

## 多行日誌處理

Java Exception 或 Stack Trace 通常跨多行，我們需要把它們合併成一筆日誌。

### Codec：Multiline

```ruby
input {
  file {
    path => "/var/log/app.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"  # 以時間戳記開頭的是新日誌
      negate => true
      what => "previous"  # 不符合的行合併到前一筆
    }
  }
}
```

### 範例

**輸入（3 筆實際日誌）**：
```
2023-04-21 10:15:23 ERROR Failed to process order
java.lang.NullPointerException: ...
	at com.example.OrderService.createOrder(OrderService.java:45)
2023-04-21 10:15:24 INFO Order created successfully
```

**輸出（2 筆合併後的日誌）**：
```json
[
  {
    "message": "2023-04-21 10:15:23 ERROR Failed to process order\njava.lang.NullPointerException: ...\n\tat com.example.OrderService.createOrder(OrderService.java:45)"
  },
  {
    "message": "2023-04-21 10:15:24 INFO Order created successfully"
  }
]
```

## 效能優化：Grok 的效能陷阱

Grok 雖然強大，但**效能很差**。

### 問題 1：貪婪匹配

```ruby
# 慢
match => { "message" => "%{GREEDYDATA:before} %{WORD:target} %{GREEDYDATA:after}" }

# 快
match => { "message" => "%{DATA:before} %{WORD:target} %{GREEDYDATA:after}" }
```

`%{DATA}` 是非貪婪匹配，比 `%{GREEDYDATA}` 快很多。

### 問題 2：多個 Grok 嘗試

```ruby
# 慢
match => [
  "%{PATTERN1}",
  "%{PATTERN2}",
  "%{PATTERN3}"
]
```

Logstash 會依序嘗試每個 Pattern，直到成功為止。如果你有 10 個 Pattern，最差情況會嘗試 10 次。

**解決方案：先用條件過濾**

```ruby
if [source] == "/var/log/nginx/access.log" {
  grok { match => { "message" => "%{NGINX_PATTERN}" } }
} else if [source] == "/var/log/app.log" {
  grok { match => { "message" => "%{APP_PATTERN}" } }
}
```

### 問題 3：複雜的正則表達式

```ruby
# 慢（超複雜的正則）
match => { "message" => "(?<complex>(\w+\s*)+\d{1,5}(\.\d+)?)" }

# 快（拆成多個簡單的 Grok）
match => { "message" => "%{WORD:simple1} %{INT:simple2}" }
```

## 測試你的 Grok Pattern

不要直接在生產環境測試！用 **Grok Debugger**：

1. Kibana → Dev Tools → Grok Debugger
2. 或線上工具：https://grokdebugger.com/

輸入你的日誌範例和 Grok Pattern，立即看到解析結果。

---

**Logstash Filter 是 ETL Pipeline 的心臟，掌握它就能把任何日誌變成有價值的數據。**
