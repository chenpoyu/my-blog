---
layout: post
title: "Vue 專案總結：購物車實戰回顧"
date: 2020-05-07 14:15:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, 專案總結, 購物車]
---

經過兩個多月的學習，從完全不懂 Vue.js 到能夠獨立開發購物車專案，這段過程讓我對前端開發有了更深入的理解。這篇文章總結這次學習的收穫與心得。

## 專案概述

專案需求是開發一個電商購物車系統，包含以下功能：

1. **商品瀏覽**：商品列表、商品詳情、搜尋篩選
2. **使用者認證**：登入、登出
3. **購物車**：加入購物車、修改數量、移除商品
4. **訂單**：建立訂單、查看訂單列表

技術棧：Vue 2.6 + Vue Router 3.x + Vuex 3.x + Axios

## 專案架構

```
src/
├── api/                    # API 呼叫模組
│   ├── client.js          # Axios 實例
│   ├── products.js        # 商品 API
│   ├── orders.js          # 訂單 API
│   ├── auth.js            # 認證 API
│   └── index.js           # 統一匯出
├── assets/                 # 靜態資源
├── components/             # 可重用元件
│   ├── BaseCard.vue
│   ├── Modal.vue
│   ├── Navbar.vue
│   └── ProductCard.vue
├── directives/             # 自訂指令
│   ├── permission.js
│   ├── clickOutside.js
│   └── lazyload.js
├── mixins/                 # 混入
│   ├── formatting.js
│   ├── auth.js
│   └── loading.js
├── router/                 # 路由設定
│   └── index.js
├── store/                  # Vuex store
│   ├── modules/
│   │   ├── cart.js
│   │   └── auth.js
│   ├── plugins/
│   │   └── cartPersistence.js
│   └── index.js
├── utils/                  # 工具函數
│   ├── validators.js
│   └── errorHandler.js
├── views/                  # 頁面元件
│   ├── Home.vue
│   ├── ProductList.vue
│   ├── ProductDetail.vue
│   ├── Cart.vue
│   ├── Checkout.vue
│   ├── OrderList.vue
│   ├── OrderDetail.vue
│   └── Login.vue
├── App.vue
└── main.js
```

## 核心功能實作

### 1. 狀態管理

使用 Vuex 管理全域狀態，將購物車和認證資訊集中管理：

```javascript
// store/modules/cart.js
export default {
  namespaced: true,
  state: {
    items: []
  },
  getters: {
    itemCount: state => state.items.length,
    totalAmount: state => {
      return state.items.reduce((sum, item) => {
        return sum + (item.price * item.quantity)
      }, 0)
    }
  },
  mutations: {
    ADD_ITEM(state, product) {
      const existing = state.items.find(item => item.id === product.id)
      if (existing) {
        existing.quantity++
      } else {
        state.items.push({ ...product, quantity: 1 })
      }
    }
    // ... 其他 mutations
  },
  actions: {
    async addItem({ commit, rootState }, product) {
      if (rootState.auth.user) {
        await api.cart.addItem({ productId: product.id, quantity: 1 })
      }
      commit('ADD_ITEM', product)
    }
    // ... 其他 actions
  }
}
```

### 2. 路由管理

使用 Vue Router 管理頁面切換，並實作路由守衛檢查登入狀態：

```javascript
// router/index.js
router.beforeEach((to, from, next) => {
  const requiresAuth = to.matched.some(record => record.meta.requiresAuth)
  const token = localStorage.getItem('token')
  
  if (requiresAuth && !token) {
    next({ path: '/login', query: { redirect: to.fullPath } })
  } else {
    next()
  }
})
```

### 3. API 整合

統一管理 API 呼叫，使用攔截器處理認證和錯誤：

```javascript
// api/client.js
apiClient.interceptors.request.use(config => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

apiClient.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)
```

### 4. 元件設計

建立可重用的基礎元件，透過 props 和 slots 提供彈性：

```vue
<!-- components/BaseCard.vue -->
<template>
  <div class="base-card">
    <div v-if="$slots.header" class="card-header">
      <slot name="header"></slot>
    </div>
    <div class="card-body">
      <slot></slot>
    </div>
    <div v-if="$slots.footer" class="card-footer">
      <slot name="footer"></slot>
    </div>
  </div>
</template>
```

## 從後端到前端的思維轉換

### 1. 資料流向

後端思維：
```
Request -> Controller -> Service -> Repository -> Database
```

前端思維：
```
User Action -> Component -> Action -> Mutation -> State -> View Update
```

### 2. 狀態管理

後端：資料存在資料庫，每次請求都從資料庫讀取

前端：資料存在記憶體（Vuex Store），頁面切換不會遺失，需要主動同步到後端

### 3. 非同步處理

後端：大多數操作都是同步的，等待資料庫回應

前端：大量非同步操作，需要處理載入狀態、錯誤處理、使用者體驗

