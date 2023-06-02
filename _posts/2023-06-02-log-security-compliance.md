---
layout: post
title: "日誌安全與合規：保護敏感資訊"
date: 2023-06-02 09:00:00 +0800
categories: [Observability, Logging, Security]
tags: [Log Security, GDPR, Compliance, Data Protection, Sensitive Data]
---

你的日誌裡有多少敏感資訊？

去年某家公司因為**日誌中包含信用卡號**，被罰款。

今天我們來談談日誌安全與合規。

## 日誌中的敏感資訊

常見的敏感資訊：

### 1. 個人身份資訊（PII）

- 姓名、電話、Email
- 身分證字號、護照號碼
- 地址、生日

### 2. 財務資訊

- 信用卡號、CVV
- 銀行帳號
- 交易金額

### 3. 認證資訊

- 密碼（明文或 Hash）
- API Token、Access Token
- Session ID、Cookie

### 4. 健康資訊（HIPAA）

- 病歷、診斷結果
- 處方記錄

### 5. 業務機密

- 商業邏輯、演算法
- 內部 API 路徑
- 資料庫 Schema

## 法規要求

### GDPR（歐盟一般資料保護規範）

**要求**：
- 使用者有「被遺忘權」（Right to be Forgotten）
- 資料外洩需在 72 小時內通報
- 未經同意不得處理個人資料

**對日誌的影響**：
- 不能在日誌中記錄使用者的個人資訊
- 如果記錄了，使用者要求刪除時，你必須能找到並刪除

### PCI DSS（支付卡產業資料安全標準）

**要求**：
- 不得儲存完整的信用卡號（可以儲存後 4 碼）
- 不得儲存 CVV
- 日誌必須加密

### HIPAA（美國健康保險可攜性及責任法案）

**要求**：
- 健康資訊必須加密
- 只有授權人員可以查看
- 保留審計日誌（誰在何時查看了哪些資料）

## 資料遮罩策略

### 策略 1：完全移除

最簡單的方式：**不要記錄敏感資訊。**

```java
// 壞
log.info("User login: username={}, password={}", username, password);

// 好
log.info("User login: username={}", username);
```

### 策略 2：部分遮罩

保留部分資訊，方便 Debug。

```java
public class MaskUtils {
    public static String maskEmail(String email) {
        if (email == null) return null;
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) return email;
        return email.charAt(0) + "***" + email.substring(atIndex);
    }
    
    public static String maskCardNumber(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) return cardNumber;
        return "****-****-****-" + cardNumber.substring(cardNumber.length() - 4);
    }
    
    public static String maskPhone(String phone) {
        if (phone == null || phone.length() < 4) return phone;
        return "***-***-" + phone.substring(phone.length() - 4);
    }
}
```

使用：

```java
log.info("Payment successful: user={}, card={}", 
    MaskUtils.maskEmail(user.getEmail()),
    MaskUtils.maskCardNumber(card.getNumber())
);

// 輸出：Payment successful: user=j***@example.com, card=****-****-****-1234
```

### 策略 3：Hash

對於需要追蹤但不需要看到原始值的資料，用 Hash。

```java
public class HashUtils {
    public static String hashUserId(String userId) {
        return DigestUtils.sha256Hex(userId + "your-secret-salt");
    }
}
```

```java
log.info("User action: userId={}, action={}", 
    HashUtils.hashUserId(user.getId()),
    action
);

// 輸出：User action: userId=a1b2c3d4..., action=create_order
```

**好處**：
- 同一個使用者的 Hash 永遠一樣，可以追蹤行為
- 無法從 Hash 反推出原始 userId

### 策略 4：Tokenization

用一個 Token 替代原始值，Token 本身沒有意義。

```java
public class TokenService {
    private final Map<String, String> tokenMap = new ConcurrentHashMap<>();
    
    public String tokenize(String value) {
        String token = UUID.randomUUID().toString();
        tokenMap.put(token, value);
        return token;
    }
    
    public String detokenize(String token) {
        return tokenMap.get(token);
    }
}
```

```java
String token = tokenService.tokenize(user.getEmail());
log.info("User registered: userToken={}", token);

// 需要時可以反查
String email = tokenService.detokenize(token);
```

## Logstash 中的資料遮罩

### 使用 Mutate Filter

```ruby
filter {
  # 移除欄位
  mutate {
    remove_field => ["password", "cvv", "secret"]
  }
  
  # 遮罩信用卡號
  if [cardNumber] {
    ruby {
      code => '
        card = event.get("cardNumber")
        if card && card.length >= 4
          event.set("cardNumber", "****-****-****-" + card[-4..-1])
        end
      '
    }
  }
  
  # 遮罩 Email
  if [email] {
    ruby {
      code => '
        email = event.get("email")
        if email && email.include?("@")
          parts = email.split("@")
          event.set("email", parts[0][0] + "***@" + parts[1])
        end
      '
    }
  }
}
```

