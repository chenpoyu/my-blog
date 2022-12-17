---
layout: post
title: "Vue 表單處理與驗證"
date: 2020-03-29 15:20:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Form, Validation]
---

電商專案需要處理登入表單、訂單表單等輸入功能。Vue 提供了 `v-model` 指令來處理表單資料綁定，但複雜的驗證邏輯還是需要額外處理。

## v-model 雙向綁定

### 基本用法

```vue
<template>
  <div>
    <!-- 文字輸入 -->
    <input v-model="username" type="text">
    <p>使用者名稱：{{ username }}</p>
    
    <!-- 多行文字 -->
    <textarea v-model="description"></textarea>
    
    <!-- 核取方塊 -->
    <input v-model="agreed" type="checkbox">
    <p>是否同意：{{ agreed }}</p>
    
    <!-- 單選按鈕 -->
    <input v-model="gender" type="radio" value="male"> 男
    <input v-model="gender" type="radio" value="female"> 女
    <p>性別：{{ gender }}</p>
    
    <!-- 下拉選單 -->
    <select v-model="city">
      <option value="">請選擇</option>
      <option value="taipei">台北</option>
      <option value="taichung">台中</option>
      <option value="kaohsiung">高雄</option>
    </select>
    <p>城市：{{ city }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      username: '',
      description: '',
      agreed: false,
      gender: '',
      city: ''
    }
  }
}
</script>
```

### 修飾符

```vue
<template>
  <div>
    <!-- .lazy：change 事件觸發時才更新，而非 input 事件 -->
    <input v-model.lazy="username">
    
    <!-- .number：自動轉換為數字 -->
    <input v-model.number="age" type="number">
    
    <!-- .trim：自動去除首尾空白 -->
    <input v-model.trim="email">
  </div>
</template>
```

## 登入表單實作

```vue
<template>
  <div class="login-form">
    <h2>登入</h2>
    
    <form @submit.prevent="handleSubmit">
      <div class="form-group">
        <label>帳號：</label>
        <input 
          v-model.trim="form.username" 
          type="text"
          placeholder="請輸入帳號"
          @blur="validateUsername"
        >
        <span v-if="errors.username" class="error">
          {{ errors.username }}
        </span>
      </div>
      
      <div class="form-group">
        <label>密碼：</label>
        <input 
          v-model="form.password" 
          type="password"
          placeholder="請輸入密碼"
          @blur="validatePassword"
        >
        <span v-if="errors.password" class="error">
          {{ errors.password }}
        </span>
      </div>
      
      <div class="form-group">
        <label>
          <input v-model="form.rememberMe" type="checkbox">
          記住我
        </label>
      </div>
      
      <button type="submit" :disabled="!isFormValid">登入</button>
    </form>
  </div>
</template>

<script>
export default {
  name: 'LoginForm',
  data() {
    return {
      form: {
        username: '',
        password: '',
        rememberMe: false
      },
      errors: {
        username: '',
        password: ''
      }
    }
  },
  computed: {
    isFormValid() {
      return this.form.username.length > 0 && 
             this.form.password.length >= 6 &&
             !this.errors.username &&
             !this.errors.password
    }
  },
  methods: {
    validateUsername() {
      if (!this.form.username) {
        this.errors.username = '請輸入帳號'
      } else if (this.form.username.length < 4) {
        this.errors.username = '帳號至少 4 個字元'
      } else if (!/^[a-zA-Z0-9_]+$/.test(this.form.username)) {
        this.errors.username = '帳號只能包含英數字和底線'
      } else {
        this.errors.username = ''
      }
    },
    
    validatePassword() {
      if (!this.form.password) {
        this.errors.password = '請輸入密碼'
      } else if (this.form.password.length < 6) {
        this.errors.password = '密碼至少 6 個字元'
      } else {
        this.errors.password = ''
      }
    },
    
    async handleSubmit() {
      // 再次驗證所有欄位
      this.validateUsername()
      this.validatePassword()
      
      if (!this.isFormValid) {
        return
      }
      
      try {
        // 呼叫登入 API
        const response = await fetch('/api/login', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            username: this.form.username,
            password: this.form.password
          })
        })
        
        if (response.ok) {
          const data = await response.json()
          localStorage.setItem('token', data.token)
          
          if (this.form.rememberMe) {
            localStorage.setItem('username', this.form.username)
          }
          
          this.$router.push('/')
        } else {
          alert('登入失敗，請檢查帳號密碼')
        }
      } catch (error) {
        console.error('登入錯誤', error)
        alert('登入發生錯誤')
      }
    }
  },
  mounted() {
    // 如果之前有勾選記住我，自動填入帳號
    const savedUsername = localStorage.getItem('username')
    if (savedUsername) {
      this.form.username = savedUsername
      this.form.rememberMe = true
    }
  }
}
</script>

<style scoped>
.login-form {
  max-width: 400px;
  margin: 50px auto;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.form-group input[type="text"],
.form-group input[type="password"] {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.error {
  color: red;
  font-size: 14px;
  display: block;
  margin-top: 5px;
}

button {
  width: 100%;
  padding: 10px;
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
</style>
```

