---
layout: post
title: "Vue HTTP 請求與 Axios 整合"
date: 2020-04-05 10:45:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Axios, HTTP, API]
render_with_liquid: false
---

前幾週的範例都是使用假資料，實際專案需要與後端 API 互動。Vue 本身不包含 HTTP 函式庫，通常會使用 Axios 來處理 API 請求。

## 安裝 Axios

```bash
npm install axios
```

## 基本使用

### GET 請求

```javascript
import axios from 'axios'

export default {
  data() {
    return {
      products: [],
      loading: false,
      error: null
    }
  },
  async created() {
    await this.fetchProducts()
  },
  methods: {
    async fetchProducts() {
      this.loading = true
      this.error = null
      
      try {
        const response = await axios.get('/api/products')
        this.products = response.data
      } catch (error) {
        console.error('載入商品失敗', error)
        this.error = '載入商品失敗，請稍後再試'
      } finally {
        this.loading = false
      }
    }
  }
}
```

### POST 請求

```javascript
methods: {
  async createOrder(orderData) {
    try {
      const response = await axios.post('/api/orders', orderData)
      console.log('訂單建立成功', response.data)
      return response.data
    } catch (error) {
      console.error('建立訂單失敗', error)
      throw error
    }
  }
}
```

### PUT / PATCH 請求

```javascript
methods: {
  async updateProduct(productId, updates) {
    try {
      const response = await axios.put(`/api/products/${productId}`, updates)
      return response.data
    } catch (error) {
      console.error('更新商品失敗', error)
      throw error
    }
  }
}
```

### DELETE 請求

```javascript
methods: {
  async deleteProduct(productId) {
    try {
      await axios.delete(`/api/products/${productId}`)
      console.log('刪除成功')
    } catch (error) {
      console.error('刪除失敗', error)
      throw error
    }
  }
}
```

## 設定 Axios 實例

建立統一的 API 客戶端，設定共用的配置：

```javascript
// src/api/client.js
import axios from 'axios'

const apiClient = axios.create({
  baseURL: process.env.VUE_APP_API_BASE_URL || 'http://localhost:8080/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
})

// 請求攔截器
apiClient.interceptors.request.use(
  (config) => {
    // 自動加入 token
    const token = localStorage.getItem('token')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    
    console.log('發送請求：', config.method.toUpperCase(), config.url)
    return config
  },
  (error) => {
    return Promise.reject(error)
  }
)

// 回應攔截器
apiClient.interceptors.response.use(
  (response) => {
    console.log('收到回應：', response.status, response.config.url)
    return response
  },
  (error) => {
    if (error.response) {
      // 伺服器回應錯誤狀態碼
      const status = error.response.status
      
      if (status === 401) {
        // 未授權，清除 token 並導向登入頁
        localStorage.removeItem('token')
        window.location.href = '/login'
      } else if (status === 403) {
        alert('沒有權限執行此操作')
      } else if (status === 404) {
        alert('找不到資源')
      } else if (status >= 500) {
        alert('伺服器錯誤，請稍後再試')
      }
    } else if (error.request) {
      // 請求已發送但沒有收到回應
      alert('網路連線異常，請檢查網路狀態')
    } else {
      // 其他錯誤
      console.error('請求錯誤', error.message)
    }
    
    return Promise.reject(error)
  }
)

export default apiClient
```

## API 模組化

將 API 呼叫封裝成獨立的模組：

```javascript
// src/api/products.js
import apiClient from './client'

export default {
  // 取得商品列表
  getProducts(params = {}) {
    return apiClient.get('/products', { params })
  },
  
  // 取得單一商品
  getProduct(id) {
    return apiClient.get(`/products/${id}`)
  },
  
  // 搜尋商品
  searchProducts(keyword, filters = {}) {
    return apiClient.get('/products/search', {
      params: {
        q: keyword,
        ...filters
      }
    })
  },
  
  // 建立商品
  createProduct(product) {
    return apiClient.post('/products', product)
  },
  
  // 更新商品
  updateProduct(id, updates) {
    return apiClient.put(`/products/${id}`, updates)
  },
  
  // 刪除商品
  deleteProduct(id) {
    return apiClient.delete(`/products/${id}`)
  }
}
```

