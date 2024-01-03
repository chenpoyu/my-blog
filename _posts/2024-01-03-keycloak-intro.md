---
layout: post
title: "Keycloak 初次接觸"
date: 2024-01-03 14:25:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, SSO, 身份認證]
---

最近因緣際會在研究企業級的身份驗證方案。現在很多企業都有多個系統，每個系統都有自己的登入機制，使用者要記一堆帳號密碼。如果能做到 Single Sign-On (SSO)，使用體驗會好很多。

研究了幾天，發現 Keycloak 好像是個不錯的選擇。它是 RedHat 開源的身份驗證和授權管理解決方案，功能蠻完整的。

## 為什麼需要 Keycloak

傳統的做法是每個系統各自管理使用者：

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public User login(String username, String password) {
        User user = userRepository.findByUsername(username);
        if (user != null && passwordEncoder.matches(password, user.getPassword())) {
            return user;
        }
        throw new AuthenticationException("帳號或密碼錯誤");
    }
}
```

每個系統都要自己處理：
- 使用者註冊
- 密碼加密存儲
- Session 管理
- 密碼重設
- 權限控制

當系統一多，問題就來了。使用者在 A 系統改了密碼，B 系統還是舊密碼。要離職的員工，得去每個系統把帳號停用。維護起來超級麻煩。

## Keycloak 基本概念

花了一個下午看文件，整理出幾個核心概念：

**Realm** 就像是一個獨立的租戶空間，可以想像成是一個完整的使用者管理系統。一個企業可能會建一個 `company-realm`，裡面管理所有員工的帳號。

**Client** 代表要接入 Keycloak 的應用系統。像是 ERP、CRM、OA 這些系統，每個都要註冊成一個 Client。

**User** 就是使用者帳號，這個比較直覺。

**Role** 是角色，用來做權限控制。可以分成 Realm Role（全域角色）和 Client Role（特定系統的角色）。

## 安裝 Keycloak

Keycloak 提供 Docker 映像檔，測試環境用 Docker 跑最快：

```bash
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin123 \
  quay.io/keycloak/keycloak:23.0 \
  start-dev
```

等個幾分鐘，開瀏覽器連 `http://localhost:8080`，就能看到 Keycloak 的管理介面。用 admin/admin123 登入後，介面還蠻直覺的。

## 建立第一個 Realm

登入後預設是在 `master` realm，這是管理用的，不該拿來放業務使用者。所以要建一個新的 realm：

1. 點左上角的 `master` 下拉選單
2. 選 `Create Realm`
3. 輸入名稱 `company`，按 Create

然後在這個 realm 裡建立測試用的使用者：

1. 進入 Users 選單
2. 點 `Add user`
3. 填入 Username: `testuser`
4. 儲存後切到 Credentials �籤
5. 設定密碼，把 Temporary 關掉（不然第一次登入會強制改密碼）

現在有了一個 realm 和一個使用者，但還不能讓我們的應用系統使用。下週要研究怎麼讓 Spring Boot 接進來。

## 一些觀察

Keycloak 的管理介面功能很多，光是設定選項就看得眼花。不過基本的使用者管理倒是很簡單，就是新增、編輯、刪除這些操作。

比較特別的是它支援很多種身份驗證協議：OpenID Connect、OAuth 2.0、SAML 2.0。以前都是用傳統的 Session-based 登入，現在要轉成 OAuth 2.0，需要重新理解一下流程。

另外發現 Keycloak 可以整合既有的 LDAP 或 Active Directory，這對企業來說很重要。很多企業已經有 AD 或 LDAP 在管理員工帳號，不用重複建立一次。

不過這次只是跑起來看看，很多細節還不懂。像是 Token 怎麼驗證、權限怎麼對應到系統的功能、Session 要怎麼同步...這些都要慢慢研究。

下週先把 Spring Boot 整合做起來，讓使用者可以透過 Keycloak 登入應用程式。
