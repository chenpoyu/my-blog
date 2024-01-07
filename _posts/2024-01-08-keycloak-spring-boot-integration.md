---
layout: post
title: "把 Spring Boot 接上 Keycloak"
date: 2024-01-08 09:15:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, Spring Boot, OAuth2]
---

上週把 Keycloak 跑起來了，這週要來試著讓 Spring Boot 應用程式接上去。說實話，一開始有點頭痛，OAuth 2.0 的流程跟之前的 Session 登入差很多。

## 在 Keycloak 註冊 Client

要讓 Spring Boot 應用程式使用 Keycloak，首先要在 Keycloak 把它註冊成一個 Client：

1. 進入 `company` realm
2. 點選 Clients，然後 Create client
3. **Client ID** 填 `spring-app`
4. **Client Protocol** 選 `openid-connect`
5. 下一步，**Client authentication** 打開（因為我們是後端應用）
6. **Valid redirect URIs** 填 `http://localhost:8081/*`
7. 儲存

儲存後會產生一個 Client Secret，這個要記下來，等下設定檔會用到。

## Spring Boot 專案設定

在 `pom.xml` 加入 Spring Security 和 OAuth2 Client 的依賴：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

然後在 `application.yml` 設定 Keycloak 的連線資訊：

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: spring-app
            client-secret: 你的-client-secret
            scope: openid, profile, email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          keycloak:
            issuer-uri: http://localhost:8080/realms/company
            user-name-attribute: preferred_username

server:
  port: 8081
```

這邊 `issuer-uri` 很重要，Spring Security 會自動去這個網址抓 OpenID Connect 的設定資訊。

## Security Configuration

接著寫一個 SecurityConfig：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .defaultSuccessUrl("/dashboard", true)
            );
        
        return http.build();
    }
}
```

這個設定的意思是：
- 首頁和 `/public/**` 路徑不用登入
- 其他路徑都要登入
- 使用 OAuth2 登入，登入成功後導向 `/dashboard`

## 寫一個測試頁面

簡單寫個 Controller 測試：

```java
@Controller
public class HomeController {
    
    @GetMapping("/")
    public String home() {
        return "home";
    }
    
    @GetMapping("/dashboard")
    public String dashboard(Model model, 
                           @AuthenticationPrincipal OAuth2User principal) {
        model.addAttribute("username", principal.getAttribute("preferred_username"));
        model.addAttribute("email", principal.getAttribute("email"));
        return "dashboard";
    }
}
```

`dashboard.html` 用 Thymeleaf 寫：

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Dashboard</title>
</head>
<body>
    <h1>歡迎回來！</h1>
    <p>使用者名稱: <span th:text="${username}"></span></p>
    <p>Email: <span th:text="${email}"></span></p>
    
    <form th:action="@{/logout}" method="post">
        <button type="submit">登出</button>
    </form>
</body>
</html>
```

## 測試登入流程

啟動 Spring Boot 應用，開瀏覽器連 `http://localhost:8081/dashboard`。因為沒登入，會自動跳轉到 Keycloak 的登入頁面。

用上週建的 `testuser` 登入後，就會導回我們的 `/dashboard`，可以看到使用者名稱和 email 顯示出來了。

點登出按鈕，會回到首頁。不過這邊有個小問題：雖然應用程式登出了，但 Keycloak 那邊的 Session 還在。如果馬上再登入，會直接通過，不會再問密碼。這個之後要處理。

## 遇到的問題

一開始設定時，一直出現 `redirect_uri_mismatch` 錯誤。查了半天才發現是 Keycloak Client 設定的 Valid redirect URIs 不對。

Spring Security OAuth2 預設的 redirect URI 格式是 `{baseUrl}/login/oauth2/code/{registrationId}`，所以要在 Keycloak 設定 `http://localhost:8081/login/oauth2/code/keycloak` 或用萬用字元 `http://localhost:8081/*`。

另外一個問題是 `issuer-uri`，一開始我填 `http://localhost:8080`，結果一直連不上。後來才知道要加上 `/realms/company` 這個 realm 的路徑。

## 下一步

現在基本的登入流程通了，但實務上還有很多問題要解決：

使用者登入後，我們要怎麼知道他有哪些權限？Keycloak 的 Role 要怎麼對應到 Spring Security 的 Authority？

還有 Token 的問題。OAuth2 登入成功後會拿到 Access Token 和 Refresh Token，這些要怎麼用？如果要呼叫其他微服務的 API，Token 要怎麼傳遞？

這些都是下週要研究的方向。目前先把登入流程打通，基本功能可以運作了。