```javascript
// src/api/orders.js
import apiClient from './client'

export default {
  // 取得訂單列表
  getOrders() {
    return apiClient.get('/orders')
  },
  
  // 取得單一訂單
  getOrder(id) {
    return apiClient.get(`/orders/${id}`)
  },
  
  // 建立訂單
  createOrder(orderData) {
    return apiClient.post('/orders', orderData)
  },
  
  // 取消訂單
  cancelOrder(id) {
    return apiClient.post(`/orders/${id}/cancel`)
  }
}
```

```javascript
// src/api/auth.js
import apiClient from './client'

export default {
  // 登入
  login(credentials) {
    return apiClient.post('/auth/login', credentials)
  },
  
  // 登出
  logout() {
    return apiClient.post('/auth/logout')
  },
  
  // 取得當前使用者資訊
  getCurrentUser() {
    return apiClient.get('/auth/me')
  }
}
```

統一匯出：

```javascript
// src/api/index.js
import products from './products'
import orders from './orders'
import auth from './auth'

export default {
  products,
  orders,
  auth
}
```

## 在元件中使用

```vue
<template>
  <div class="product-list">
    <div v-if="loading">載入中...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else>
      <div v-for="product in products" :key="product.id" class="product-card">
        <h3>{{ product.name }}</h3>
        <p>NT$ {{ product.price }}</p>
        <button @click="addToCart(product)">加入購物車</button>
      </div>
    </div>
  </div>
</template>

<script>
import api from '@/api'

export default {
  name: 'ProductList',
  data() {
    return {
      products: [],
      loading: false,
      error: null
    }
  },
  async created() {
    await this.fetchProducts()
  },
  methods: {
    async fetchProducts() {
      this.loading = true
      this.error = null
      
      try {
        const response = await api.products.getProducts({
          page: 1,
          limit: 20,
          sort: 'price'
        })
        this.products = response.data
      } catch (error) {
        this.error = '載入商品失敗'
      } finally {
        this.loading = false
      }
    },
    
    async addToCart(product) {
      // 加入購物車邏輯
      console.log('加入購物車', product)
    }
  }
}
</script>
```

## 實戰範例：完整購物流程

### 商品詳情頁

```vue
<template>
  <div class="product-detail">
    <div v-if="loading">載入中...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else-if="product">
      <h2>{{ product.name }}</h2>
      <img :src="product.image" :alt="product.name">
      <p class="price">NT$ {{ product.price }}</p>
      <p>{{ product.description }}</p>
      <p>庫存：{{ product.stock }}</p>
      
      <div class="quantity-selector">
        <button @click="quantity--" :disabled="quantity <= 1">-</button>
        <span>{{ quantity }}</span>
        <button @click="quantity++" :disabled="quantity >= product.stock">+</button>
      </div>
      
      <button 
        @click="handleAddToCart" 
        :disabled="product.stock === 0 || adding"
      >
        {{ adding ? '加入中...' : '加入購物車' }}
      </button>
    </div>
  </div>
</template>

<script>
import api from '@/api'

export default {
  name: 'ProductDetail',
  data() {
    return {
      product: null,
      loading: false,
      error: null,
      quantity: 1,
      adding: false
    }
  },
  async created() {
    await this.fetchProduct()
  },
  methods: {
    async fetchProduct() {
      const productId = this.$route.params.id
      this.loading = true
      this.error = null
      
      try {
        const response = await api.products.getProduct(productId)
        this.product = response.data
      } catch (error) {
        this.error = '載入商品失敗'
      } finally {
        this.loading = false
      }
    },
    
    async handleAddToCart() {
      this.adding = true
      
      try {
        await api.cart.addItem({
          productId: this.product.id,
          quantity: this.quantity
        })
        
        alert('已加入購物車')
        this.quantity = 1
      } catch (error) {
        alert('加入購物車失敗')
      } finally {
        this.adding = false
      }
    }
  }
}
</script>
```

### 登入頁面

```vue
<template>
  <div class="login-page">
    <h2>登入</h2>
    
    <form @submit.prevent="handleLogin">
      <div class="form-group">
        <input 
          v-model="form.username" 
          placeholder="帳號"
          required
        >
      </div>
      
      <div class="form-group">
        <input 
          v-model="form.password" 
          type="password"
          placeholder="密碼"
          required
        >
      </div>
      
      <button type="submit" :disabled="loading">
        {{ loading ? '登入中...' : '登入' }}
      </button>
    </form>
    
    <p v-if="error" class="error">{{ error }}</p>
  </div>
</template>

<script>
import api from '@/api'

export default {
  name: 'Login',
  data() {
    return {
      form: {
        username: '',
        password: ''
      },
      loading: false,
      error: null
    }
  },
  methods: {
    async handleLogin() {
      this.loading = true
      this.error = null
      
      try {
        const response = await api.auth.login({
          username: this.form.username,
          password: this.form.password
        })
        
        const { token } = response.data
        localStorage.setItem('token', token)
        
        // 導向原本要去的頁面或首頁
        const redirect = this.$route.query.redirect || '/'
        this.$router.push(redirect)
      } catch (error) {
        if (error.response && error.response.status === 401) {
          this.error = '帳號或密碼錯誤'
        } else {
          this.error = '登入失敗，請稍後再試'
        }
      } finally {
        this.loading = false
      }
    }
  }
}
</script>
```

