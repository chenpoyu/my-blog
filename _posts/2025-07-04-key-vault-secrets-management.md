---
layout: post
title: "Key Vault：不要把密碼寫在 code 裡"
date: 2025-07-04 09:45:00 +0800
categories: [安全性]
tags: [Azure Key Vault, Managed Identity, 安全性]
---

## 資安顧問的警告

上週客戶的資安顧問來做 security review，看了我們的 code 之後皺眉：

「你們的 connection string 和 API key 都放在哪裡？」

「Azure App Configuration。」我說。

「可以給我看一下嗎？」

我打開 Azure Portal，展示給他看：

```
SQL_CONNECTION_STRING = "Server=tcp:xxx.database.windows.net;Database=xxx;..."
REDIS_CONNECTION_STRING = "xxx.redis.cache.windows.net,password=xxx..."
PAYMENT_API_KEY = "sk_live_xxxxxxxxxxxxxxxx"
```

他搖頭：「這些是機密資訊，不應該放在 App Configuration。」

「為什麼？」

「因為任何有 Azure Portal 權限的人，都可以看到這些資訊。而且 App Configuration 的 access log 不夠詳細。」

「應該用 **Azure Key Vault**。」

## Azure Key Vault 是什麼

Azure Key Vault 是專門用來存放機密資訊的服務：

- **Secrets**: 密碼、API key、connection string
- **Keys**: 加密用的 key（對稱或非對稱）
- **Certificates**: SSL 憑證

它的特點：
1. **嚴格的存取控制**：用 Azure AD + RBAC 控制誰可以讀取
2. **詳細的 audit log**：誰在什麼時候讀取了哪個 secret
3. **自動輪替**：可以定期自動更換 secret
4. **HSM 保護**：Premium tier 可以用硬體加密模組

「聽起來不錯，但改起來會不會很麻煩？」我問。

「不會，用 Managed Identity 就很簡單。」

## Managed Identity

在 Azure，如果你要讓 App Service 存取 Key Vault，有兩種方式：

### 方式 1：用 Service Principal（不推薦）

```csharp
var credential = new ClientSecretCredential(
    tenantId: "xxx",
    clientId: "xxx",
    clientSecret: "xxx"  // 又是一個 secret！
);
```

問題：你需要另一個 secret 來存取 Key Vault⋯⋯這不就又回到原點了嗎？

### 方式 2：用 Managed Identity（推薦）

Managed Identity 是 Azure 提供的身份認證機制：
- **System-assigned**: Azure 自動幫 App Service 建立一個 identity
- **User-assigned**: 自己建立一個 identity，多個服務共用

不需要任何 credential，Azure 會自動處理。

```csharp
var credential = new DefaultAzureCredential();  // 就這樣！
```

簡單到不行。

## 實作：搬到 Key Vault

### Step 1: 建立 Key Vault

在 Azure Portal 建立 Key Vault：

```
Name: my-app-keyvault
Pricing tier: Standard
Soft delete: Enabled (預設)
Purge protection: Enabled
```

### Step 2: 啟用 Managed Identity

在 App Service 的設定中啟用 System-assigned managed identity：

```
Settings → Identity → System assigned → Status: On
```

Azure 會自動建立一個 identity，並給你一個 Object ID。

### Step 3: 給 Managed Identity 權限

在 Key Vault 的 Access policies 中，新增一個 policy：

```
Principal: [App Service 的 Managed Identity]
Secret permissions: Get, List
```

這樣 App Service 就可以讀取 Key Vault 的 secrets 了。

### Step 4: 把 secrets 搬到 Key Vault

在 Key Vault 中建立 secrets：

```
Name: SQL-CONNECTION-STRING
Value: Server=tcp:xxx.database.windows.net;...

Name: REDIS-CONNECTION-STRING
Value: xxx.redis.cache.windows.net,password=...

Name: PAYMENT-API-KEY
Value: sk_live_xxx...
```

注意：**Key Vault 的 secret name 不能有底線 `_`**，要用破折號 `-`。

### Step 5: 改 code

