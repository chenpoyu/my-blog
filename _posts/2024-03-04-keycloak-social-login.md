---
layout: post
title: "Keycloak 整合社交登入"
date: 2024-03-04 09:45:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, OAuth, Social Login]
---

這週研究了 Keycloak 的社交登入功能，讓使用者可以用 Google、Microsoft、Facebook 等帳號登入。這在某些場景蠻實用的，特別是對外的服務，不用強制使用者註冊新帳號。

不過這功能要小心用，企業內部系統通常還是要用自己的帳號體系，但如果是 Partner Portal 或客戶服務平台，社交登入就很方便。

## Identity Provider 的概念

在 Keycloak 裡，外部的登入提供者叫做 Identity Provider（IdP）。Keycloak 本身也是一個 IdP，但它可以串接其他 IdP。

運作流程是這樣：

1. 使用者點「用 Google 登入」
2. 導向 Google 的登入頁面
3. 使用者在 Google 登入
4. Google 回傳使用者資訊給 Keycloak
5. Keycloak 建立或更新使用者，發 Token

對應用程式來說，不管使用者是用密碼登入還是 Google 登入，拿到的都是 Keycloak 的 Token，不用改任何程式碼。

## 設定 Google Identity Provider

要用 Google 登入，首先要去 Google Cloud Console 註冊一個 OAuth 2.0 Client：

1. 進入 https://console.cloud.google.com
2. 建立新專案（或用既有的）
3. 啟用 Google+ API
4. 建立 OAuth 2.0 認證資訊
5. 設定 Authorized redirect URIs：`http://localhost:8080/realms/company/broker/google/endpoint`

Google 會給你 Client ID 和 Client Secret，記下來。

然後在 Keycloak 設定：

1. 進入 `company` realm
2. 點選 Identity Providers
3. 從下拉選單選 `Google`
4. 填入剛才的 Client ID 和 Client Secret
5. 儲存

這樣就設定好了！

## 測試 Google 登入

登出後重新連到登入頁面，會看到多了一個「Sign in with Google」的按鈕。

點下去，會跳轉到 Google 的登入頁面。用 Google 帳號登入後，第一次會詢問是否授權給 Keycloak 存取基本資料（名字、email）。

授權後就導回應用程式，登入成功！

在 Keycloak 的 Users 列表裡，可以看到這個 Google 使用者被自動建立了，username 是 `google_<google-id>` 這種格式。

## 帳號連結

如果使用者已經有 Keycloak 帳號，又用 Google 登入，會變成兩個獨立的帳號。這時可以用帳號連結（Account Linking）功能。

使用者可以在帳號管理頁面（Account Console）連結多個 IdP：

1. 連到 `http://localhost:8080/realms/company/account`
2. 登入 Keycloak 帳號
3. 切到 `Linked Accounts`
4. 點 Google 旁邊的 Link
5. 用 Google 登入

這樣兩個帳號就連結起來了，之後用任一種方式登入都會是同一個使用者。

## First Login Flow

第一次用社交登入時，Keycloak 可以要求使用者補充一些資訊。進入 Identity Providers → Google → First Login Flow，可以設定：

- 要不要讓使用者檢視/修改從 Google 拿到的資訊
- 要不要要求使用者同意服務條款
- 要不要自動產生 username 還是讓使用者自己輸入

預設是 `first broker login` flow，會顯示一個確認頁面讓使用者檢視資料。如果想更流暢，可以改成 `direct grant` 模式，直接建立使用者。

## 屬性對應

Google 回傳的使用者資訊包括 email、name、picture 等，Keycloak 要把這些資訊對應到自己的使用者屬性。

在 Identity Providers → Google → Mappers 可以設定：

1. 點 Add mapper
2. 選 `Attribute Importer`
3. 設定：
   - Name: `email mapper`
   - Social Profile JSON Field Path: `email`
   - User Attribute Name: `email`

這樣 Google 的 email 就會自動填到 Keycloak 使用者的 email 欄位。

預設的 mapper 通常夠用，但如果有特殊需求，像是要把 Google 的某個欄位對應到自訂的 user attribute，就可以自己加。

## 其他 Identity Providers

除了 Google，Keycloak 內建支援很多 IdP：

- **Microsoft**：用 Microsoft 帳號登入
- **Facebook**：用 Facebook 登入
- **GitHub**：用 GitHub 登入
- **OpenID Connect**：任何支援 OIDC 的 IdP
- **SAML**：企業級的 IdP，像是 Azure AD、Okta

設定方式都類似，去對應的平台註冊 OAuth Client，拿到 Client ID 和 Secret，然後填到 Keycloak。

我試了一下 Microsoft 登入，設定方式幾乎一樣，只是要去 Azure Portal 註冊應用程式。

## 安全性考量

社交登入要注意幾個安全問題：

**Email 驗證**：從 Google 拿到的 email 通常已經驗證過了，但不是所有 IdP 都會驗證。要檢查 `email_verified` claim，避免有人用未驗證的 email 註冊。

**帳號接管**：如果同時支援密碼登入和社交登入，要確保不會有帳號接管的問題。比如有人先用 email 註冊，後來另一個人用同樣 email 的 Google 帳號登入，不應該自動連結，要有確認機制。

**IdP 的信任**：要確保只有信任的 IdP 能用。不要隨便讓使用者自己加 IdP，不然可能會有安全漏洞。

**Scope 控制**：向 IdP 請求資料時，只要必要的 scope。比如不需要存取使用者的通訊錄或日曆，就不要要求這些權限。

## 實務經驗

試了幾天社交登入，有些心得：

對外的系統很適合用社交登入，可以降低註冊門檻。但要準備好處理各種邊界情況，像是使用者用不同 IdP 登入但 email 相同、使用者的 email 變更等等。

企業內部系統通常不適合社交登入，因為要控制誰能登入。不過如果是 Partner 或供應商的入口網站，可以考慮用 Microsoft/Google 的企業帳號，再配合 domain 白名單。

UI/UX 要設計好，讓使用者清楚知道用哪種方式登入。如果支援太多 IdP，頁面會很亂。通常放 1-2 個主要的就好。

下週想研究 Keycloak 的 Event 和 Admin Event，看怎麼把這些日誌整合到監控系統，做到登入異常告警。