### 訂單列表

```vue
<template>
  <div class="order-list">
    <h2>我的訂單</h2>
    
    <div v-if="loading">載入中...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else-if="orders.length === 0">
      <p>尚無訂單</p>
    </div>
    <div v-else>
      <div v-for="order in orders" :key="order.id" class="order-card">
        <h3>訂單編號：{{ order.orderNumber }}</h3>
        <p>訂單日期：{{ formatDate(order.createdAt) }}</p>
        <p>訂單狀態：{{ getStatusText(order.status) }}</p>
        <p>總金額：NT$ {{ order.totalAmount }}</p>
        
        <div class="order-items">
          <div v-for="item in order.items" :key="item.id">
            {{ item.productName }} x {{ item.quantity }}
          </div>
        </div>
        
        <button @click="viewOrderDetail(order.id)">查看詳情</button>
        
        <button 
          v-if="order.status === 'pending'"
          @click="cancelOrder(order.id)"
        >
          取消訂單
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import api from '@/api'

export default {
  name: 'OrderList',
  data() {
    return {
      orders: [],
      loading: false,
      error: null
    }
  },
  async created() {
    await this.fetchOrders()
  },
  methods: {
    async fetchOrders() {
      this.loading = true
      this.error = null
      
      try {
        const response = await api.orders.getOrders()
        this.orders = response.data
      } catch (error) {
        this.error = '載入訂單失敗'
      } finally {
        this.loading = false
      }
    },
    
    async cancelOrder(orderId) {
      if (!confirm('確定要取消此訂單嗎？')) {
        return
      }
      
      try {
        await api.orders.cancelOrder(orderId)
        alert('訂單已取消')
        await this.fetchOrders()  // 重新載入訂單列表
      } catch (error) {
        alert('取消訂單失敗')
      }
    },
    
    viewOrderDetail(orderId) {
      this.$router.push(`/orders/${orderId}`)
    },
    
    formatDate(dateString) {
      const date = new Date(dateString)
      return date.toLocaleDateString('zh-TW')
    },
    
    getStatusText(status) {
      const statusMap = {
        pending: '待處理',
        processing: '處理中',
        shipped: '已出貨',
        delivered: '已送達',
        cancelled: '已取消'
      }
      return statusMap[status] || status
    }
  }
}
</script>
```

## 錯誤處理最佳實務

```javascript
// 統一的錯誤處理
export function handleApiError(error, context = '') {
  let message = '操作失敗'
  
  if (error.response) {
    const status = error.response.status
    
    switch (status) {
      case 400:
        message = error.response.data.message || '請求參數錯誤'
        break
      case 401:
        message = '請先登入'
        break
      case 403:
        message = '沒有權限執行此操作'
        break
      case 404:
        message = '找不到資源'
        break
      case 500:
        message = '伺服器錯誤'
        break
      default:
        message = error.response.data.message || '操作失敗'
    }
  } else if (error.request) {
    message = '網路連線異常'
  }
  
  if (context) {
    message = `${context}：${message}`
  }
  
  return message
}
```

使用方式：

```javascript
import { handleApiError } from '@/utils/errorHandler'

methods: {
  async fetchProducts() {
    try {
      const response = await api.products.getProducts()
      this.products = response.data
    } catch (error) {
      this.error = handleApiError(error, '載入商品')
    }
  }
}
```

## 小結

Axios 整合的要點：

1. **統一配置**：建立 Axios 實例，設定共用的 baseURL、timeout、headers
2. **攔截器**：自動加入 token、統一處理錯誤
3. **API 模組化**：將 API 呼叫封裝成獨立模組，方便管理和重用
4. **錯誤處理**：統一處理不同的 HTTP 狀態碼

類似後端的 Service 層，將 HTTP 請求邏輯與元件分離，讓程式碼更易維護。