### 使用 Grok 解析時直接遮罩

```ruby
filter {
  grok {
    match => {
      "message" => "cardNumber=%{DATA:cardNumber_full}"
    }
  }
  
  ruby {
    code => '
      card = event.get("cardNumber_full")
      if card && card.length >= 4
        event.set("cardNumber", "****-****-****-" + card[-4..-1])
      end
    '
  }
  
  mutate {
    remove_field => ["cardNumber_full"]  # 刪除完整卡號
  }
}
```

## Elasticsearch 的安全設定

### 1. 啟用 X-Pack Security

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

### 2. 建立不同的角色

```bash
# 唯讀角色（給一般開發者）
PUT _security/role/logs_readonly
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read"]
    }
  ]
}

# 完全權限角色（給 SRE）
PUT _security/role/logs_admin
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["all"]
    }
  ]
}
```

### 3. 建立使用者

```bash
PUT _security/user/developer1
{
  "password": "password123",
  "roles": ["logs_readonly"]
}

PUT _security/user/sre1
{
  "password": "password456",
  "roles": ["logs_admin"]
}
```

### 4. Field-Level Security（欄位級安全）

某些欄位只有特定角色可以看。

```bash
PUT _security/role/logs_restricted
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read"],
      "field_security": {
        "grant": ["*"],
        "except": ["cardNumber", "password", "ssn"]  # 隱藏這些欄位
      }
    }
  ]
}
```

### 5. Document-Level Security（文件級安全）

某些文件只有特定角色可以看。

```bash
PUT _security/role/logs_us_only
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read"],
      "query": {
        "term": {
          "country": "US"  # 只能看 US 的日誌
        }
      }
    }
  ]
}
```

## 日誌加密

### 傳輸加密（In-Transit）

Logstash 到 Elasticsearch 使用 TLS：

```ruby
output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    ssl => true
    cacert => "/path/to/ca.crt"
    user => "logstash"
    password => "password"
  }
}
```

### 儲存加密（At-Rest）

Elasticsearch 支援磁碟加密：

```yaml
# elasticsearch.yml
xpack.security.encryptedSavedObjects.encryptionKey: "your-32-char-encryption-key-here"
```

或使用作業系統層級的加密（如 LUKS、BitLocker）。

## 審計日誌

記錄「誰在何時查看了哪些日誌」。

### Elasticsearch Audit Log

```yaml
# elasticsearch.yml
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: ["access_granted", "access_denied"]
```

審計日誌會記錄：

```json
{
  "@timestamp": "2023-06-02T10:15:23.123Z",
  "event.type": "access_granted",
  "user.name": "developer1",
  "url.path": "/_search",
  "indices": ["logs-2023.06.02"]
}
```

## 資料保留策略

### GDPR 要求

使用者有權要求刪除他的資料。

**實作方式 1：定期清理**

用 ILM 自動刪除舊日誌：

```json
{
  "delete": {
    "min_age": "30d",
    "actions": {
      "delete": {}
    }
  }
}
```

**實作方式 2：使用者 ID 索引**

為每個使用者建立一個索引：

```
logs-user-12345-*
```

當使用者要求刪除時，直接刪除該索引。

**實作方式 3：Delete By Query**

```bash
POST logs-*/_delete_by_query
{
  "query": {
    "term": {
      "userId": "12345"
    }
  }
}
```

## 實戰案例：日誌外洩事件

### 事件

某家公司的 Kibana 沒有設定密碼，被 Google 索引到。任何人都可以查看他們的日誌。

### 外洩的資訊

- 使用者的 Email、手機號碼
- API Token（明文）
- 資料庫連線字串（包含密碼）
- 內部系統架構

### 後果

- GDPR 罰款：50 萬歐元
- 使用者信任流失
- 強制進行全系統的密碼重置

### 如何避免

1. **Kibana 必須設定密碼**
2. **不要在日誌中記錄密碼或 Token**
3. **定期掃描公開的 Elasticsearch / Kibana**（如用 Shodan）
4. **使用 VPN 或 IP 白名單限制訪問**

## 檢查清單

### 開發階段

- [ ] 所有密碼都遮罩了嗎？
- [ ] 信用卡號只保留後 4 碼嗎？
- [ ] Email 和手機號碼有遮罩嗎？
- [ ] Session ID 和 Token 沒有被記錄嗎？

### 部署階段

- [ ] Elasticsearch 啟用了認證嗎？
- [ ] Kibana 設定了密碼嗎？
- [ ] TLS 加密啟用了嗎？
- [ ] 審計日誌啟用了嗎？

### 運維階段

- [ ] 有定期審查日誌內容嗎？
- [ ] 有定期刪除舊日誌嗎？
- [ ] 有監控異常的查詢嗎？
- [ ] 有備份日誌嗎（並加密）？

---

**日誌安全不是選項，而是必須。**

一次外洩事件可能讓你的公司關門，不要等到出事才後悔。