```javascript
async fetchProducts() {
  this.loading = true
  this.error = null
  
  try {
    const response = await api.products.getProducts()
    this.products = response.data
  } catch (error) {
    this.error = '載入失敗'
  } finally {
    this.loading = false
  }
}
```

## 學到的最佳實務

### 1. 元件拆分原則

- 單一職責：一個元件只做一件事
- 可重用性：抽取共用元件（BaseCard、Modal）
- 大小適中：不要太大也不要太小

### 2. 命名規範

- 元件：PascalCase（ProductList.vue）
- 方法：camelCase（handleSubmit）
- 常量：UPPER_SNAKE_CASE（API_BASE_URL）
- CSS 類別：kebab-case（product-card）

### 3. 程式碼組織

- 相關邏輯放在一起
- 按功能模組化（api、store、utils）
- 避免深層巢狀

### 4. 效能優化

- 使用 computed 快取計算結果
- v-if vs v-show：頻繁切換用 v-show，條件很少改變用 v-if
- 圖片懶載入：使用自訂指令延遲載入圖片
- 路由懶載入：使用動態 import

```javascript
const routes = [
  {
    path: '/products',
    component: () => import('@/views/ProductList.vue')
  }
]
```

## 遇到的問題與解決

### 1. 響應式資料更新問題

**問題**：直接修改陣列元素，畫面不更新

```javascript
// 錯誤
this.items[0].quantity = 5

// 正確
this.$set(this.items, 0, { ...this.items[0], quantity: 5 })
// 或使用 Vuex mutation
```

### 2. 非同步資料載入時機

**問題**：在 created 中取得的資料，在 mounted 中 DOM 還沒更新

**解決**：使用 `$nextTick`

```javascript
async created() {
  await this.fetchProducts()
  this.$nextTick(() => {
    // DOM 已更新
    this.initScrollEvent()
  })
}
```

### 3. 路由參數變化元件不重新載入

**問題**：從 `/products/1` 切換到 `/products/2`，元件被複用

**解決**：監聽 `$route`

```javascript
watch: {
  '$route.params.id'() {
    this.fetchProduct()
  }
}
```

### 4. Vuex 狀態持久化

**問題**：重新整理頁面，Vuex 狀態消失

**解決**：使用 localStorage 或 plugin

```javascript
// store/plugins/cartPersistence.js
export default function createCartPersistencePlugin() {
  return (store) => {
    const savedCart = localStorage.getItem('cart')
    if (savedCart) {
      store.commit('cart/SET_ITEMS', JSON.parse(savedCart))
    }
    
    store.subscribe((mutation, state) => {
      if (mutation.type.startsWith('cart/')) {
        localStorage.setItem('cart', JSON.stringify(state.cart.items))
      }
    })
  }
}
```

## 與後端開發的比較

### 相似之處

1. **模組化**：都需要將程式碼拆分成模組
2. **分層架構**：前端有 View/Store/API，後端有 Controller/Service/Repository
3. **狀態管理**：前端用 Vuex，後端用 Session/Cache
4. **依賴注入**：前端用 Vue 的 provide/inject，後端用 Spring 的 @Autowired

### 不同之處

1. **執行環境**：前端在瀏覽器，資源有限；後端在伺服器，資源豐富
2. **資料持久化**：前端資料是暫時的；後端資料存在資料庫
3. **使用者互動**：前端直接面對使用者，需要即時回饋；後端專注業務邏輯
4. **除錯方式**：前端用瀏覽器開發者工具；後端用 IDE 除錯器

## 心得總結

### 收穫

1. **響應式思維**：理解資料驅動 UI 的概念
2. **元件化開發**：學會拆分和組合元件
3. **狀態管理**：掌握 Vuex 管理複雜狀態
4. **前端工程化**：了解建置工具、模組化、打包

### 挑戰

1. **思維轉換**：從命令式到宣告式
2. **非同步處理**：Promise、async/await 的使用
3. **CSS 佈局**：前端的視覺呈現需要額外學習
4. **瀏覽器差異**：需要考慮不同瀏覽器的相容性

### 持續學習

1. **Vue 2 進階**：深入理解響應式原理、虛擬 DOM
2. **TypeScript**：型別安全，更好的開發體驗
3. **測試**：Jest、Vue Test Utils
4. **效能優化**：虛擬滾動、程式碼分割

## 最後

從後端工程師的角度學習 Vue.js，最大的感受是前後端的思維方式有相通之處，但也有很大的差異。前端需要更關注使用者體驗、視覺呈現和互動細節。

這次購物車專案讓我體會到全端開發的價值，能夠理解前後端的需求和限制，有助於設計更好的 API 和系統架構。

Vue.js 的學習曲線相對友善，文件完整，社群活躍，是後端工程師學習前端的好選擇。接下來會持續深入學習 Vue 2 進階特性和 TypeScript，提升前端開發能力。
