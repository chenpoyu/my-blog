---
layout: post
title: "Keycloak Session 管理與單點登出"
date: 2024-03-18 11:25:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, Session, SSO]
---

這週研究了 Keycloak 的 Session 管理機制。做 SSO 有個重要問題：使用者在一個地方登入，所有系統都能用。那登出呢？要怎麼確保登出後，所有系統都真的登出？

還有一些實務問題，像是使用者同時在多台電腦登入、帳號被停用要馬上踢掉所有 Session、管理員要能強制登出特定使用者等等。

## Keycloak 的 Session 架構

Keycloak 的 Session 其實有兩層：

**User Session**：使用者在 Keycloak 的 Session，記錄在 Keycloak 的記憶體（或 Infinispan cache）。使用者登入後建立，登出或過期後刪除。

**Client Session**：使用者在各個應用程式的 Session，這是應用程式自己管理的。可能是 HTTP Session，也可能只是存一個 Access Token。

SSO 的單點登出要同時處理這兩層，才能確保真正登出。

## 查看 Session

在 Keycloak 管理介面，進入 Realm settings → Sessions，可以看到目前活躍的 Session 統計。

進入 Users，選一個使用者，切到 Sessions 頁籤，可以看到這個使用者的所有 Session：

```
Session ID: abc-123-def
Started: 2024-03-18 10:30:00
Last Access: 2024-03-18 11:20:00
Clients: spring-app, another-app
IP Address: 192.168.1.100
```

可以看到使用者從哪個 IP 登入、用了哪些 Client、Session 什麼時候建立的。

如果要強制登出，點旁邊的 `Sign out` 按鈕。這會刪除 Keycloak 的 User Session，並通知各個 Client。

## Session Timeout 設定

在 Realm settings → Tokens 可以設定 Session 的存活時間：

**SSO Session Idle**：如果使用者閒置這麼久，Session 就會過期。預設是 30 分鐘。

**SSO Session Max**：不管有沒有活動，Session 最多存活多久。預設是 10 小時。

**Client Session Idle**：Client 層級的閒置逾時。

**Client Session Max**：Client 層級的最大存活時間。

這些時間要根據實際需求調整。像是網銀這種敏感系統，可能要設短一點。內部系統如果使用者一直在用，可以設長一點避免頻繁重新登入。

## 單點登出（Single Logout）

單點登出有兩種模式：

**Front-Channel Logout**：瀏覽器導向模式。Keycloak 會對每個 Client 發送登出請求，讓瀏覽器去訪問各個 Client 的登出 endpoint。

流程是這樣：
1. 使用者在 App A 點登出
2. App A 導向 Keycloak 的 logout endpoint
3. Keycloak 刪除 User Session
4. Keycloak 產生一個隱藏的 iframe，裡面對每個 Client 發送登出請求
5. 各個 Client 收到請求，清除自己的 Session
6. 完成後導回指定頁面

**Back-Channel Logout**：後端對後端模式。Keycloak 直接用 HTTP POST 通知各個 Client 登出。

流程是這樣：
1. 使用者在 App A 點登出
2. App A 導向 Keycloak 的 logout endpoint
3. Keycloak 刪除 User Session
4. Keycloak 對每個 Client 的 logout endpoint 發送 HTTP POST
5. 各個 Client 收到通知，清除 Session
6. 導回指定頁面

Back-Channel 比較可靠，因為不依賴瀏覽器。但要確保 Keycloak 能連到各個 Client 的網路。

## 設定 Spring Boot 的登出

修改 Spring Security 設定，處理 Keycloak 的登出通知：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .oauth2Login(oauth2 -> oauth2
                .defaultSuccessUrl("/dashboard", true)
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessHandler(oidcLogoutSuccessHandler())
            );
        
        return http.build();
    }
    
    private LogoutSuccessHandler oidcLogoutSuccessHandler() {
        OidcClientInitiatedLogoutSuccessHandler handler = 
            new OidcClientInitiatedLogoutSuccessHandler(clientRegistrationRepository);
        handler.setPostLogoutRedirectUri("{baseUrl}");
        return handler;
    }
}
```

這個設定會在使用者登出時，自動導向 Keycloak 的 logout endpoint，完成單點登出。

## Back-Channel Logout 的設定

如果要用 Back-Channel Logout，需要在 Client 設定 logout URL：

1. 進入 Clients，選擇 `spring-app`
2. 切到 Settings 頁籤
3. 打開 `Backchannel logout`
4. 填入 `Backchannel logout URL`：`http://localhost:8081/logout/backchannel`
5. 儲存

