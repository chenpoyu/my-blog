---
layout: post
title: "客製化 Keycloak 登入頁面"
date: 2024-02-12 13:55:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, Theme, UI]
---

這週研究了 Keycloak 的 UI 客製化。實務上，企業通常會希望把 Keycloak 的登入頁面改成公司的品牌風格。預設的畫面太「開源」了，看起來不夠專業。

研究了一下，Keycloak 支援自訂 Theme（主題），可以改登入頁、註冊頁、帳號管理頁面等等的樣式和內容。

## Keycloak Theme 結構

Keycloak 的 theme 分成幾種類型：

- **login**：登入相關頁面（登入、註冊、忘記密碼等）
- **account**：使用者帳號管理頁面
- **admin**：管理介面
- **email**：系統寄出的 email 樣板

我們主要要改的是 login theme。

Theme 的檔案結構：

```
themes/
  └── my-company/
      ├── login/
      │   ├── resources/
      │   │   ├── css/
      │   │   │   └── login.css
      │   │   └── img/
      │   │       └── logo.png
      │   ├── messages/
      │   │   └── messages_zh_TW.properties
      │   └── theme.properties
      └── ...
```

每個 theme 要有 `theme.properties` 設定檔，指定要繼承哪個 parent theme 和其他設定。

## 建立自訂 Theme

如果是用 Docker 跑 Keycloak，要把 theme 檔案掛載進去。我先在本機建立 theme 資料夾：

```bash
mkdir -p keycloak-themes/my-company/login/resources/css
mkdir -p keycloak-themes/my-company/login/resources/img
```

建立 `theme.properties`：

```properties
parent=keycloak
import=common/keycloak
styles=css/login.css
```

這邊設定繼承 keycloak 預設 theme，然後載入我們自己的 CSS。

寫一個簡單的 `login.css`：

```css
.login-pf body {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.login-pf .card-pf {
    border-radius: 10px;
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
}

.login-pf .card-pf h1 {
    color: #667eea;
    font-weight: 600;
}

#kc-header-wrapper {
    padding: 30px 0 20px;
}

#kc-header-wrapper img {
    max-height: 60px;
}

.btn-primary {
    background-color: #667eea;
    border-color: #667eea;
}

.btn-primary:hover {
    background-color: #5568d3;
    border-color: #5568d3;
}
```

放一張公司 logo 到 `resources/img/logo.png`。

## 掛載 Theme 到 Docker

修改 Docker 啟動指令，把 theme 資料夾掛載進去：

```bash
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin123 \
  -v $(pwd)/keycloak-themes:/opt/keycloak/themes \
  quay.io/keycloak/keycloak:23.0 \
  start-dev
```

重啟 Keycloak 後，進入管理介面：

1. 選擇 `company` realm
2. 進入 Realm settings
3. 切到 Themes 頁籤
4. Login theme 選 `my-company`
5. 儲存

現在登出，再重新登入，就能看到套用了新 theme 的登入頁面。背景變成漸層紫色，登入框有圓角和陰影，看起來專業多了！

## 修改頁面內容

除了 CSS，還可以改頁面的 HTML 結構和文字內容。Keycloak 用 FreeMarker 當樣板引擎。

如果要改登入頁的 HTML，要建立 `login.ftl`：

```bash
mkdir -p keycloak-themes/my-company/login
```

然後從 Keycloak 的預設 theme 複製 `login.ftl` 過來修改。預設 theme 在 Keycloak 的安裝目錄下，或者可以從 GitHub 上找：

```
https://github.com/keycloak/keycloak/tree/main/themes/src/main/resources/theme/base/login
```

把 `login.ftl` 複製到我們的 theme 資料夾，然後修改：

```html
<#-- 在 form 前面加上 logo -->
<div id="kc-header">
    <div id="kc-header-wrapper">
        <img src="${url.resourcesPath}/img/logo.png" alt="Company Logo" />
    </div>
</div>

<#-- 原本的 form -->
<form id="kc-form-login" ...>
    ...
</form>
```

重新整理頁面，公司 logo 就出現在登入框上方了。

## 中文化

Keycloak 預設是英文，但支援多國語言。要改成中文，建立 `messages/messages_zh_TW.properties`：

```properties
loginTitle=登入您的帳號
usernameOrEmail=使用者名稱或電子郵件
password=密碼
doLogIn=登入
doForgotPassword=忘記密碼？
doRegister=註冊新帳號
invalidUsernameOrPasswordMessage=帳號或密碼錯誤
accountDisabledMessage=帳號已停用，請聯絡管理員
```

然後在 Realm settings → Localization 新增支援的語系：

1. 點 Add supported locales
2. 選 `繁體中文 (zh-TW)`
3. Default locale 也設成 `zh-TW`

這樣登入頁就會顯示中文了。

## 遇到的問題

一開始 theme 檔案放好了，但 Keycloak 一直讀不到。後來發現是 Docker volume 的路徑問題，要確認掛載的路徑是對的：

```bash
docker exec keycloak ls -la /opt/keycloak/themes
```

確認 theme 資料夾有正確出現在 container 裡。

另外 Keycloak 會快取 theme，改了 CSS 之後要清快取才會生效。在 Realm settings → Cache 可以清除快取，或者在開發模式下可以設定不要快取：

```bash
docker run ... \
  -e KC_SPI_THEME_STATIC_MAX_AGE=-1 \
  -e KC_SPI_THEME_CACHE_THEMES=false \
  -e KC_SPI_THEME_CACHE_TEMPLATES=false \
  ...
```

這樣改完 CSS 重新整理就會生效，不用一直清快取。

## 更進階的客製化

如果要更大幅度的改動，比如加上公司的服務條款、改變整個頁面布局，就要直接修改 FreeMarker 樣板。

Keycloak 的樣板蠻複雜的，有很多條件判斷和變數。建議先從預設 theme 複製過來，然後一點一點改，不要整個重寫。

還可以用 JavaScript 加上一些動態效果，或者整合 Google reCAPTCHA 防止機器人登入。這些都可以在 theme 裡做。

## 實際效果

把客製化的 theme 套用後，登入頁面看起來專業多了。可以依需求調整登入按鈕的顏色、錯誤訊息的用詞等等。這些都是小改動，改改 CSS 和 properties 就好。

客製化 theme 這個功能真的很實用，對企業應用來說很重要。使用者不會覺得是被「踢」到另一個系統去登入，整個體驗是一致的。

## 總結

## 小結

經過這幾週的研究，Keycloak 的基本功能都摸得差不多了：

- 安裝和基本設定
- Spring Boot 整合
- 角色權限管理
- Token 機制
- LDAP 整合
- Admin API 自動化
- Theme 客製化

不過離正式環境還有一段距離。測試環境用的是開發模式，資料存在內建的 H2 資料庫。正式環境要考慮的東西更多：高可用性、資料庫選擇、HTTPS 設定、效能調校等等。下週來研究正式環境的部署方式。