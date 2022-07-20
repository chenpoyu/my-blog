---
layout: post
title: "React 測試策略：從零建立測試文化"
date: 2022-07-21 14:45:00 +0800
categories: [前端開發]
tags: [React, Testing, Jest, React Testing Library]
---

## 測試的缺席

轉換到 React 的前兩個月，我們完全沒寫測試。

原因很簡單：
- 專案時程緊迫
- 團隊對 React 還不熟悉
- 不確定該怎麼測前端

但隨著功能越來越多，我們開始遇到問題：
- 改 A 功能，B 功能壞掉
- 重構時不確定有沒有影響其他地方
- 上線前要手動測一輪，很花時間

是時候建立測試文化了。

## 測試金字塔

在研究前端測試時，我們學到「測試金字塔」的概念：

```
        /\
       /E2E\      <- 少量：整體流程測試
      /------\
     /  整合  \    <- 中量：元件互動測試
    /----------\
   /   單元測試  \  <- 大量：函式與邏輯測試
  /--------------\
```

**原則**：
- 單元測試數量最多、速度最快
- E2E 測試數量最少、執行最慢
- 不同層級的測試比例約為 70:20:10

## 技術選型

我們選擇的測試工具：

- **測試框架**：Jest（React 預設配置）
- **元件測試**：React Testing Library
- **E2E 測試**：（暫時不做，資源有限）

React Testing Library 的理念很棒：**測試使用者行為，而非實作細節**。

## 第一個測試：工具函式

從最簡單的開始：測試工具函式。

```javascript
// utils/validation.js
export function validateEmail(email) {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

export function validateIdNumber(idNumber) {
  const regex = /^[A-Z][12]\d{8}$/;
  return regex.test(idNumber);
}
```

測試檔案：

```javascript
// utils/validation.test.js
import { validateEmail, validateIdNumber } from './validation';

describe('validateEmail', () => {
  test('有效的 Email 應該通過驗證', () => {
    expect(validateEmail('test@example.com')).toBe(true);
    expect(validateEmail('user.name@domain.co.tw')).toBe(true);
  });

  test('無效的 Email 應該不通過驗證', () => {
    expect(validateEmail('invalid')).toBe(false);
    expect(validateEmail('test@')).toBe(false);
    expect(validateEmail('@example.com')).toBe(false);
  });
});

describe('validateIdNumber', () => {
  test('有效的身分證字號應該通過驗證', () => {
    expect(validateIdNumber('A123456789')).toBe(true);
    expect(validateIdNumber('Z212345678')).toBe(true);
  });

  test('無效的身分證字號應該不通過驗證', () => {
    expect(validateIdNumber('A12345678')).toBe(false);  // 少一碼
    expect(validateIdNumber('a123456789')).toBe(false); // 小寫
    expect(validateIdNumber('A323456789')).toBe(false); // 第二碼錯誤
  });
});
```

執行測試：

```bash
npm test
```

**心得**：工具函式最容易測試，也是建立信心的好起點。

## 測試自訂 Hook

測試 `useDataTable` Hook：

```javascript
// hooks/useDataTable.test.js
import { renderHook, waitFor } from '@testing-library/react';
import { useDataTable } from './useDataTable';
import axios from 'axios';

// Mock axios
jest.mock('axios');

describe('useDataTable', () => {
  const mockData = {
    data: {
      items: [{ id: 1, name: 'Test' }],
      total: 1
    }
  };

  beforeEach(() => {
    axios.get.mockResolvedValue(mockData);
  });

  test('初次載入應該呼叫 API', async () => {
    const { result } = renderHook(() => useDataTable('/api/users'));

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(axios.get).toHaveBeenCalledWith('/api/users', expect.any(Object));
    expect(result.current.data).toEqual(mockData.data.items);
  });

  test('改變分頁應該重新載入資料', async () => {
    const { result } = renderHook(() => useDataTable('/api/users'));

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    // 改變分頁
    result.current.handleTableChange({ current: 2, pageSize: 10 });

    await waitFor(() => {
      expect(axios.get).toHaveBeenCalledTimes(2);
    });
  });
});
```

## 測試元件

測試 `SearchForm` 元件：

```javascript
// components/SearchForm.test.jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import SearchForm from './SearchForm';

describe('SearchForm', () => {
  const mockColumns = [
    { key: 'name', label: '姓名', searchable: true },
    { key: 'email', label: 'Email', searchable: true }
  ];

  const mockOnSearch = jest.fn();

  test('應該渲染所有可搜尋欄位', () => {
    render(<SearchForm columns={mockColumns} onSearch={mockOnSearch} />);

    expect(screen.getByLabelText('姓名')).toBeInTheDocument();
    expect(screen.getByLabelText('Email')).toBeInTheDocument();
  });

  test('點擊查詢按鈕應該呼叫 onSearch', async () => {
    render(<SearchForm columns={mockColumns} onSearch={mockOnSearch} />);

    const nameInput = screen.getByLabelText('姓名');
    const searchButton = screen.getByText('查詢');

    fireEvent.change(nameInput, { target: { value: '測試' } });
    fireEvent.click(searchButton);

    await waitFor(() => {
      expect(mockOnSearch).toHaveBeenCalledWith({ name: '測試' });
    });
  });

  test('重置按鈕應該清空表單', async () => {
    render(<SearchForm columns={mockColumns} onSearch={mockOnSearch} />);

    const nameInput = screen.getByLabelText('姓名');
    const resetButton = screen.getByText('重置');

    fireEvent.change(nameInput, { target: { value: '測試' } });
    fireEvent.click(resetButton);

    await waitFor(() => {
      expect(nameInput.value).toBe('');
    });
  });
});
```

**關鍵點**：
- 使用 `screen.getByLabelText` 而非 `querySelector`（更接近使用者行為）
- 測試互動結果，而非內部狀態
- Mock 外部依賴（API、props）

## 測試覆蓋率

查看測試覆蓋率：

```bash
npm test -- --coverage
```

我們的目標：
- **工具函式**：100%
- **自訂 Hook**：80% 以上
- **元件**：60% 以上（UI 元件較難達到高覆蓋率）

不追求 100% 覆蓋率，重點是**測試關鍵邏輯**。

## 團隊實踐

### 1. 測試先行？後補？

理想上應該 TDD（Test-Driven Development），但實務上：
- 新功能：邊寫邊測（時間允許的話）
- 重構：一定要先補測試
- Bug 修復：先寫測試重現問題，再修 bug

### 2. Pull Request 要求

我們在 Code Review 時會檢查：
- 工具函式必須有測試
- 關鍵 Hook 必須有測試
- 元件測試視複雜度決定

不強制 100%，但要有基本的測試。

### 3. CI/CD 整合

在 GitHub Actions 中加入測試：

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
      - run: npm ci
      - run: npm test
```

Pull Request 必須通過測試才能合併。

## 測試的投資報酬率

一個月後，我們感受到測試帶來的好處：
- **重構更有信心**：改完跑測試就知道有沒有壞
- **減少回歸問題**：改 A 不會壞 B
- **文件化**：測試檔案本身就是使用範例

雖然寫測試要花時間，但長期來看絕對划算。

## 小結

前端測試不是奢侈品，而是必需品。

重點是：
- 從簡單的開始（工具函式）
- 測試使用者行為，不是實作細節
- 不追求 100% 覆蓋率，專注在關鍵邏輯

下週會分享 React 專案的部署流程與 CI/CD 設定。