然後在 Spring Boot 實作一個 endpoint 接收登出通知：

```java
@RestController
public class LogoutController {
    
    @Autowired
    private SessionRegistry sessionRegistry;
    
    @PostMapping("/logout/backchannel")
    public ResponseEntity<?> backchannelLogout(@RequestBody String logoutToken) {
        // 解析 logout token，拿到 session ID 或 user ID
        // 找出對應的 HTTP Session，強制登出
        
        try {
            Jwt jwt = jwtDecoder.decode(logoutToken);
            String sid = jwt.getClaimAsString("sid");
            
            // 找出這個 session 並 invalidate
            List<SessionInformation> sessions = sessionRegistry.getAllSessions(
                principal, false);
            for (SessionInformation session : sessions) {
                if (session.getSessionId().equals(sid)) {
                    session.expireNow();
                }
            }
            
            return ResponseEntity.ok().build();
        } catch (Exception e) {
            return ResponseEntity.badRequest().build();
        }
    }
}
```

這樣當使用者在任一個系統登出，Keycloak 會通知所有 Client，全部一起登出。

## 強制登出特定使用者

有時候需要強制登出使用者，比如帳號被停用、偵測到異常行為、密碼被重設等。

可以用 Admin REST API：

```java
public void forceLogout(String username) {
    RealmResource realmResource = keycloak.realm("company");
    UsersResource usersResource = realmResource.users();
    
    List<UserRepresentation> users = usersResource.search(username);
    if (!users.isEmpty()) {
        String userId = users.get(0).getId();
        
        // 登出這個使用者的所有 Session
        usersResource.get(userId).logout();
    }
}
```

或者直接在管理介面操作：

1. 進入 Users，選擇該使用者
2. 切到 Sessions 頁籤
3. 點 `Sign out all sessions`

使用者的所有 Session 會被刪除，下次存取任何系統時都要重新登入。

## 限制同時登入裝置數

有些場景需要限制使用者只能同時在幾台裝置登入，比如影音串流平台限制帳號共享。

Keycloak 沒有內建這個功能，但可以自己實作：

寫一個 Event Listener，在 LOGIN 事件時檢查這個使用者的 Session 數量：

```java
@Override
public void onEvent(Event event) {
    if (event.getType() == EventType.LOGIN) {
        String userId = event.getUserId();
        
        // 查詢這個使用者的 Session 數量
        UserSessionProvider sessions = session.sessions();
        List<UserSessionModel> userSessions = 
            sessions.getUserSessions(session.getContext().getRealm(), userId);
        
        // 如果超過限制，刪除最舊的 Session
        if (userSessions.size() > MAX_SESSIONS) {
            UserSessionModel oldestSession = userSessions.get(0);
            sessions.removeUserSession(
                session.getContext().getRealm(), oldestSession);
        }
    }
}
```

這樣當使用者嘗試第 N+1 次登入時，會自動踢掉最舊的 Session。

## Session 同步問題

如果 Keycloak 是 cluster 部署，Session 要在多個節點間同步。Keycloak 用 Infinispan 來做 Session replication。

預設的設定對小型部署夠用，但如果 Session 量很大，要調整 Infinispan 的設定：

```xml
<distributed-cache name="sessions" owners="2">
    <expiration lifespan="-1"/>
</distributed-cache>
```

`owners="2"` 表示每個 Session 會複製到 2 個節點。增加 owners 可以提高可用性，但會增加網路流量。

另一種做法是用 external Infinispan cluster，或者改用 Redis 做 Session store（需要額外的 extension）。

## 實務經驗

單點登出說起來簡單，實作起來有很多眉角：

**網路問題**：如果用 Back-Channel Logout，Keycloak 要能連到所有 Client。在微服務架構裡，可能有些服務在內網，要處理好網路路由。

**逾時處理**：登出通知可能會失敗或 timeout。要設定合理的 timeout，避免登出流程卡住。

**非同步處理**：如果 Client 很多，逐一通知會很慢。可以考慮用 Message Queue 非同步處理。

**前端 SPA**：如果前端是 React/Vue 這種 SPA，登出後要記得清除前端存的 Token 和狀態，不然可能會有殘留。

下週想研究 Keycloak 的 Client Adapter，看看除了 Spring Boot，其他語言和框架（像是 Node.js、Python、.NET）要怎麼整合。
