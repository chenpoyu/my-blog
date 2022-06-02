---
layout: post
title: "通用 CRUD 元件實作：列表、分頁與查詢"
date: 2022-06-02 11:45:00 +0800
categories: [前端開發]
tags: [React, CRUD, Ant Design, 元件設計]
---

## CrudPage 元件架構

上週介紹了 Config-Driven 的概念，這週來看實際實作。`CrudPage` 元件主要由三個部分組成：

1. **查詢區塊**：根據配置產生搜尋欄位
2. **列表區塊**：顯示資料表格，支援排序與分頁
3. **操作區塊**：新增、編輯、刪除按鈕

## 核心 Hook：useDataTable

我們將資料表格的邏輯抽成一個自訂 Hook：

```javascript
// hooks/useDataTable.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export function useDataTable(apiPath) {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(false);
  const [pagination, setPagination] = useState({
    current: 1,
    pageSize: 10,
    total: 0
  });
  const [filters, setFilters] = useState({});
  const [sorter, setSorter] = useState({});

  const fetchData = async () => {
    setLoading(true);
    try {
      const params = {
        page: pagination.current,
        size: pagination.pageSize,
        ...filters,
        ...sorter
      };
      
      const response = await axios.get(apiPath, { params });
      
      setData(response.data.items);
      setPagination(prev => ({
        ...prev,
        total: response.data.total
      }));
    } catch (error) {
      console.error('Failed to fetch data:', error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchData();
  }, [pagination.current, pagination.pageSize, filters, sorter]);

  const handleTableChange = (newPagination, newFilters, newSorter) => {
    setPagination(newPagination);
    // 處理排序
    if (newSorter.order) {
      setSorter({
        sortBy: newSorter.field,
        sortOrder: newSorter.order === 'ascend' ? 'asc' : 'desc'
      });
    } else {
      setSorter({});
    }
  };

  const handleSearch = (searchFilters) => {
    setFilters(searchFilters);
    setPagination(prev => ({ ...prev, current: 1 }));
  };

  return {
    data,
    loading,
    pagination,
    handleTableChange,
    handleSearch,
    refresh: fetchData
  };
}
```

## 查詢表單生成

根據配置動態產生搜尋欄位：

```javascript
// components/SearchForm.jsx
import { Form, Input, Select, Button, Row, Col } from 'antd';

function SearchForm({ columns, onSearch }) {
  const [form] = Form.useForm();
  
  const searchableColumns = columns.filter(col => col.searchable);

  const handleSubmit = (values) => {
    // 過濾掉空值
    const filters = Object.fromEntries(
      Object.entries(values).filter(([_, v]) => v != null && v !== '')
    );
    onSearch(filters);
  };

  return (
    <Form form={form} onFinish={handleSubmit}>
      <Row gutter={16}>
        {searchableColumns.map(column => (
          <Col span={8} key={column.key}>
            <Form.Item name={column.key} label={column.label}>
              {column.type === 'select' ? (
                <Select placeholder={`請選擇${column.label}`} allowClear>
                  {column.options.map(opt => (
                    <Select.Option key={opt} value={opt}>
                      {opt}
                    </Select.Option>
                  ))}
                </Select>
              ) : (
                <Input placeholder={`請輸入${column.label}`} />
              )}
            </Form.Item>
          </Col>
        ))}
        <Col span={8}>
          <Form.Item>
            <Button type="primary" htmlType="submit">
              查詢
            </Button>
            <Button style={{ marginLeft: 8 }} onClick={() => form.resetFields()}>
              重置
            </Button>
          </Form.Item>
        </Col>
      </Row>
    </Form>
  );
}

export default SearchForm;
```

## 資料表格元件

```javascript
// components/DataTable.jsx
import { Table, Button, Space, Popconfirm } from 'antd';

function DataTable({ 
  columns, 
  data, 
  loading, 
  pagination, 
  onChange,
  onEdit,
  onDelete 
}) {
  // 加入操作欄位
  const tableColumns = [
    ...columns.map(col => ({
      title: col.label,
      dataIndex: col.key,
      key: col.key,
      sorter: col.sortable
    })),
    {
      title: '操作',
      key: 'action',
      render: (_, record) => (
        <Space>
          <Button type="link" onClick={() => onEdit(record)}>
            編輯
          </Button>
          <Popconfirm
            title="確定要刪除嗎？"
            onConfirm={() => onDelete(record.id)}
          >
            <Button type="link" danger>
              刪除
            </Button>
          </Popconfirm>
        </Space>
      )
    }
  ];

  return (
    <Table
      columns={tableColumns}
      dataSource={data}
      loading={loading}
      pagination={pagination}
      onChange={onChange}
      rowKey="id"
    />
  );
}

export default DataTable;
```

## 組合成 CrudPage

```javascript
// components/CrudPage.jsx
import { Button } from 'antd';
import SearchForm from './SearchForm';
import DataTable from './DataTable';
import { useDataTable } from '@/hooks/useDataTable';

function CrudPage({ config }) {
  const {
    data,
    loading,
    pagination,
    handleTableChange,
    handleSearch,
    refresh
  } = useDataTable(config.apiPath);

  const handleEdit = (record) => {
    // 開啟編輯表單 Modal（下週實作）
  };

  const handleDelete = async (id) => {
    // 呼叫刪除 API
    await axios.delete(`${config.apiPath}/${id}`);
    refresh();
  };

  return (
    <div>
      <h2>{config.title}</h2>
      <SearchForm columns={config.columns} onSearch={handleSearch} />
      <Button type="primary" style={{ marginBottom: 16 }}>
        新增
      </Button>
      <DataTable
        columns={config.columns}
        data={data}
        loading={loading}
        pagination={pagination}
        onChange={handleTableChange}
        onEdit={handleEdit}
        onDelete={handleDelete}
      />
    </div>
  );
}

export default CrudPage;
```

## 實際效果

有了這套架構，新增一個 CRUD 頁面只需要：
1. 撰寫配置檔（5 分鐘）
2. 建立頁面元件並引入配置（1 分鐘）

相比過去手刻一個頁面要 2-3 小時，效率提升非常明顯。

## 下一步

下週會分享表單的實作，包括新增與編輯的 Modal、表單驗證，以及如何處理不同欄位類型。