## 訂單表單實作

```vue
<template>
  <div class="order-form">
    <h2>訂單資訊</h2>
    
    <form @submit.prevent="handleSubmit">
      <!-- 收件人資訊 -->
      <fieldset>
        <legend>收件人資訊</legend>
        
        <div class="form-group">
          <label>姓名 *</label>
          <input v-model.trim="form.name" @blur="validateField('name')">
          <span v-if="errors.name" class="error">{{ errors.name }}</span>
        </div>
        
        <div class="form-group">
          <label>電話 *</label>
          <input v-model.trim="form.phone" @blur="validateField('phone')">
          <span v-if="errors.phone" class="error">{{ errors.phone }}</span>
        </div>
        
        <div class="form-group">
          <label>Email *</label>
          <input v-model.trim="form.email" type="email" @blur="validateField('email')">
          <span v-if="errors.email" class="error">{{ errors.email }}</span>
        </div>
      </fieldset>
      
      <!-- 配送地址 -->
      <fieldset>
        <legend>配送地址</legend>
        
        <div class="form-group">
          <label>縣市 *</label>
          <select v-model="form.city" @change="validateField('city')">
            <option value="">請選擇</option>
            <option value="台北市">台北市</option>
            <option value="新北市">新北市</option>
            <option value="台中市">台中市</option>
            <option value="高雄市">高雄市</option>
          </select>
          <span v-if="errors.city" class="error">{{ errors.city }}</span>
        </div>
        
        <div class="form-group">
          <label>地址 *</label>
          <input v-model.trim="form.address" @blur="validateField('address')">
          <span v-if="errors.address" class="error">{{ errors.address }}</span>
        </div>
      </fieldset>
      
      <!-- 付款方式 -->
      <fieldset>
        <legend>付款方式</legend>
        
        <div class="form-group">
          <label>
            <input v-model="form.paymentMethod" type="radio" value="credit">
            信用卡
          </label>
          <label>
            <input v-model="form.paymentMethod" type="radio" value="atm">
            ATM 轉帳
          </label>
          <label>
            <input v-model="form.paymentMethod" type="radio" value="cod">
            貨到付款
          </label>
        </div>
      </fieldset>
      
      <!-- 備註 -->
      <div class="form-group">
        <label>備註</label>
        <textarea v-model.trim="form.notes" rows="3"></textarea>
      </div>
      
      <button type="submit" :disabled="!isFormValid">送出訂單</button>
    </form>
  </div>
</template>

<script>
export default {
  name: 'OrderForm',
  data() {
    return {
      form: {
        name: '',
        phone: '',
        email: '',
        city: '',
        address: '',
        paymentMethod: 'credit',
        notes: ''
      },
      errors: {}
    }
  },
  computed: {
    isFormValid() {
      return this.form.name &&
             this.form.phone &&
             this.form.email &&
             this.form.city &&
             this.form.address &&
             Object.keys(this.errors).every(key => !this.errors[key])
    }
  },
  methods: {
    validateField(fieldName) {
      switch (fieldName) {
        case 'name':
          if (!this.form.name) {
            this.errors.name = '請輸入姓名'
          } else if (this.form.name.length < 2) {
            this.errors.name = '姓名至少 2 個字'
          } else {
            this.errors.name = ''
          }
          break
          
        case 'phone':
          const phonePattern = /^09\d{8}$/
          if (!this.form.phone) {
            this.errors.phone = '請輸入電話'
          } else if (!phonePattern.test(this.form.phone)) {
            this.errors.phone = '電話格式不正確（09xxxxxxxx）'
          } else {
            this.errors.phone = ''
          }
          break
          
        case 'email':
          const emailPattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
          if (!this.form.email) {
            this.errors.email = '請輸入 Email'
          } else if (!emailPattern.test(this.form.email)) {
            this.errors.email = 'Email 格式不正確'
          } else {
            this.errors.email = ''
          }
          break
          
        case 'city':
          if (!this.form.city) {
            this.errors.city = '請選擇縣市'
          } else {
            this.errors.city = ''
          }
          break
          
        case 'address':
          if (!this.form.address) {
            this.errors.address = '請輸入地址'
          } else if (this.form.address.length < 5) {
            this.errors.address = '地址過短'
          } else {
            this.errors.address = ''
          }
          break
      }
      
      // 強制重新渲染
      this.$forceUpdate()
    },
    
    validateAll() {
      this.validateField('name')
      this.validateField('phone')
      this.validateField('email')
      this.validateField('city')
      this.validateField('address')
    },
    
    async handleSubmit() {
      this.validateAll()
      
      if (!this.isFormValid) {
        alert('請檢查表單內容')
        return
      }
      
      try {
        const response = await fetch('/api/orders', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${localStorage.getItem('token')}`
          },
          body: JSON.stringify(this.form)
        })
        
        if (response.ok) {
          const order = await response.json()
          alert('訂單建立成功')
          this.$router.push(`/orders/${order.id}`)
        } else {
          alert('訂單建立失敗')
        }
      } catch (error) {
        console.error('提交訂單錯誤', error)
        alert('提交訂單發生錯誤')
      }
    }
  }
}
</script>

