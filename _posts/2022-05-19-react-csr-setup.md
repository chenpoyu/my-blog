---
layout: post
title: "React CSR 環境建置：快速啟動管理後台專案"
date: 2022-05-19 14:20:00 +0800
categories: [前端開發]
tags: [React, CSR, Create React App, 專案架構]
---

## 選擇 Create React App

在確定採用 CSR 方案後，我們選擇 Create React App (CRA) 作為專案基礎。雖然市面上有 Vite 等更新的工具，但考量到：

- 官方維護，穩定性高
- 文件完整，團隊學習成本低
- 預設配置已足夠應付後台系統需求

對政府專案來說，「穩」比「新」更重要。

## 專案結構規劃

我們根據過去 Vue 專案的經驗，設計了以下目錄結構：

```
src/
├── components/       # 共用元件
├── pages/           # 頁面元件
├── configs/         # 頁面配置檔（重點！）
├── services/        # API 呼叫
├── utils/           # 工具函式
├── hooks/           # 自訂 Hooks
└── styles/          # 全域樣式
```

其中 `configs/` 目錄是我們這次架構的核心，之後會詳細說明。

## 技術選型

除了 React 本身，我們還導入以下技術：

**UI 框架**：Ant Design  
政府專案常用，元件齊全且符合視覺規範

**狀態管理**：React Context + useState  
後台系統狀態管理需求單純，不需要 Redux

**路由**：React Router v6  
標準配置，與 Vue Router 概念類似

**HTTP Client**：Axios  
團隊已熟悉，可沿用既有的 interceptor 設定

**表單處理**：React Hook Form  
效能好，減少不必要的重新渲染

## 與 Vue 開發的差異感受

剛從 Vue 轉到 React，最明顯的差異是：

1. **沒有模板語法**：一開始不習慣 JSX，但很快就適應了
2. **狀態管理更直接**：`useState` 比 `data()` 更直覺
3. **生命週期簡化**：`useEffect` 統一處理，概念更清晰
4. **彈性更高**：想怎麼寫就怎麼寫，但也需要更多規範

## 開發規範

為了避免「彈性太高」導致程式碼風格混亂，我們制定了幾個原則：

- 元件以功能命名，採用 PascalCase
- 自訂 Hook 必須以 `use` 開頭
- 頁面元件統一放在 `pages/` 目錄
- 共用邏輯優先寫成 Hook 而非 HOC

這些規範讓團隊在轉換過程中能快速達成共識。

## 小結

環境建置完成後，接下來就是重頭戲：如何設計一套「設定即開發」的 CRUD 系統。

這套系統將讓我們能用最少的程式碼，快速產出功能完整的管理介面。
