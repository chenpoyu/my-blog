---
layout: post
title: "用 Keycloak Admin REST API 管理使用者"
date: 2024-02-05 11:30:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, REST API, 自動化]
---

這週研究了 Keycloak 的 Admin REST API。想到一個實際場景：如果企業的 HR 系統有員工的入離職資料，應該要能自動在 Keycloak 建立或停用帳號，不要每次都要人工去管理介面操作。

Keycloak 提供完整的 Admin REST API，可以用程式化的方式管理 realm、使用者、角色等等。剛好可以拿來做自動化。

## 取得 Admin Access Token

要呼叫 Admin API，首先要拿到有管理權限的 Access Token。Keycloak 預設的 `admin` 帳號是在 `master` realm，但我們不該用 master realm 的帳號來管理其他 realm。

正確做法是在 `company` realm 建立一個 service account：

1. 在 Keycloak 建立一個新的 client：
   - Client ID: `admin-cli`
   - Client authentication: 打開
   - Service accounts roles: 打開（這樣可以用 client credentials 流程）
2. 儲存後，在 Service account roles 頁籤，給它管理權限
3. 從 Filter by realm roles 選擇 `admin`，或者更細緻的 `manage-users`、`manage-clients` 等等

這樣建立出來的 client 可以用 Client Credentials 流程直接拿 Token，不需要使用者登入。

取得 Token 的方式：

```bash
curl -X POST http://localhost:8080/realms/company/protocol/openid-connect/token \
  -d "client_id=admin-cli" \
  -d "client_secret=你的-client-secret" \
  -d "grant_type=client_credentials"
```

回應會包含 `access_token`，用這個 Token 就能呼叫 Admin API。

## 用 Java 呼叫 Admin API

Keycloak 提供官方的 Java Admin Client，加入依賴：

```xml
<dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-admin-client</artifactId>
    <version>23.0.4</version>
</dependency>
```

寫一個 Service 來管理使用者：

```java
@Service
public class KeycloakUserService {
    
    private final Keycloak keycloak;
    private final String realm = "company";
    
    public KeycloakUserService() {
        this.keycloak = KeycloakBuilder.builder()
            .serverUrl("http://localhost:8080")
            .realm("company")
            .clientId("admin-cli")
            .clientSecret("你的-client-secret")
            .grantType(OAuth2Constants.CLIENT_CREDENTIALS)
            .build();
    }
    
    public String createUser(String username, String email, 
                            String firstName, String lastName, String password) {
        
        RealmResource realmResource = keycloak.realm(realm);
        UsersResource usersResource = realmResource.users();
        
        UserRepresentation user = new UserRepresentation();
        user.setUsername(username);
        user.setEmail(email);
        user.setFirstName(firstName);
        user.setLastName(lastName);
        user.setEnabled(true);
        
        Response response = usersResource.create(user);
        
        if (response.getStatus() == 201) {
            String userId = response.getLocation().getPath()
                .replaceAll(".*/([^/]+)$", "$1");
            
            // 設定密碼
            CredentialRepresentation credential = new CredentialRepresentation();
            credential.setType(CredentialRepresentation.PASSWORD);
            credential.setValue(password);
            credential.setTemporary(false);
            
            usersResource.get(userId).resetPassword(credential);
            
            return userId;
        } else {
            throw new RuntimeException("建立使用者失敗: " + response.getStatusInfo());
        }
    }
    
    public void disableUser(String username) {
        RealmResource realmResource = keycloak.realm(realm);
        UsersResource usersResource = realmResource.users();
        
        List<UserRepresentation> users = usersResource.search(username);
        if (!users.isEmpty()) {
            UserRepresentation user = users.get(0);
            user.setEnabled(false);
            usersResource.get(user.getId()).update(user);
        }
    }
    
    public void assignRole(String username, String roleName) {
        RealmResource realmResource = keycloak.realm(realm);
        UsersResource usersResource = realmResource.users();
        
        List<UserRepresentation> users = usersResource.search(username);
        if (!users.isEmpty()) {
            String userId = users.get(0).getId();
            
            ClientRepresentation client = realmResource.clients()
                .findByClientId("spring-app").get(0);
            
            RoleRepresentation role = realmResource.clients()
                .get(client.getId())
                .roles()
                .get(roleName)
                .toRepresentation();
            
            usersResource.get(userId)
                .roles()
                .clientLevel(client.getId())
                .add(Arrays.asList(role));
        }
    }
}
```

