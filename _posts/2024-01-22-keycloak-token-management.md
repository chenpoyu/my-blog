---
layout: post
title: "搞懂 Keycloak 的 Token 機制"
date: 2024-01-22 10:35:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, JWT, Token]
---

這週深入研究了 Keycloak 的 Token 機制。之前只知道登入後會拿到 Token，但具體是怎麼運作的、Token 裡面有什麼內容、要怎麼驗證，這些都要弄清楚，特別是如果系統架構是微服務，Token 的傳遞和驗證很關鍵。

## OAuth 2.0 的授權流程

先回顧一下整個登入流程（Authorization Code Flow）：

1. 使用者訪問我們的應用 `http://localhost:8081/dashboard`
2. Spring Security 發現使用者沒登入，重導向到 Keycloak 登入頁面
3. 使用者在 Keycloak 輸入帳號密碼
4. Keycloak 驗證通過，重導向回我們的應用，並帶上一個 `code`
5. Spring Security 用這個 `code` 去 Keycloak 換取 Token
6. 拿到 Token 後，使用者就算登入成功了

關鍵是第 5 步拿到的 Token，其實有三個：

- **Access Token**：存取資源用的，有效期短（預設 5 分鐘）
- **Refresh Token**：用來換新的 Access Token，有效期長（預設 30 分鐘）
- **ID Token**：OpenID Connect 專用，包含使用者身份資訊

## 解析 JWT Token

Keycloak 產生的 Token 是 JWT（JSON Web Token）格式。JWT 由三部分組成，用 `.` 分隔：

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

三段分別是：
1. Header（演算法和 Token 類型）
2. Payload（實際資料）
3. Signature（簽章）

想看 Token 內容，可以用 [jwt.io](https://jwt.io) 這個線上工具，或者自己寫程式解析。

我寫了一個測試 endpoint 來看 Token 內容：

```java
@RestController
public class TokenController {
    
    @GetMapping("/token/info")
    public Map<String, Object> tokenInfo(
            @RegisteredOAuth2AuthorizedClient("keycloak") 
            OAuth2AuthorizedClient authorizedClient) {
        
        OAuth2AccessToken accessToken = authorizedClient.getAccessToken();
        
        Map<String, Object> info = new HashMap<>();
        info.put("tokenValue", accessToken.getTokenValue());
        info.put("issuedAt", accessToken.getIssuedAt());
        info.put("expiresAt", accessToken.getExpiresAt());
        info.put("scopes", accessToken.getScopes());
        
        return info;
    }
}
```

登入後訪問 `/token/info`，可以看到 Access Token 的基本資訊。把 `tokenValue` 複製到 jwt.io 解析，可以看到 payload 內容：

```json
{
  "exp": 1705903520,
  "iat": 1705903220,
  "jti": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "iss": "http://localhost:8080/realms/company",
  "aud": "account",
  "sub": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "typ": "Bearer",
  "azp": "spring-app",
  "session_state": "3d42f9b0-a8c7-4e56-b23f-8d9a6f7e1234",
  "preferred_username": "testuser",
  "email": "testuser@example.com",
  "resource_access": {
    "spring-app": {
      "roles": ["finance-manager"]
    }
  }
}
```

重點欄位：
- `exp`：過期時間（Unix timestamp）
- `iss`：發行者（Keycloak 的位址）
- `sub`：使用者唯一識別碼
- `preferred_username`：使用者名稱
- `resource_access`：使用者的角色權限

## Token 驗證機制

Token 是怎麼驗證的？JWT 的好處是可以做「無狀態驗證」，不用每次都去 Keycloak 問。

JWT 的 Signature 是用 Keycloak 的私鑰簽署的，我們可以用公鑰來驗證。Keycloak 有個公開的 endpoint 提供公鑰資訊：

```
http://localhost:8080/realms/company/protocol/openid-connect/certs
```

Spring Security 會自動去抓這個公鑰，然後驗證 Token 的簽章。只要簽章正確，而且 Token 沒過期，就代表這是合法的 Token。

## 手動驗證 Token

如果我們的後端 API 要手動驗證 Token（比如其他微服務），可以用 Spring Security OAuth2 Resource Server：

在 `pom.xml` 加入依賴：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

在 `application.yml` 設定：

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/company
```

然後修改 SecurityConfig：

```java
@Bean
public SecurityFilterChain resourceServerFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/**").authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt
                .jwtAuthenticationConverter(jwtAuthenticationConverter())
            )
        );
    
    return http.build();
}

private JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(new KeycloakRoleConverter());
    return converter;
}
```

這樣設定後，API 請求要在 Header 帶上 Token：

```
Authorization: Bearer eyJhbGci...
```

Spring Security 會自動驗證 Token 的簽章和有效期限。

## Refresh Token 的使用

Access Token 只有 5 分鐘，過期後怎麼辦？這時就要用 Refresh Token 去換新的 Access Token。

Spring Security OAuth2 Client 會自動處理這件事。當 Access Token 過期時，它會用 Refresh Token 去 Keycloak 換新的，對使用者來說是無感的。

但如果是前端是 SPA（Single Page Application），就要自己處理。前端需要：
1. 偵測 API 回傳 401 Unauthorized
2. 用 Refresh Token 去 Keycloak 換新的 Access Token
3. 重新發送原本的 API 請求

這部分比較複雜，之後如果要做前後端分離的架構，再來研究。

## Token 的安全性考量

研究下來，發現幾個要注意的地方：

Token 不能存在 localStorage，因為容易被 XSS 攻擊。如果是 SPA 架構，建議存在 httpOnly cookie，或者用 BFF（Backend For Frontend）模式。

Token 雖然有簽章，但內容是 Base64 編碼而已，任何人都能解碼看到 payload 的內容。所以不要把敏感資訊（像是密碼、信用卡號）放在 Token 裡。

Access Token 的有效期要設短一點，降低被盜用的風險。但也不能太短，不然會一直換 Token，影響效能。5 分鐘到 30 分鐘之間算是合理的範圍。

## 下一步

Token 機制大概搞懂了，但實際應用還有很多細節。像是：

前端要怎麼拿到 Token？如果是傳統的 Server-Side Rendering 還好，但如果是 React 或 Vue 這種 SPA，流程會不一樣。

微服務之間要怎麼傳遞 Token？用 Feign Client 的話，要怎麼自動把 Token 加到 Header？

如果使用者同時開多個瀏覽器分頁，Token 的同步要怎麼處理？

這些問題之後慢慢研究。目前先把基本的 Token 驗證機制搞定，讓後端 API 可以正常運作。
