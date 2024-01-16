---
layout: post
title: "Keycloak 的角色與權限處理"
date: 2024-01-17 15:40:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, RBAC, Spring Security]
---

這週花了不少時間在研究 Keycloak 的角色權限機制。一般企業系統的需求是：不同部門的人登入系統後，看到的功能和資料要不一樣。財務部只能看財務相關功能，業務部看業務相關的。

## Keycloak 的 Role 架構

Keycloak 的 Role 分兩種：

**Realm Role** 是全域性的角色，整個 realm 內的所有 client 都能用。比如 `admin`、`user` 這種通用角色。

**Client Role** 是特定 client 專屬的角色。比如 ERP 系統可能有 `erp-finance`、`erp-sales` 這些角色，其他系統用不到。

一開始我搞不清楚該用哪種。後來想了想，如果是多系統整合，每個系統的角色定義都不一樣，用 Client Role 比較適合。

## 在 Keycloak 建立角色

進入 `spring-app` 這個 client，點 Roles 頁籤：

1. 點 Create role
2. Role name 填 `finance-manager`
3. 再建一個 `sales-user`

然後把角色指派給使用者。進入 Users，選 `testuser`：

1. 切到 Role mapping 頁籤
2. 點 Assign role
3. 從 Filter by clients 選 `spring-app`
4. 勾選 `finance-manager`，點 Assign

這樣 `testuser` 就有了 `finance-manager` 這個角色。

## 讓 Spring Boot 讀取 Role

預設情況下，Spring Security 拿到的 OAuth2User 裡面沒有 Keycloak 的 roles。要做一些設定才能把 role 資訊帶進來。

首先在 Keycloak 調整 Client Scope：

1. 進入 `spring-app` client
2. 切到 Client scopes 頁籤
3. 點進 `spring-app-dedicated`
4. 點 Add mapper，選 By configuration
5. 選 User Client Role
6. 設定：
   - Name: `client-roles`
   - Client ID: `spring-app`
   - Token Claim Name: `resource_access.${client_id}.roles`
   - Add to userinfo: 打開

這樣 Access Token 裡就會包含 client roles 了。

然後修改 Spring Security 設定，寫一個 Converter 把 role 轉成 Spring Security 的 Authority：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()
                .requestMatchers("/finance/**").hasRole("finance-manager")
                .requestMatchers("/sales/**").hasRole("sales-user")
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(userInfo -> userInfo
                    .userAuthoritiesMapper(grantedAuthoritiesMapper())
                )
            );
        
        return http.build();
    }
    
    private GrantedAuthoritiesMapper grantedAuthoritiesMapper() {
        return (authorities) -> {
            Set<GrantedAuthority> mappedAuthorities = new HashSet<>();
            
            authorities.forEach(authority -> {
                if (authority instanceof OAuth2UserAuthority oauth2Authority) {
                    Map<String, Object> userAttributes = oauth2Authority.getAttributes();
                    
                    // 從 resource_access 取得 client roles
                    Map<String, Object> resourceAccess = 
                        (Map<String, Object>) userAttributes.get("resource_access");
                    if (resourceAccess != null) {
                        Map<String, Object> clientAccess = 
                            (Map<String, Object>) resourceAccess.get("spring-app");
                        if (clientAccess != null) {
                            List<String> roles = (List<String>) clientAccess.get("roles");
                            if (roles != null) {
                                roles.forEach(role -> 
                                    mappedAuthorities.add(new SimpleGrantedAuthority("ROLE_" + role))
                                );
                            }
                        }
                    }
                }
            });
            
            return mappedAuthorities;
        };
    }
}
```

這段 code 看起來有點複雜，主要是在從 OAuth2 的 token 裡挖出 `resource_access.spring-app.roles`，然後轉成 Spring Security 的 `GrantedAuthority`。

記得要加上 `ROLE_` 前綴，因為 Spring Security 的 `hasRole()` 會自動加這個前綴去比對。

## 測試權限控制

寫兩個測試用的 Controller：

```java
@Controller
public class FinanceController {
    
    @GetMapping("/finance/report")
    public String report(Model model) {
        model.addAttribute("message", "這是財務報表");
        return "finance-report";
    }
}

@Controller
public class SalesController {
    
    @GetMapping("/sales/orders")
    public String orders(Model model) {
        model.addAttribute("message", "這是訂單列表");
        return "sales-orders";
    }
}
```

用 `testuser`（有 finance-manager 角色）登入，可以存取 `/finance/report`，但訪問 `/sales/orders` 會被擋下來，出現 403 Forbidden。

符合預期！

## 在程式裡判斷角色

有時候不是整個頁面要擋，而是頁面內的某些功能要根據角色顯示或隱藏。可以在 Controller 裡用 `@PreAuthorize` 註解：

```java
@Controller
public class DashboardController {
    
    @GetMapping("/dashboard")
    public String dashboard(Model model, Authentication authentication) {
        model.addAttribute("username", authentication.getName());
        
        Collection<? extends GrantedAuthority> authorities = 
            authentication.getAuthorities();
        model.addAttribute("isFinanceManager", 
            authorities.stream().anyMatch(a -> a.getAuthority().equals("ROLE_finance-manager")));
        
        return "dashboard";
    }
}
```

或者用 Thymeleaf 的 Spring Security 整合：

```html
<div sec:authorize="hasRole('finance-manager')">
    <a href="/finance/report">財務報表</a>
</div>

<div sec:authorize="hasRole('sales-user')">
    <a href="/sales/orders">訂單管理</a>
</div>
```

這樣不同角色的使用者看到的選單就不一樣了。

## 一些心得

Keycloak 的角色機制其實蠻彈性的，除了 Realm Role 和 Client Role，還可以設定 Composite Role（組合角色），把多個角色組合成一個大角色。不過目前的需求還不需要用到這麼複雜的設定。

比較麻煩的是 Spring Security 和 Keycloak 之間的對接。OAuth2 的 token 格式、claim 的位置、authority 的轉換，這些都要自己處理。雖然有一些第三方的 adapter 可以用，但我還是想搞懂底層的原理，之後遇到問題才知道怎麼 debug。

下週想研究 Token 的細節，Access Token 和 Refresh Token 的生命週期、如何手動驗證 Token、如何在微服務之間傳遞 Token 等等。這些對系統架構蠻重要的。
