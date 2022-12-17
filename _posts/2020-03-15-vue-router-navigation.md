---
layout: post
title: "Vue Router 路由管理"
date: 2020-03-15 16:40:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Vue Router, SPA]
---

電商專案需要多個頁面：首頁、商品列表、商品詳情、購物車、訂單等。傳統的多頁面應用需要伺服器端路由，但 Vue 是單頁應用（SPA），需要使用 Vue Router 來管理前端路由。

> 本文使用 **Vue Router 3.x** (對應 Vue 2)

## 為什麼需要前端路由

後端開發習慣的路由模式：

```
GET  /products       -> 商品列表頁面
GET  /products/:id   -> 商品詳情頁面
GET  /cart           -> 購物車頁面
POST /orders         -> 建立訂單
```

在 SPA 中，所有頁面都在同一個 HTML 檔案中，透過 JavaScript 動態切換元件。Vue Router 讓我們可以用類似的 URL 結構來管理前端頁面。

## 安裝與設定

```bash
# 安裝 Vue Router
npm install vue-router@3
```

建立路由設定檔 `src/router/index.js`：

```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import Home from '@/views/Home.vue'
import ProductList from '@/views/ProductList.vue'
import ProductDetail from '@/views/ProductDetail.vue'
import Cart from '@/views/Cart.vue'
import Login from '@/views/Login.vue'

Vue.use(VueRouter)

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home
  },
  {
    path: '/products',
    name: 'ProductList',
    component: ProductList
  },
  {
    path: '/products/:id',
    name: 'ProductDetail',
    component: ProductDetail
  },
  {
    path: '/cart',
    name: 'Cart',
    component: Cart
  },
  {
    path: '/login',
    name: 'Login',
    component: Login
  }
]

const router = new VueRouter({
  mode: 'history',  // 使用 HTML5 History API，URL 不會有 #
  base: process.env.BASE_URL,
  routes
})

export default router
```

在 `main.js` 中掛載路由：

```javascript
import Vue from 'vue'
import App from './App.vue'
import router from './router'

new Vue({
  router,  // 注入路由
  render: h => h(App)
}).$mount('#app')
```

## 路由視圖與導航

### router-view

在 `App.vue` 中放置路由視圖：

```vue
<template>
  <div id="app">
    <nav>
      <router-link to="/">首頁</router-link>
      <router-link to="/products">商品列表</router-link>
      <router-link to="/cart">購物車</router-link>
      <router-link to="/login">登入</router-link>
    </nav>
    
    <!-- 路由匹配到的元件會顯示在這裡 -->
    <router-view></router-view>
  </div>
</template>
```

### router-link

`router-link` 會渲染成 `<a>` 標籤，點擊時不會重新載入頁面：

```vue
<!-- 基本用法 -->
<router-link to="/products">商品列表</router-link>

<!-- 使用 name 導航 -->
<router-link :to="{ name: 'ProductList' }">商品列表</router-link>

<!-- 帶參數 -->
<router-link :to="{ name: 'ProductDetail', params: { id: 123 } }">
  查看商品
</router-link>

<!-- 帶查詢參數 -->
<router-link :to="{ path: '/products', query: { category: '3C' } }">
  3C 商品
</router-link>

<!-- 自訂 class -->
<router-link to="/cart" active-class="active">
  購物車
</router-link>
```

## 動態路由

商品詳情頁需要根據商品 ID 顯示不同內容：

```javascript
// router/index.js
{
  path: '/products/:id',
  name: 'ProductDetail',
  component: ProductDetail
}
```

在元件中取得路由參數：

```vue
<template>
  <div class="product-detail">
    <h2>商品 ID: {{ productId }}</h2>
    <div v-if="product">
      <h3>{{ product.name }}</h3>
      <p>價格：NT$ {{ product.price }}</p>
      <button @click="addToCart">加入購物車</button>
    </div>
    <div v-else>
      <p>載入中...</p>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ProductDetail',
  data() {
    return {
      product: null
    }
  },
  computed: {
    productId() {
      return this.$route.params.id
    }
  },
  created() {
    this.fetchProduct()
  },
  watch: {
    // 當路由參數變化時重新載入資料
    '$route.params.id'() {
      this.fetchProduct()
    }
  },
  methods: {
    async fetchProduct() {
      // 模擬 API 呼叫
      const products = [
        { id: 1, name: '筆記型電腦', price: 35000 },
        { id: 2, name: '無線滑鼠', price: 890 }
      ]
      this.product = products.find(p => p.id === parseInt(this.productId))
    },
    addToCart() {
      console.log('加入購物車：', this.product)
      // 導航到購物車頁面
      this.$router.push('/cart')
    }
  }
}
</script>
```

## 程式化導航

除了使用 `router-link`，也可以在 JavaScript 中進行導航：

