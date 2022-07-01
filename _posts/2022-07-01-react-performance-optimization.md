---
layout: post
title: "React 效能優化實戰：解決列表卡頓問題"
date: 2022-07-01 09:45:00 +0800
categories: [前端開發]
tags: [React, 效能優化, React.memo, Virtualization]
---

## 效能問題浮現

上週我們的「案件管理」頁面上線後，使用者反應列表捲動時會卡頓。檢查後發現：
- 列表有 500+ 筆資料
- 每筆資料有 10+ 個欄位
- 每次捲動都會觸發大量重新渲染

這是典型的「渲染太多 DOM 節點」問題。

## 步驟 1：找出效能瓶頸

使用 React DevTools 的 Profiler 檢查：

1. 開啟 Profiler 標籤
2. 點擊「Record」
3. 操作頁面（捲動、排序、篩選）
4. 停止錄製並檢視結果

發現問題：
- `DataTable` 元件每次捲動都會完整重新渲染
- 每個 `TableRow` 元件渲染時間約 5ms
- 500 筆資料 = 2500ms（卡頓的元兇）

## 步驟 2：使用 React.memo 減少重渲染

```javascript
// components/TableRow.jsx
import { memo } from 'react';

const TableRow = memo(function TableRow({ record, columns, onEdit, onDelete }) {
  return (
    <tr>
      {columns.map(col => (
        <td key={col.key}>{record[col.key]}</td>
      ))}
      <td>
        <Button onClick={() => onEdit(record)}>編輯</Button>
        <Button onClick={() => onDelete(record.id)}>刪除</Button>
      </td>
    </tr>
  );
}, (prevProps, nextProps) => {
  // 自訂比較邏輯：只有 record 變化才重新渲染
  return prevProps.record.id === nextProps.record.id &&
         prevProps.record.updatedAt === nextProps.record.updatedAt;
});

export default TableRow;
```

**效果**：捲動時不再重渲染未變化的列，效能提升約 30%。

但還是不夠流暢。

## 步驟 3：導入虛擬滾動 (Virtualization)

虛擬滾動的核心概念：**只渲染可視範圍內的項目**。

我們使用 `react-window` 套件：

```bash
npm install react-window
```

改寫 `DataTable` 元件：

```javascript
// components/VirtualizedTable.jsx
import { FixedSizeList as List } from 'react-window';

function VirtualizedTable({ columns, data, onEdit, onDelete }) {
  const Row = ({ index, style }) => {
    const record = data[index];
    
    return (
      <div style={style} className="table-row">
        {columns.map(col => (
          <div key={col.key} className="table-cell">
            {record[col.key]}
          </div>
        ))}
        <div className="table-cell">
          <Button onClick={() => onEdit(record)}>編輯</Button>
          <Button onClick={() => onDelete(record.id)}>刪除</Button>
        </div>
      </div>
    );
  };

  return (
    <List
      height={600}           // 容器高度
      itemCount={data.length}
      itemSize={50}          // 每列高度
      width="100%"
    >
      {Row}
    </List>
  );
}

export default VirtualizedTable;
```

**效果**：
- 無論資料有多少筆，只渲染約 15 筆（可視範圍）
- 捲動流暢度大幅提升
- 記憶體使用量顯著降低

## 步驟 4：優化篩選與排序

虛擬滾動解決了顯示問題，但篩選與排序仍然很慢，因為要處理所有資料。

### 使用 useMemo 快取計算結果

```javascript
function DataTable({ data, filters, sorter }) {
  // 快取篩選結果
  const filteredData = useMemo(() => {
    return data.filter(item => {
      return Object.entries(filters).every(([key, value]) => {
        if (!value) return true;
        return String(item[key]).includes(value);
      });
    });
  }, [data, filters]);

  // 快取排序結果
  const sortedData = useMemo(() => {
    if (!sorter.field) return filteredData;
    
    return [...filteredData].sort((a, b) => {
      const aValue = a[sorter.field];
      const bValue = b[sorter.field];
      
      if (sorter.order === 'asc') {
        return aValue > bValue ? 1 : -1;
      } else {
        return aValue < bValue ? 1 : -1;
      }
    });
  }, [filteredData, sorter]);

  return <VirtualizedTable data={sortedData} />;
}
```

**重點**：只有在 `filters` 或 `sorter` 變化時才重新計算，避免每次渲染都執行。

## 步驟 5：延遲篩選查詢

使用者在搜尋框輸入時，不需要每個字都立即查詢。

```javascript
function SearchBox({ onSearch }) {
  const [inputValue, setInputValue] = useState('');
  const debouncedValue = useDebounce(inputValue, 300);

  useEffect(() => {
    onSearch(debouncedValue);
  }, [debouncedValue, onSearch]);

  return (
    <input 
      value={inputValue}
      onChange={(e) => setInputValue(e.target.value)}
      placeholder="搜尋..."
    />
  );
}
```

**效果**：減少不必要的 API 呼叫與資料處理。

## 優化前後對比

| 項目 | 優化前 | 優化後 |
|-----|-------|-------|
| 初次渲染時間 | 2500ms | 80ms |
| 捲動 FPS | ~20 | ~60 |
| 記憶體使用 | 120MB | 45MB |
| 篩選回應時間 | 500ms | 50ms |

## 其他優化技巧

### 1. 避免在 render 中建立新物件/陣列

```javascript
// 錯誤：每次渲染都建立新陣列
<Component options={['A', 'B', 'C']} />

// 正確：提到元件外
const OPTIONS = ['A', 'B', 'C'];
<Component options={OPTIONS} />
```

### 2. 使用 key 正確地識別列表項

```javascript
// 錯誤：使用 index 當 key
{data.map((item, index) => <Row key={index} data={item} />)}

// 正確：使用唯一識別符
{data.map(item => <Row key={item.id} data={item} />)}
```

### 3. 拆分大元件

```javascript
// 錯誤：一個元件包含所有邏輯
function UserPage() {
  // 100+ 行程式碼
}

// 正確：拆分成多個小元件
function UserPage() {
  return (
    <>
      <UserHeader />
      <UserProfile />
      <UserActivity />
    </>
  );
}
```

## 小結

效能優化的關鍵是：
1. **先測量，再優化**：用 Profiler 找出真正的瓶頸
2. **不要過度優化**：簡單頁面不需要虛擬滾動
3. **優先使用簡單方案**：React.memo 通常就夠用了

虛擬滾動適合：
- 列表項目超過 100 筆
- 每個項目的渲染成本較高
- 使用者需要快速捲動瀏覽

這次優化讓我深刻體會到，效能問題要從「使用者體驗」出發，而非追求技術上的完美。

下週會分享一些實務上遇到的「複雜互動」場景處理方式。
