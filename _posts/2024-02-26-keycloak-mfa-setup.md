---
layout: post
title: "Keycloak 多因子認證設定"
date: 2024-02-26 14:20:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, MFA, OTP]
---

這週研究了 Keycloak 的多因子認證（MFA）功能。現在資安越來越被重視，單純帳號密碼已經不夠安全了，特別是一些敏感系統，加上第二層驗證會更保險。

Keycloak 內建支援多種 MFA 方式，包括 OTP（One-Time Password）、WebAuthn、SMS 等。這週先來試試最常見的 OTP。

## OTP 的運作原理

OTP 就是常見的「動態密碼」，使用者手機上裝個 App（像 Google Authenticator 或 Microsoft Authenticator），App 會每 30 秒產生一組 6 位數的驗證碼。

登入時除了輸入帳號密碼，還要輸入手機上顯示的驗證碼。就算密碼被偷了，沒有手機還是登不進去。

原理是基於 TOTP（Time-based One-Time Password），用共享的金鑰和當前時間戳來產生驗證碼，所以雙方只要時間同步，就能算出一樣的數字。

## 在 Keycloak 啟用 OTP

進入 `company` realm，點選 Authentication：

1. 切到 Flows 頁籤
2. 選擇 `browser` flow
3. 看到 `Browser - Conditional OTP` 這個步驟
4. 點旁邊的設定圖示
5. 把 `Requirement` 從 `CONDITIONAL` 改成 `REQUIRED`

這樣設定後，所有使用者登入都必須設定 OTP。不過這樣有點強制，實務上可能只有特定角色才需要 MFA。

## 更彈性的設定

如果只想讓特定使用者啟用 OTP，可以保持 `CONDITIONAL`，然後在 `Condition - User Configured` 設定裡調整。

或者改用 Required Actions：

1. 進入 Users，選擇要啟用 OTP 的使用者
2. 切到 Required User Actions 頁籤
3. 勾選 `Configure OTP`
4. 儲存

這樣使用者下次登入時，會被強制要求設定 OTP。

## 使用者設定 OTP

用 testuser 登入，會被導向 OTP 設定頁面：

1. 頁面上顯示一個 QR Code
2. 用手機的 Authenticator App 掃描 QR Code
3. App 會開始產生驗證碼
4. 把驗證碼輸入到 Keycloak 的輸入框
5. 設定完成

之後每次登入，在輸入完帳號密碼後，會多一個步驟要輸入 OTP 驗證碼。

## 備用碼

如果手機不見了怎麼辦？Keycloak 沒有內建備用碼功能，但可以自己實作。

或者管理員可以幫使用者重設 OTP：

1. 進入 Users，選擇該使用者
2. 切到 Credentials 頁籤
3. 找到 OTP 那一行，點刪除
4. 通知使用者重新設定

## 不同裝置的處理

Keycloak 預設不會記住已驗證的裝置，每次登入都要輸入 OTP。這樣很安全但有點麻煩。

可以調整 Authentication Flow，加入 Cookie 機制：

1. 複製 `browser` flow，建立一個新的 `browser-with-cookie`
2. 在流程中加入 `Cookie` 這個 Authenticator
3. 調整優先順序，讓 Cookie 在 OTP 之前
4. 設定 Cookie 的存活時間

這樣使用者在同一台電腦上，一段時間內不用重複輸入 OTP。

## WebAuthn 簡單測試

除了 OTP，Keycloak 也支援 WebAuthn，就是用硬體金鑰（像 YubiKey）或生物辨識（指紋、臉部辨識）來驗證。

在 Authentication 頁面可以啟用 `WebAuthn Authenticator`：

1. 把 `WebAuthn Authenticator` 的 Requirement 設為 `REQUIRED`
2. 使用者登入時會被要求註冊 WebAuthn 裝置

不過 WebAuthn 需要 HTTPS 才能運作，而且瀏覽器要支援。桌面瀏覽器通常沒問題，但一些老舊的環境可能不支援。

我在 MacBook 上測試，用 Touch ID 就能完成驗證，比輸入 OTP 快多了。不過這需要使用者的裝置支援，不是所有場景都適合。

## 實務考量

實際導入 MFA 時，有幾個要注意的：

**漸進式導入**：不要一次強制所有人都啟用，先從特權帳號開始，像是管理員、財務人員。觀察一陣子沒問題，再逐步擴大範圍。

**使用者教育**：很多使用者不知道怎麼用 Authenticator App，要準備好教學文件和影片。特別是年紀大的員工，可能需要一對一指導。

**緊急存取**：一定要有 backup plan。手機壞了、遺失了，要有辦法讓使用者還是能登入，或者至少能找管理員重設。

**相容性**：有些老舊系統可能不支援 MFA 的登入流程，這時可能要為這些系統設定例外規則，或者改用其他驗證方式。

## 記錄與監控

啟用 MFA 後，要監控使用情況：

- 有多少人完成 OTP 設定？
- 登入成功率有沒有下降？（可能是使用者不會用）
- 有沒有頻繁的重設請求？

Keycloak 的 Events 可以記錄這些資訊，進入 Realm settings → Events，可以看到所有的登入事件，包括 OTP 驗證成功或失敗。

## 下一步

MFA 的基本功能試過了，下週想研究社交登入整合，讓使用者可以用 Google、Microsoft 帳號登入。這在某些場景很實用，特別是對外的服務，不用強制使用者註冊新帳號。

另外也想看看 Keycloak 的事件和監控機制，怎麼整合到 ELK 或 Prometheus，做到即時告警。這些對正式環境運維很重要。