安裝套件：

```bash
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
```

建立一個 helper class：

```csharp
public class KeyVaultService
{
    private readonly SecretClient _secretClient;
    
    public KeyVaultService(IConfiguration configuration)
    {
        var keyVaultUrl = configuration["KeyVaultUrl"];  // https://my-app-keyvault.vault.azure.net/
        _secretClient = new SecretClient(
            new Uri(keyVaultUrl), 
            new DefaultAzureCredential()
        );
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        var secret = await _secretClient.GetSecretAsync(secretName);
        return secret.Value.Value;
    }
}
```

在 `Program.cs` 中註冊：

```csharp
builder.Services.AddSingleton<KeyVaultService>();

// 從 Key Vault 讀取 connection string
var keyVaultService = builder.Services.BuildServiceProvider()
    .GetRequiredService<KeyVaultService>();

var sqlConnectionString = await keyVaultService.GetSecretAsync("SQL-CONNECTION-STRING");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(sqlConnectionString));

var redisConnectionString = await keyVaultService.GetSecretAsync("REDIS-CONNECTION-STRING");
builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = redisConnectionString);
```

完成！

## 遇到的問題

### 問題 1: DefaultAzureCredential 找不到 credential

本地開發時，`DefaultAzureCredential` 會噴錯：

```
Azure.Identity.AuthenticationFailedException: 
DefaultAzureCredential failed to retrieve a token
```

因為本地環境沒有 Managed Identity。

解決方式：用 **Azure CLI** 登入：

```bash
az login
```

`DefaultAzureCredential` 會自動使用 Azure CLI 的 credential。

### 問題 2: Key Vault 的權限模型

Key Vault 有兩種權限模型：

1. **Access policies**（舊）
2. **RBAC**（新）

如果用 RBAC，要給 Managed Identity 這個 role：

```
Key Vault Secrets User
```

不要給 `Key Vault Contributor`，那個權限太大了。

### 問題 3: 讀取 Key Vault 會變慢

每次讀取 Key Vault 都要透過網路，會增加啟動時間。

解決方式：**Cache secrets**。

```csharp
public class KeyVaultService
{
    private readonly SecretClient _secretClient;
    private readonly Dictionary<string, string> _cache = new();
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        if (_cache.ContainsKey(secretName))
        {
            return _cache[secretName];
        }
        
        var secret = await _secretClient.GetSecretAsync(secretName);
        var value = secret.Value.Value;
        
        _cache[secretName] = value;  // 存到 cache
        return value;
    }
}
```

但要小心：**如果 secret 更新了，cache 不會自動更新**。

可以設定 cache 的 TTL，例如 1 小時後自動清除。

## 額外的好處

用 Key Vault 還有一些額外好處：

### 1. Audit Log

Key Vault 會記錄所有存取：

- 誰（哪個 identity）
- 什麼時候
- 讀取了哪個 secret
- 結果（成功或失敗）

這些 log 可以整合到 Azure Monitor 或 Log Analytics。

### 2. Secret Rotation

Key Vault 可以自動輪替 secret：

- 設定多久輪替一次（例如 90 天）
- 輪替時觸發 webhook 通知
- 可以整合 Azure Functions 自動更新資料庫密碼

### 3. Soft Delete

如果不小心刪掉 secret，可以在 90 天內還原。

這在 production 環境很重要。

## 成本

Key Vault 的計價：

| 項目 | 價格 |
|-----|-----|
| Standard tier | $0.03 per 10,000 operations |
| Premium tier (with HSM) | $1/key/month + operations |

我們用 Standard tier，一個月大概 100 萬次 operations（因為有 cache）。

成本：100 萬 / 10,000 × $0.03 = **$3/月**。

超便宜。

## 下一步

Key Vault 搞定了，但還有其他安全性問題要處理：

1. **API 認證**：JWT 的實作
2. **Rate limiting**：防止 API 被濫用
3. **CORS 設定**：前端只能從特定 domain 呼叫
4. **DR plan**：災難恢復計畫

下週繼續。