<style scoped>
.order-form {
  max-width: 600px;
  margin: 20px auto;
  padding: 20px;
}

fieldset {
  border: 1px solid #ddd;
  padding: 15px;
  margin-bottom: 20px;
  border-radius: 4px;
}

legend {
  font-weight: bold;
  padding: 0 10px;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
}

.form-group input[type="text"],
.form-group input[type="email"],
.form-group select,
.form-group textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.error {
  color: red;
  font-size: 14px;
  display: block;
  margin-top: 5px;
}

button {
  width: 100%;
  padding: 12px;
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
</style>
```

## 自訂驗證工具

將驗證邏輯抽離成可重用的工具：

```javascript
// utils/validators.js
export const validators = {
  required(value) {
    return value ? '' : '此欄位為必填'
  },
  
  minLength(min) {
    return (value) => {
      return value.length >= min ? '' : `至少需要 ${min} 個字元`
    }
  },
  
  maxLength(max) {
    return (value) => {
      return value.length <= max ? '' : `不可超過 ${max} 個字元`
    }
  },
  
  email(value) {
    const pattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    return pattern.test(value) ? '' : 'Email 格式不正確'
  },
  
  phone(value) {
    const pattern = /^09\d{8}$/
    return pattern.test(value) ? '' : '電話格式不正確'
  },
  
  url(value) {
    try {
      new URL(value)
      return ''
    } catch {
      return 'URL 格式不正確'
    }
  },
  
  number(value) {
    return !isNaN(value) ? '' : '必須是數字'
  },
  
  integer(value) {
    return Number.isInteger(Number(value)) ? '' : '必須是整數'
  },
  
  min(minValue) {
    return (value) => {
      return Number(value) >= minValue ? '' : `不可小於 ${minValue}`
    }
  },
  
  max(maxValue) {
    return (value) => {
      return Number(value) <= maxValue ? '' : `不可大於 ${maxValue}`
    }
  }
}

// 組合多個驗證器
export function validate(value, validatorList) {
  for (const validator of validatorList) {
    const error = validator(value)
    if (error) {
      return error
    }
  }
  return ''
}
```

使用方式：

```javascript
import { validators, validate } from '@/utils/validators'

export default {
  methods: {
    validateEmail() {
      this.errors.email = validate(this.form.email, [
        validators.required,
        validators.email
      ])
    },
    
    validateAge() {
      this.errors.age = validate(this.form.age, [
        validators.required,
        validators.integer,
        validators.min(18),
        validators.max(100)
      ])
    }
  }
}
```

## 小結

表單處理的關鍵技巧：

1. **v-model**：雙向綁定，自動同步 UI 和資料
2. **驗證時機**：blur 事件驗證個別欄位，submit 驗證全部
3. **錯誤顯示**：即時顯示錯誤訊息，提升使用者體驗
4. **禁用按鈕**：表單無效時禁用送出按鈕

下週會整合這些表單到完整的購物流程中，包含登入、加入購物車、結帳等功能。
