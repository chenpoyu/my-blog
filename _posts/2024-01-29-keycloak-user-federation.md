---
layout: post
title: "整合既有的 LDAP 到 Keycloak"
date: 2024-01-29 16:50:00 +0800
categories: [系統架構, 安全性]
tags: [Keycloak, LDAP, User Federation]
---

這週研究了 Keycloak 的 User Federation 功能。大多數企業其實已經有 LDAP 或 Active Directory 在管理員工帳號了，如果能直接整合，就不用再建一套使用者資料。

Keycloak 的 User Federation 可以整合外部的使用者資料來源，包括 LDAP 和 Active Directory。這樣使用者資料還是存在原本的系統，Keycloak 只是當作一個「門面」來做認證。

## 測試用的 LDAP Server

為了測試，先在本機用 Docker 跑一個測試用的 OpenLDAP：

```bash
docker run -d \
  --name openldap \
  -p 389:389 \
  -e LDAP_ORGANISATION="My Company" \
  -e LDAP_DOMAIN="mycompany.com" \
  -e LDAP_ADMIN_PASSWORD="admin123" \
  osixia/openldap:1.5.0
```

然後用 Apache Directory Studio 連上去，手動建立一些測試資料：

```
dc=mycompany,dc=com
├── ou=users
│   ├── uid=john.doe,ou=users,dc=mycompany,dc=com
│   │   - cn: John Doe
│   │   - mail: john.doe@mycompany.com
│   │   - userPassword: password123
│   └── uid=jane.smith,ou=users,dc=mycompany,dc=com
│       - cn: Jane Smith
│       - mail: jane.smith@mycompany.com
│       - userPassword: password123
└── ou=groups
    ├── cn=finance,ou=groups,dc=mycompany,dc=com
    └── cn=sales,ou=groups,dc=mycompany,dc=com
```

LDAP 的結構跟關聯式資料庫很不一樣，是樹狀的階層結構。一開始看得有點暈，不過大概理解了 `dc`（domain component）、`ou`（organizational unit）、`uid`（user id）這些概念。

## 在 Keycloak 設定 LDAP Federation

進入 Keycloak 管理介面，`company` realm：

1. 點選 User federation
2. 選 Add LDAP provider
3. 填入設定：
   - **Console Display Name**: `Company LDAP`
   - **Vendor**: `Other` （如果是 AD 就選 Active Directory）
   - **Connection URL**: `ldap://localhost:389`
   - **Users DN**: `ou=users,dc=mycompany,dc=com`
   - **Bind DN**: `cn=admin,dc=mycompany,dc=com`
   - **Bind Credential**: `admin123`

然後往下找到 **Authentication Settings**：
   - **Edit Mode**: 先選 `READ_ONLY`（只讀，不會改到 LDAP 的資料）
   - **Username LDAP attribute**: `uid`
   - **RDN LDAP attribute**: `uid`
   - **UUID LDAP attribute**: `entryUUID`

儲存後，點 Test connection 和 Test authentication，確認能連上 LDAP。

## 同步 LDAP 使用者

設定好後，Keycloak 還不會自動把 LDAP 的使用者抓進來。要手動觸發同步：

1. 在 User federation 頁面，找到剛建立的 `Company LDAP`
2. 點 Synchronize all users

等一下，回到 Users 選單，就能看到從 LDAP 同步過來的使用者了：`john.doe` 和 `jane.smith`。

## 測試 LDAP 登入

用 `john.doe` / `password123` 登入我們的 Spring Boot 應用。成功了！Keycloak 會去 LDAP 驗證密碼，驗證通過就發 Token 給我們。

神奇的是，在 Keycloak 的 Users 列表裡雖然看得到這些使用者，但實際資料還是存在 LDAP，Keycloak 只是做了一層快取。

## LDAP Group Mapping

接下來要處理角色的問題。LDAP 裡有 groups（finance、sales），我們要把它對應到 Keycloak 的 roles。

在 LDAP provider 設定頁面，切到 Mappers 頁籤：

1. 點 Add mapper
2. 選 group-ldap-mapper
3. 填入設定：
   - **Name**: `group-mapper`
   - **LDAP Groups DN**: `ou=groups,dc=mycompany,dc=com`
   - **Group Name LDAP Attribute**: `cn`
   - **Group Object Classes**: `groupOfNames`
   - **Membership LDAP Attribute**: `member`
   - **Mode**: `READ_ONLY`

這樣設定後，Keycloak 會去讀 LDAP 的 groups，並且把使用者的 group membership 對應成 Keycloak 的 groups。

不過這邊要注意，LDAP 的 groups 會對應到 Keycloak 的 groups，不是 roles。如果要對應到 roles，還要再設定 Group-Role mapping。

進入 Groups，可以看到從 LDAP 同步過來的 `finance` 和 `sales` group。點進去，切到 Role mapping 頁籤，把它對應到我們之前建立的 client roles。

這樣 LDAP 的 group 就能對應到 Spring Boot 應用的角色了。

## Read-Only vs Writable

一開始我把 Edit Mode 設成 `READ_ONLY`，這樣比較安全，不會不小心改到 LDAP 的資料。

但如果希望使用者可以在系統裡改自己的 email 或個人資料，然後同步回 LDAP，就要改成 `WRITABLE` 或 `UNSYNCED`。

`WRITABLE` 模式下，使用者在 Keycloak 改的資料會寫回 LDAP。`UNSYNCED` 則是改動只存在 Keycloak，不會同步回去。

測試階段先用 `READ_ONLY`，之後確認沒問題再改成 `WRITABLE`。畢竟 LDAP 通常是企業的核心系統，不能隨便動。

## 遇到的問題

LDAP 的設定真的很容易出錯。DN（Distinguished Name）的格式要完全正確，多一個空格或少一個逗號都不行。

一開始我 Users DN 設成 `ou=users, dc=mycompany, dc=com`（逗號後面有空格），結果一直連不上，查了好久才發現。

另外 LDAP 的 schema 每家都不太一樣，OpenLDAP、Active Directory、FreeIPA 的 attribute 名稱和 object classes 都有差異。要針對實際環境調整。

還有效能問題。如果 LDAP 有幾萬筆使用者資料，全部同步會很慢。可以設定 Pagination（分頁），或者只同步特定 OU 底下的使用者。

## 下一步

LDAP 整合的基本功能通了，但還有一些進階設定要研究：

使用者屬性的對應，比如 LDAP 的 `telephoneNumber` 要對應到 Keycloak 的哪個欄位。

密碼變更的處理，使用者在系統裡改密碼，要怎麼同步回 LDAP。

還有 LDAP 的高可用性，如果企業的 LDAP 有多台，要怎麼設定 failover。

不過這些可以之後再調整。目前先把基本整合做起來，確認沒問題再來處理細節。這週總算把 User Federation 這個難關過了。
