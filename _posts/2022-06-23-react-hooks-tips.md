---
layout: post
title: "React Hooks 實戰心得：從濫用到善用"
date: 2022-06-23 14:15:00 +0800
categories: [前端開發]
tags: [React, Hooks, 效能優化, 最佳實踐]
---

## 從 Vue 到 React 的思維轉換

剛從 Vue 轉到 React 時，最不習慣的就是 Hooks。Vue 的 Options API 有明確的 `data`、`methods`、`computed`，React 卻把所有邏輯都塞在函式裡。

一開始我們寫出很多問題：
- `useEffect` 無限迴圈
- 不必要的重新渲染
- 自訂 Hook 的依賴陣列搞不清楚

經過這段時間的實戰，總算摸索出一些心得。

## useEffect 的常見陷阱

### 陷阱 1：忘記加依賴

```javascript
// 錯誤：少了依賴，count 變化不會重新執行
useEffect(() => {
  fetchData(count);
}, []);

// 正確
useEffect(() => {
  fetchData(count);
}, [count]);
```

### 陷阱 2：依賴項是物件或陣列

```javascript
// 錯誤：每次渲染都會建立新的物件，導致無限迴圈
useEffect(() => {
  console.log(filters);
}, [filters]);  // filters 是物件

// 正確：只依賴會變化的屬性
useEffect(() => {
  console.log(filters);
}, [filters.name, filters.status]);

// 或使用 JSON.stringify（適合簡單物件）
useEffect(() => {
  console.log(filters);
}, [JSON.stringify(filters)]);
```

### 陷阱 3：在 useEffect 中呼叫 setState

```javascript
// 錯誤：可能導致無限迴圈
useEffect(() => {
  setCount(count + 1);
}, [count]);

// 正確：使用 functional update
useEffect(() => {
  setCount(prev => prev + 1);
}, []);  // 不依賴 count
```

## 自訂 Hook 的設計原則

### 原則 1：單一職責

```javascript
// 錯誤：一個 Hook 做太多事
function useEverything() {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(false);
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  // ... 太多邏輯
}

// 正確：拆分成多個 Hook
function useDataFetch(url) { /* ... */ }
function useAuth() { /* ... */ }
function useTheme() { /* ... */ }
```

### 原則 2：回傳值的結構要穩定

```javascript
// 錯誤：有時回傳物件，有時回傳 null
function useUser(id) {
  const [user, setUser] = useState(null);
  // ...
  return user;  // 可能是 null
}

// 正確：始終回傳相同結構
function useUser(id) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  return { user, loading, error };  // 始終是物件
}
```

## useMemo 與 useCallback 的使用時機

一開始我們到處使用 `useMemo` 和 `useCallback`，以為這樣能優化效能。後來發現反而讓程式碼變複雜，效能也沒提升多少。

### 何時該用 useMemo

```javascript
// 適合：計算成本高的操作
const expensiveValue = useMemo(() => {
  return data.reduce((sum, item) => sum + item.value, 0);
}, [data]);

// 不適合：簡單的計算
const fullName = useMemo(() => {
  return `${firstName} ${lastName}`;
}, [firstName, lastName]);  // 多此一舉
```

### 何時該用 useCallback

```javascript
// 適合：傳遞給子元件的函式
const MemoizedChild = React.memo(ChildComponent);

function Parent() {
  const handleClick = useCallback(() => {
    // ...
  }, []);
  
  return <MemoizedChild onClick={handleClick} />;
}

// 不適合：沒有傳給子元件的函式
function Component() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);  // 多餘的優化
  
  return <button onClick={handleClick}>Click</button>;
}
```

我們的經驗法則：**先不優化，遇到效能問題再用 DevTools 找出瓶頸。**

## 實用的自訂 Hook

這段時間我們寫了幾個常用的 Hook：

### useDebounce - 延遲輸入

```javascript
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}

// 使用：搜尋框延遲查詢
function SearchBox() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 500);

  useEffect(() => {
    if (debouncedSearchTerm) {
      fetchSearchResults(debouncedSearchTerm);
    }
  }, [debouncedSearchTerm]);

  return <input onChange={(e) => setSearchTerm(e.target.value)} />;
}
```

### useLocalStorage - 持久化狀態

```javascript
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue];
}
```

### usePrevious - 取得前一次的值

```javascript
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

// 使用：比較前後變化
function Component({ count }) {
  const prevCount = usePrevious(count);
  
  return (
    <div>
      <p>現在：{count}</p>
      <p>之前：{prevCount}</p>
      <p>變化：{count - prevCount}</p>
    </div>
  );
}
```

## 小結

React Hooks 的學習曲線確實比 Vue 的 Options API 陡峭，但掌握之後會發現它的彈性更高。

關鍵是要理解：
- Hooks 是用來「提取與重用邏輯」的工具
- 不是所有情況都需要優化
- 自訂 Hook 應該符合單一職責原則

下週會分享一些效能優化的實戰案例，以及如何用 React DevTools 找出瓶頸。
