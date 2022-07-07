---
layout: post
title: "複雜表單互動：欄位聯動與條件顯示"
date: 2022-07-07 16:00:00 +0800
categories: [前端開發]
tags: [React, Form, 表單驗證, 業務邏輯]
---

## 真實世界的表單沒那麼簡單

Config-Driven 的表單系統能處理標準 CRUD，但政府專案的表單常有複雜的業務規則：

- 選擇「個人」或「公司」會顯示不同欄位
- 縣市與鄉鎮市區的下拉選單聯動
- 某些欄位的驗證規則取決於其他欄位的值

這週遇到一個典型案例：**補助申請表單**。

## 需求分析

申請人類型會影響表單內容：

**個人申請**：
- 身分證字號（必填）
- 出生年月日（必填）
- 聯絡電話（必填）

**公司申請**：
- 統一編號（必填）
- 公司名稱（必填）
- 負責人姓名（必填）
- 公司電話（必填）

此外，如果年齡小於 18 歲，需要填寫「法定代理人」資訊。

## 實作方式 1：條件渲染

最直覺的做法：

```javascript
function ApplicationForm() {
  const [applicantType, setApplicantType] = useState('individual');
  const [birthDate, setBirthDate] = useState(null);

  const age = calculateAge(birthDate);
  const needGuardian = age < 18;

  return (
    <Form>
      <Form.Item label="申請人類型">
        <Radio.Group value={applicantType} onChange={(e) => setApplicantType(e.target.value)}>
          <Radio value="individual">個人</Radio>
          <Radio value="company">公司</Radio>
        </Radio.Group>
      </Form.Item>

      {applicantType === 'individual' && (
        <>
          <Form.Item label="身分證字號" name="idNumber" rules={[{ required: true }]}>
            <Input />
          </Form.Item>
          <Form.Item label="出生年月日" name="birthDate" rules={[{ required: true }]}>
            <DatePicker onChange={setBirthDate} />
          </Form.Item>
        </>
      )}

      {applicantType === 'company' && (
        <>
          <Form.Item label="統一編號" name="taxId" rules={[{ required: true }]}>
            <Input />
          </Form.Item>
          <Form.Item label="公司名稱" name="companyName" rules={[{ required: true }]}>
            <Input />
          </Form.Item>
        </>
      )}

      {needGuardian && (
        <Form.Item label="法定代理人" name="guardianName" rules={[{ required: true }]}>
          <Input />
        </Form.Item>
      )}
    </Form>
  );
}
```

這樣寫沒問題，但表單欄位一多，程式碼會變得冗長且難維護。

## 實作方式 2：設定檔 + 動態渲染

把欄位配置抽出來：

```javascript
const fieldConfigs = {
  individual: [
    { name: 'idNumber', label: '身分證字號', type: 'text', required: true },
    { name: 'birthDate', label: '出生年月日', type: 'date', required: true },
    { name: 'phone', label: '聯絡電話', type: 'text', required: true }
  ],
  company: [
    { name: 'taxId', label: '統一編號', type: 'text', required: true },
    { name: 'companyName', label: '公司名稱', type: 'text', required: true },
    { name: 'contactPerson', label: '負責人姓名', type: 'text', required: true },
    { name: 'companyPhone', label: '公司電話', type: 'text', required: true }
  ]
};

function DynamicForm() {
  const [applicantType, setApplicantType] = useState('individual');
  const [formValues, setFormValues] = useState({});

  const currentFields = fieldConfigs[applicantType];
  const age = calculateAge(formValues.birthDate);

  return (
    <Form>
      <Form.Item label="申請人類型">
        <Radio.Group value={applicantType} onChange={(e) => setApplicantType(e.target.value)}>
          <Radio value="individual">個人</Radio>
          <Radio value="company">公司</Radio>
        </Radio.Group>
      </Form.Item>

      {currentFields.map(field => (
        <Form.Item 
          key={field.name}
          label={field.label} 
          name={field.name}
          rules={[{ required: field.required }]}
        >
          {renderFormControl(field)}
        </Form.Item>
      ))}

      {age < 18 && (
        <Form.Item label="法定代理人" name="guardianName" rules={[{ required: true }]}>
          <Input />
        </Form.Item>
      )}
    </Form>
  );
}
```

