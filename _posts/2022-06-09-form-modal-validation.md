---
layout: post
title: "動態表單生成：Modal、驗證與資料提交"
date: 2022-06-09 15:30:00 +0800
categories: [前端開發]
tags: [React, Form, Validation, Ant Design]
---

## 表單 Modal 的設計

延續上週的內容，這週要實作新增與編輯功能。我們選擇使用 Modal（對話框）而非獨立頁面，原因是：

- 操作更直覺，不需要跳轉頁面
- 可以同時看到列表與表單
- 符合後台系統的使用習慣

## FormModal 元件實作

```javascript
// components/FormModal.jsx
import { Modal, Form, Input, Select, InputNumber } from 'antd';
import { useEffect } from 'react';

function FormModal({ 
  visible, 
  mode,           // 'create' | 'edit'
  formFields, 
  initialValues,
  onSubmit, 
  onCancel 
}) {
  const [form] = Form.useForm();

  useEffect(() => {
    if (visible) {
      if (mode === 'edit' && initialValues) {
        form.setFieldsValue(initialValues);
      } else {
        form.resetFields();
      }
    }
  }, [visible, mode, initialValues, form]);

  const handleOk = async () => {
    try {
      const values = await form.validateFields();
      await onSubmit(values);
      form.resetFields();
    } catch (error) {
      console.error('Validation failed:', error);
    }
  };

  return (
    <Modal
      title={mode === 'create' ? '新增' : '編輯'}
      open={visible}
      onOk={handleOk}
      onCancel={onCancel}
      width={600}
    >
      <Form form={form} layout="vertical">
        {formFields.map(field => (
          <Form.Item
            key={field.name}
            name={field.name}
            label={field.label}
            rules={[
              { required: field.required, message: `請輸入${field.label}` },
              ...buildValidationRules(field)
            ]}
          >
            {renderFormControl(field)}
          </Form.Item>
        ))}
      </Form>
    </Modal>
  );
}

export default FormModal;
```

## 動態生成表單控制項

根據欄位類型產生對應的輸入元件：

```javascript
function renderFormControl(field) {
  switch (field.type) {
    case 'text':
    case 'email':
      return <Input placeholder={`請輸入${field.label}`} />;
    
    case 'textarea':
      return <Input.TextArea rows={4} placeholder={`請輸入${field.label}`} />;
    
    case 'number':
      return <InputNumber style={{ width: '100%' }} />;
    
    case 'select':
      return (
        <Select placeholder={`請選擇${field.label}`}>
          {field.options.map(opt => (
            <Select.Option key={opt} value={opt}>
              {opt}
            </Select.Option>
          ))}
        </Select>
      );
    
    case 'date':
      return <DatePicker style={{ width: '100%' }} />;
    
    default:
      return <Input />;
  }
}
```

## 驗證規則生成

將配置檔中的驗證規則轉換成 Ant Design 的格式：

```javascript
function buildValidationRules(field) {
  const rules = [];
  
  if (field.rules) {
    // Email 驗證
    if (field.type === 'email') {
      rules.push({
        type: 'email',
        message: '請輸入有效的 Email'
      });
    }
    
    // 長度驗證
    if (field.rules.minLength) {
      rules.push({
        min: field.rules.minLength,
        message: `長度不可少於 ${field.rules.minLength} 個字元`
      });
    }
    
    if (field.rules.maxLength) {
      rules.push({
        max: field.rules.maxLength,
        message: `長度不可超過 ${field.rules.maxLength} 個字元`
      });
    }
    
    // 數字範圍驗證
    if (field.rules.min !== undefined) {
      rules.push({
        type: 'number',
        min: field.rules.min,
        message: `數值不可小於 ${field.rules.min}`
      });
    }
    
    if (field.rules.max !== undefined) {
      rules.push({
        type: 'number',
        max: field.rules.max,
        message: `數值不可大於 ${field.rules.max}`
      });
    }
    
    // 自訂驗證
    if (field.rules.pattern) {
      rules.push({
        pattern: new RegExp(field.rules.pattern),
        message: field.rules.message || '格式不正確'
      });
    }
  }
  
  return rules;
}
```

## 整合到 CrudPage

更新 `CrudPage` 元件，加入表單 Modal：

```javascript
// components/CrudPage.jsx (部分程式碼)
import { useState } from 'react';
import axios from 'axios';
import FormModal from './FormModal';

function CrudPage({ config }) {
  // ... 前面的 code
  
  const [modalVisible, setModalVisible] = useState(false);
  const [modalMode, setModalMode] = useState('create');
  const [editingRecord, setEditingRecord] = useState(null);

  const handleCreate = () => {
    setModalMode('create');
    setEditingRecord(null);
    setModalVisible(true);
  };

  const handleEdit = (record) => {
    setModalMode('edit');
    setEditingRecord(record);
    setModalVisible(true);
  };

  const handleSubmit = async (values) => {
    try {
      if (modalMode === 'create') {
        await axios.post(config.apiPath, values);
      } else {
        await axios.put(`${config.apiPath}/${editingRecord.id}`, values);
      }
      setModalVisible(false);
      refresh();
    } catch (error) {
      console.error('Submit failed:', error);
    }
  };

  return (
    <div>
      {/* ... 前面的 JSX */}
      
      <Button type="primary" onClick={handleCreate}>
        新增
      </Button>
      
      {/* ... DataTable */}
      
      <FormModal
        visible={modalVisible}
        mode={modalMode}
        formFields={config.formFields}
        initialValues={editingRecord}
        onSubmit={handleSubmit}
        onCancel={() => setModalVisible(false)}
      />
    </div>
  );
}
```

## 配置檔擴充範例

現在我們可以在配置檔中定義更詳細的驗證規則：

```javascript
formFields: [
  { 
    name: 'username', 
    label: '帳號', 
    type: 'text', 
    required: true,
    rules: { 
      minLength: 4,
      maxLength: 20,
      pattern: '^[a-zA-Z0-9_]+$',
      message: '只能包含英文、數字和底線'
    }
  },
  { 
    name: 'age', 
    label: '年齡', 
    type: 'number',
    rules: { min: 18, max: 100 }
  }
]
```

## 實戰心得

這套表單系統上線後，團隊反應相當好：

- **開發速度**：新增頁面從 2 小時降到 10 分鐘
- **程式碼一致性**：所有頁面的操作邏輯統一
- **維護容易**：修改通用邏輯只需改一處

唯一的缺點是學習曲線稍高，新成員需要時間理解配置檔的結構。

## 下一步

接下來會分享如何處理複雜頁面的客製化需求，以及這套系統在實際專案中遇到的挑戰。
