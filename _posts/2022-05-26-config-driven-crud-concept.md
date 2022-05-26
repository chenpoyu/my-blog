---
layout: post
title: "Config-Driven 開發：用設定檔驅動 CRUD 頁面生成"
date: 2022-05-26 09:15:00 +0800
categories: [前端開發]
tags: [React, CRUD, Config-Driven, 開發效率]
---

## 後台系統的痛點

政府專案的管理後台有個特色：**大量重複的 CRUD 操作**。

每個功能模組都需要：
- 列表頁面（查詢、分頁、排序）
- 新增表單
- 編輯表單
- 刪除確認

如果每個模組都手刻一次，不僅浪費時間，維護起來也很痛苦。

## 從後端思維找靈感

在後端開發中，我們常用「配置驅動」的方式來減少重複程式碼。例如：
- Spring Boot 的 `application.yml`
- Hibernate 的 Entity 配置

這讓我們想到：**前端是否也能用類似的方式？**

## Config-Driven CRUD 的核心概念

我們設計了一套「設定檔驅動頁面」的架構：

1. **撰寫 JSON 配置檔**：定義欄位、驗證規則、API 路徑
2. **通用元件讀取配置**：自動產生列表與表單
3. **邏輯全部抽象**：分頁、排序、CRUD 操作都已實作

開發者只需要專注在「這個頁面有哪些欄位」，而不用重複寫分頁、表單驗證等邏輯。

## 配置檔範例

以「使用者管理」為例，我們的配置檔長這樣：

```javascript
// configs/userConfig.js
export const userConfig = {
  // API 路徑
  apiPath: '/api/users',
  
  // 頁面標題
  title: '使用者管理',
  
  // 列表欄位
  columns: [
    { key: 'id', label: 'ID', sortable: true },
    { key: 'name', label: '姓名', searchable: true },
    { key: 'email', label: 'Email', searchable: true },
    { key: 'role', label: '角色', type: 'select', options: ['admin', 'user'] },
    { key: 'createdAt', label: '建立時間', type: 'date' }
  ],
  
  // 表單欄位
  formFields: [
    { 
      name: 'name', 
      label: '姓名', 
      type: 'text', 
      required: true,
      rules: { minLength: 2, maxLength: 50 }
    },
    { 
      name: 'email', 
      label: 'Email', 
      type: 'email', 
      required: true 
    },
    { 
      name: 'role', 
      label: '角色', 
      type: 'select', 
      options: ['admin', 'user'],
      required: true 
    }
  ]
};
```

## 頁面元件使用配置

有了配置檔後，頁面元件只需要這樣寫：

```javascript
// pages/UserPage.jsx
import CrudPage from '@/components/CrudPage';
import { userConfig } from '@/configs/userConfig';

function UserPage() {
  return <CrudPage config={userConfig} />;
}

export default UserPage;
```

就這樣，一個完整的 CRUD 頁面就完成了！

## 何時需要客製化

並非所有頁面都適合用配置生成。當遇到以下情況，我們會選擇手刻：

- **複雜的互動邏輯**：例如多步驟表單、欄位間相互影響
- **特殊的視覺需求**：超出 Ant Design 預設樣式的設計
- **效能敏感頁面**：需要精細控制渲染的場景

大約 70% 的後台頁面都能用配置完成，剩下 30% 才需要客製化開發。

## 下一步

接下來會分享 `CrudPage` 元件的實作細節，以及如何處理分頁、排序、篩選等常見需求。

這套架構讓我們在專案中快速產出功能，也讓團隊成員能更專注在業務邏輯而非重複勞動。