這樣就可以在程式裡管理 Keycloak 的使用者了。

## 測試 API

寫個簡單的 REST Controller 測試：

```java
@RestController
@RequestMapping("/admin/users")
public class UserAdminController {
    
    @Autowired
    private KeycloakUserService keycloakUserService;
    
    @PostMapping
    public ResponseEntity<?> createUser(@RequestBody CreateUserRequest request) {
        try {
            String userId = keycloakUserService.createUser(
                request.getUsername(),
                request.getEmail(),
                request.getFirstName(),
                request.getLastName(),
                request.getPassword()
            );
            
            // 指派角色
            if (request.getRoles() != null) {
                for (String role : request.getRoles()) {
                    keycloakUserService.assignRole(request.getUsername(), role);
                }
            }
            
            return ResponseEntity.ok(Map.of("userId", userId));
        } catch (Exception e) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", e.getMessage()));
        }
    }
    
    @DeleteMapping("/{username}")
    public ResponseEntity<?> disableUser(@PathVariable String username) {
        keycloakUserService.disableUser(username);
        return ResponseEntity.ok().build();
    }
}
```

用 Postman 測試一下：

```bash
POST http://localhost:8081/admin/users
Content-Type: application/json

{
  "username": "new.employee",
  "email": "new.employee@mycompany.com",
  "firstName": "New",
  "lastName": "Employee",
  "password": "Welcome123",
  "roles": ["sales-user"]
}
```

成功！去 Keycloak 管理介面看，新使用者已經建立了，而且有 `sales-user` 這個角色。

## 整合 HR 系統

有了這個 API，就可以讓 HR 系統在員工入職時自動呼叫，建立 Keycloak 帳號。離職時就停用帳號。

不過這邊要注意權限控制，不能讓任何人都能呼叫這個 API。要加上 Spring Security 的保護：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt());
        
        return http.build();
    }
}
```

這樣只有有 `ADMIN` 角色的使用者才能呼叫 `/admin/**` 的 API。

或者更嚴格一點，用 API Key 或 IP 白名單來限制只有 HR 系統能呼叫。

## Admin API 的其他功能

除了使用者管理，Admin API 還能做很多事：

- **Client 管理**：動態建立、修改 client 設定
- **Realm 管理**：建立新的 realm、匯出設定
- **角色管理**：動態建立角色、設定角色權限
- **Session 管理**：查詢目前登入的 session、強制登出

這些功能對我們做系統整合很有幫助。比如說，如果客戶要新增一個系統，我們可以用程式自動在 Keycloak 建立對應的 client，不用手動去介面操作。

## 效能考量

用 Admin Client 要注意效能問題。每次建立 `Keycloak` 物件都會去取得新的 Access Token，如果頻繁呼叫會有 overhead。

比較好的做法是把 `Keycloak` 物件做成 singleton，或者用 connection pool。官方建議一個應用程式只建立一個 `Keycloak` instance。

另外 Admin API 的 Token 也有有效期限，預設是 1 分鐘。如果長時間執行的批次作業，要處理 Token 過期的問題。Keycloak Admin Client 會自動 refresh token，所以一般不用擔心。

## 下一步

Admin API 的基本功能會用了，但還有一些進階主題想研究：

如何監控 Keycloak 的狀態？有沒有 API 可以查詢系統資源使用情況、登入失敗次數之類的。

批次處理大量使用者時，有沒有比較有效率的方式？一筆一筆建立使用者會很慢。

還有 Keycloak 的事件監聽，能不能在使用者登入、登出時發送通知到系統。這些之後慢慢研究。

目前 Admin API 已經能滿足自動化的需求了，基本功能都通了。