更清晰，也更容易擴充。

## 欄位聯動：縣市與鄉鎮市區

台灣地址選擇是常見需求：

```javascript
function AddressSelector() {
  const [cities] = useState(['台北市', '新北市', '桃園市', '台中市']);
  const [districts, setDistricts] = useState([]);
  const [selectedCity, setSelectedCity] = useState('');

  const districtMap = {
    '台北市': ['中正區', '大同區', '中山區', '松山區'],
    '新北市': ['板橋區', '新莊區', '中和區', '永和區'],
    // ...
  };

  const handleCityChange = (city) => {
    setSelectedCity(city);
    setDistricts(districtMap[city] || []);
    // 清空鄉鎮市區選擇
    form.setFieldsValue({ district: undefined });
  };

  return (
    <>
      <Form.Item label="縣市" name="city" rules={[{ required: true }]}>
        <Select onChange={handleCityChange}>
          {cities.map(city => (
            <Select.Option key={city} value={city}>{city}</Select.Option>
          ))}
        </Select>
      </Form.Item>

      <Form.Item label="鄉鎮市區" name="district" rules={[{ required: true }]}>
        <Select disabled={!selectedCity}>
          {districts.map(district => (
            <Select.Option key={district} value={district}>{district}</Select.Option>
          ))}
        </Select>
      </Form.Item>
    </>
  );
}
```

**重點**：當縣市變化時，需要：
1. 更新鄉鎮市區的選項
2. 清空原本的選擇

## 動態驗證規則

有時驗證規則也會受其他欄位影響：

```javascript
function DynamicValidation() {
  const [form] = Form.useForm();
  const applicantType = Form.useWatch('applicantType', form);

  return (
    <Form form={form}>
      {/* 身分證字號只有個人申請才需要驗證 */}
      <Form.Item
        name="idNumber"
        label="身分證字號"
        rules={[
          { 
            required: applicantType === 'individual',
            message: '請輸入身分證字號'
          },
          {
            pattern: /^[A-Z][12]\d{8}$/,
            message: '身分證字號格式不正確'
          }
        ]}
      >
        <Input />
      </Form.Item>

      {/* 統一編號只有公司申請才需要驗證 */}
      <Form.Item
        name="taxId"
        label="統一編號"
        rules={[
          {
            required: applicantType === 'company',
            message: '請輸入統一編號'
          },
          {
            pattern: /^\d{8}$/,
            message: '統一編號必須是 8 碼數字'
          }
        ]}
      >
        <Input />
      </Form.Item>
    </Form>
  );
}
```

使用 `Form.useWatch` 監聽欄位變化，動態調整驗證規則。

## 跨欄位驗證

某些驗證需要比較多個欄位：

```javascript
<Form.Item
  name="confirmPassword"
  label="確認密碼"
  dependencies={['password']}  // 依賴 password 欄位
  rules={[
    { required: true, message: '請輸入確認密碼' },
    ({ getFieldValue }) => ({
      validator(_, value) {
        if (!value || getFieldValue('password') === value) {
          return Promise.resolve();
        }
        return Promise.reject(new Error('兩次輸入的密碼不一致'));
      }
    })
  ]}
>
  <Input.Password />
</Form.Item>
```

## 自訂 Hook：useFormWatch

把欄位監聽邏輯包成 Hook：

```javascript
function useFormWatch(form, fieldName, callback) {
  const fieldValue = Form.useWatch(fieldName, form);

  useEffect(() => {
    if (fieldValue !== undefined) {
      callback(fieldValue);
    }
  }, [fieldValue, callback]);

  return fieldValue;
}

// 使用
function MyForm() {
  const [form] = Form.useForm();

  useFormWatch(form, 'city', (city) => {
    // 當 city 變化時執行
    form.setFieldsValue({ district: undefined });
    loadDistricts(city);
  });

  return <Form form={form}>{/* ... */}</Form>;
}
```

## 小結

處理複雜表單的關鍵：
1. **配置驅動**：簡化重複邏輯
2. **狀態集中管理**：使用 Form 實例統一控制
3. **善用 Hook**：抽取可重用的監聽邏輯

下週會分享 React 轉換專案的整體回顧，以及這段時間的心得總結。
