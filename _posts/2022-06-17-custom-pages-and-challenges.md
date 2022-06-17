---
layout: post
title: "當 Config 不夠用：複雜頁面的客製化策略"
date: 2022-06-17 10:00:00 +0800
categories: [前端開發]
tags: [React, 架構設計, 彈性設計]
---

## 通用方案的極限

Config-Driven 的 CRUD 系統雖然能處理大部分頁面，但實務上總會遇到「配置搞不定」的情況。

上週我們遇到一個需求：**訂單管理頁面需要顯示多個子表單，且子表單的內容會根據訂單狀態動態變化。**

這種複雜的業務邏輯，硬塞進配置檔會變得非常難維護。

## 何時該放棄配置

我們歸納出幾個「需要客製化」的訊號：

1. **欄位間有複雜的相依關係**：選擇 A 會影響 B 和 C 的可選項
2. **多步驟操作**：例如新增訂單需要先選商品、再填數量、最後確認
3. **特殊的資料展示**：圖表、時間軸、樹狀結構等
4. **非典型的操作流程**：批次處理、拖曳排序、即時預覽

遇到這些情況，強行用配置實作只會讓程式碼更複雜。

## 混合策略：部分使用配置

即使需要客製化，我們也不會完全拋棄通用元件。通常會採用「混合策略」：

```javascript
// pages/OrderPage.jsx
import { useState } from 'react';
import DataTable from '@/components/DataTable';
import CustomOrderForm from './components/CustomOrderForm';
import { useDataTable } from '@/hooks/useDataTable';

function OrderPage() {
  // 列表部分仍使用通用邏輯
  const {
    data,
    loading,
    pagination,
    handleTableChange,
    handleSearch,
    refresh
  } = useDataTable('/api/orders');

  const [modalVisible, setModalVisible] = useState(false);
  const [editingOrder, setEditingOrder] = useState(null);

  return (
    <div>
      <h2>訂單管理</h2>
      
      {/* 列表使用通用元件 */}
      <DataTable
        columns={orderColumns}
        data={data}
        loading={loading}
        pagination={pagination}
        onChange={handleTableChange}
        onEdit={(record) => {
          setEditingOrder(record);
          setModalVisible(true);
        }}
      />
      
      {/* 表單使用客製化元件 */}
      <CustomOrderForm
        visible={modalVisible}
        order={editingOrder}
        onClose={() => setModalVisible(false)}
        onSuccess={refresh}
      />
    </div>
  );
}
```

**重點**：列表與查詢邏輯仍用通用方案，只有表單部分客製化。

## 擴充點設計

為了讓配置系統更有彈性，我們在 `CrudPage` 加入幾個「擴充點」：

```javascript
function CrudPage({ 
  config,
  // 新增的擴充點
  customSearchForm,      // 自訂查詢表單
  customFormModal,       // 自訂新增/編輯表單
  customActions,         // 自訂操作按鈕
  beforeSubmit,          // 提交前的資料處理
  afterFetch             // 取得資料後的處理
}) {
  // ...
  
  return (
    <div>
      {/* 如果有提供自訂查詢表單，就用自訂的 */}
      {customSearchForm || <SearchForm columns={config.columns} />}
      
      {customActions || <Button onClick={handleCreate}>新增</Button>}
      
      <DataTable {...tableProps} />
      
      {customFormModal || <FormModal {...formProps} />}
    </div>
  );
}
```

這樣一來，需要客製化的部分可以逐一替換，不需要整個重寫。

## 實際案例：報表匯出功能

有個需求是「匯出 Excel 前需要選擇欄位與日期區間」，這不在標準 CRUD 流程中。

我們的做法是擴充 `customActions`：

```javascript
<CrudPage
  config={reportConfig}
  customActions={
    <>
      <Button onClick={handleCreate}>新增</Button>
      <Button onClick={handleExport} icon={<DownloadOutlined />}>
        匯出報表
      </Button>
    </>
  }
/>
```

匯出的邏輯寫在 `handleExport` 中，不影響原本的 CRUD 操作。

## 團隊共識：70/30 原則

經過幾週的實戰，我們達成共識：

- **70% 的頁面**：用配置完成，快速產出
- **30% 的頁面**：部分或全部客製化，保持彈性

重點不是「所有頁面都要用配置」，而是「讓簡單的事情保持簡單」。

## 遇到的挑戰

這套架構也不是沒有問題：

1. **學習成本**：新成員需要時間理解配置結構
2. **除錯困難**：出問題時需要追蹤通用元件的邏輯
3. **過度抽象**：有時候簡單的頁面反而多了一層配置

我們的應對方式是：
- 寫清楚的文件與範例
- 在通用元件中加入詳細的錯誤訊息
- 不強制使用，讓團隊自行評估

## 小結

從 Vue 轉到 React 的這個月，最大的收穫不是學會新框架，而是重新思考「如何設計可擴充的系統」。

配置驅動的方式確實提升了開發效率，但更重要的是，我們建立了一套「在通用與彈性之間取得平衡」的開發文化。

下週會分享一些實際遇到的 React Hooks 使用心得與效能優化技巧。