```javascript
// 導航到指定路徑
this.$router.push('/products')

// 使用 name
this.$router.push({ name: 'ProductList' })

// 帶參數
this.$router.push({ name: 'ProductDetail', params: { id: 123 } })

// 帶查詢參數
this.$router.push({ path: '/products', query: { category: '3C', sort: 'price' } })

// 後退
this.$router.go(-1)
this.$router.back()

// 前進
this.$router.go(1)
this.$router.forward()

// 替換當前路由（不會在 history 中留下記錄）
this.$router.replace('/login')
```

## 路由守衛

需要在進入某些頁面前檢查登入狀態：

### 全域守衛

```javascript
// router/index.js
const router = new VueRouter({
  mode: 'history',
  routes
})

// 全域前置守衛
router.beforeEach((to, from, next) => {
  // 檢查是否需要登入
  const requiresAuth = to.matched.some(record => record.meta.requiresAuth)
  const isLoggedIn = localStorage.getItem('token')
  
  if (requiresAuth && !isLoggedIn) {
    // 需要登入但未登入，導向登入頁
    next('/login')
  } else {
    // 允許進入
    next()
  }
})

export default router
```

### 路由 meta 資訊

```javascript
const routes = [
  {
    path: '/cart',
    name: 'Cart',
    component: Cart,
    meta: {
      requiresAuth: true,  // 需要登入
      title: '購物車'
    }
  },
  {
    path: '/orders',
    name: 'OrderList',
    component: OrderList,
    meta: {
      requiresAuth: true,
      title: '我的訂單'
    }
  },
  {
    path: '/login',
    name: 'Login',
    component: Login,
    meta: {
      title: '登入'
    }
  }
]
```

### 元件內守衛

```javascript
export default {
  name: 'Cart',
  beforeRouteEnter(to, from, next) {
    // 進入路由前
    console.log('準備進入購物車頁面')
    next()
  },
  
  beforeRouteUpdate(to, from, next) {
    // 路由改變但元件被複用時
    next()
  },
  
  beforeRouteLeave(to, from, next) {
    // 離開路由前
    const answer = window.confirm('購物車內容尚未結帳，確定要離開嗎？')
    if (answer) {
      next()
    } else {
      next(false)  // 取消導航
    }
  }
}
```

## 實戰範例：完整路由設定

```javascript
// router/index.js
import Vue from 'vue'
import VueRouter from 'vue-router'

Vue.use(VueRouter)

const routes = [
  {
    path: '/',
    name: 'Home',
    component: () => import('@/views/Home.vue'),
    meta: { title: '首頁' }
  },
  {
    path: '/products',
    name: 'ProductList',
    component: () => import('@/views/ProductList.vue'),
    meta: { title: '商品列表' }
  },
  {
    path: '/products/:id',
    name: 'ProductDetail',
    component: () => import('@/views/ProductDetail.vue'),
    meta: { title: '商品詳情' }
  },
  {
    path: '/cart',
    name: 'Cart',
    component: () => import('@/views/Cart.vue'),
    meta: { 
      title: '購物車',
      requiresAuth: true
    }
  },
  {
    path: '/orders',
    name: 'OrderList',
    component: () => import('@/views/OrderList.vue'),
    meta: { 
      title: '我的訂單',
      requiresAuth: true
    }
  },
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/views/Login.vue'),
    meta: { title: '登入' }
  },
  {
    path: '/404',
    name: 'NotFound',
    component: () => import('@/views/NotFound.vue'),
    meta: { title: '頁面不存在' }
  },
  {
    path: '*',
    redirect: '/404'
  }
]

const router = new VueRouter({
  mode: 'history',
  base: process.env.BASE_URL,
  routes,
  scrollBehavior(to, from, savedPosition) {
    // 切換路由時回到頁面頂端
    if (savedPosition) {
      return savedPosition
    } else {
      return { x: 0, y: 0 }
    }
  }
})

// 全域前置守衛
router.beforeEach((to, from, next) => {
  // 設定頁面標題
  document.title = to.meta.title || '購物商城'
  
  // 檢查登入狀態
  const requiresAuth = to.matched.some(record => record.meta.requiresAuth)
  const token = localStorage.getItem('token')
  
  if (requiresAuth && !token) {
    next({
      path: '/login',
      query: { redirect: to.fullPath }  // 記錄原本要去的頁面
    })
  } else {
    next()
  }
})

export default router
```

登入成功後導回原本的頁面：

```javascript
// Login.vue
methods: {
  async handleLogin() {
    try {
      // 呼叫登入 API
      const token = await this.loginApi(this.username, this.password)
      localStorage.setItem('token', token)
      
      // 導回原本要去的頁面，或是首頁
      const redirect = this.$route.query.redirect || '/'
      this.$router.push(redirect)
    } catch (error) {
      alert('登入失敗')
    }
  }
}
```

## 小結

Vue Router 是 SPA 的核心功能：

1. **宣告式導航**：使用 `router-link` 元件
2. **程式化導航**：使用 `$router.push()` 等方法
3. **動態路由**：使用 `$route.params` 取得參數
4. **路由守衛**：控制頁面存取權限

類似後端的 Controller 路由，但所有切換都在前端完成，不需要重新載入頁面，使用者體驗更流暢。
