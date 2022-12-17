---
layout: post
title: "Vuex 狀態管理"
date: 2020-04-12 10:30:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Vuex, State Management]
render_with_liquid: false
---

購物車功能需要在多個元件間共享資料：商品列表要能加入購物車、導航列要顯示購物車數量、購物車頁面要能修改數量。如果用 props 和 emit 來傳遞，會變得很複雜。Vuex 提供了集中式的狀態管理方案。

> 本文使用 **Vuex 3.x** (對應 Vue 2)

## 為什麼需要 Vuex

沒有 Vuex 的痛苦：

```
商品列表元件
  -> emit('add-to-cart') 
    -> App.vue 
      -> :cart-items="cartItems" 
        -> 導航列元件
        
商品列表元件
  -> emit('add-to-cart')
    -> App.vue
      -> :cart-items="cartItems"
        -> 購物車元件
          -> emit('update-quantity')
            -> App.vue
              -> 更新 cartItems
```

資料和事件需要層層傳遞，當元件結構複雜時會很難維護。

使用 Vuex 後：

```
商品列表元件 -> dispatch('cart/addItem') -> Vuex Store
導航列元件 -> 從 Store 取得 cartItemCount
購物車元件 -> 從 Store 取得 cartItems，dispatch('cart/updateQuantity')
```

所有元件都直接與 Store 互動，不需要層層傳遞。類似後端的 Spring 容器，集中管理共享的狀態。

## 安裝與設定

```bash
npm install vuex@3
```

建立 Store：

```javascript
// src/store/index.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
  state: {
    // 狀態
  },
  getters: {
    // 計算屬性
  },
  mutations: {
    // 同步修改狀態
  },
  actions: {
    // 非同步操作
  },
  modules: {
    // 模組
  }
})
```

在 `main.js` 中掛載：

```javascript
import Vue from 'vue'
import App from './App.vue'
import store from './store'

new Vue({
  store,
  render: h => h(App)
}).$mount('#app')
```

## Vuex 核心概念

### State

存放應用程式的狀態資料：

```javascript
state: {
  user: null,
  cartItems: [],
  products: []
}
```

在元件中存取：

```javascript
// 方式 1：直接存取
this.$store.state.cartItems

// 方式 2：使用 mapState
import { mapState } from 'vuex'

export default {
  computed: {
    ...mapState(['user', 'cartItems'])
    // 等同於：
    // user() { return this.$store.state.user }
    // cartItems() { return this.$store.state.cartItems }
  }
}
```

### Getters

類似 computed，用於計算衍生狀態：

```javascript
getters: {
  // 購物車商品數量
  cartItemCount(state) {
    return state.cartItems.length
  },
  
  // 購物車總金額
  cartTotalAmount(state) {
    return state.cartItems.reduce((total, item) => {
      return total + (item.price * item.quantity)
    }, 0)
  },
  
  // 是否已登入
  isLoggedIn(state) {
    return state.user !== null
  },
  
  // 取得特定商品
  getProductById: (state) => (id) => {
    return state.products.find(product => product.id === id)
  }
}
```

在元件中使用：

```javascript
// 方式 1：直接存取
this.$store.getters.cartItemCount

// 方式 2：使用 mapGetters
import { mapGetters } from 'vuex'

export default {
  computed: {
    ...mapGetters(['cartItemCount', 'cartTotalAmount', 'isLoggedIn'])
  }
}
```

### Mutations

同步修改狀態，類似 setter：

```javascript
mutations: {
  // 設定使用者
  SET_USER(state, user) {
    state.user = user
  },
  
  // 加入購物車
  ADD_TO_CART(state, product) {
    const existingItem = state.cartItems.find(item => item.id === product.id)
    
    if (existingItem) {
      existingItem.quantity++
    } else {
      state.cartItems.push({
        ...product,
        quantity: 1
      })
    }
  },
  
  // 更新商品數量
  UPDATE_CART_ITEM_QUANTITY(state, { productId, quantity }) {
    const item = state.cartItems.find(item => item.id === productId)
    if (item) {
      item.quantity = quantity
    }
  },
  
  // 從購物車移除
  REMOVE_FROM_CART(state, productId) {
    const index = state.cartItems.findIndex(item => item.id === productId)
    if (index > -1) {
      state.cartItems.splice(index, 1)
    }
  },
  
  // 清空購物車
  CLEAR_CART(state) {
    state.cartItems = []
  }
}
```

呼叫 mutation：

```javascript
// 方式 1：commit
this.$store.commit('ADD_TO_CART', product)

// 方式 2：物件風格
this.$store.commit({
  type: 'UPDATE_CART_ITEM_QUANTITY',
  productId: 1,
  quantity: 3
})

// 方式 3：使用 mapMutations
import { mapMutations } from 'vuex'

export default {
  methods: {
    ...mapMutations(['ADD_TO_CART', 'REMOVE_FROM_CART']),
    
    handleAddToCart(product) {
      this.ADD_TO_CART(product)
    }
  }
}
```

### Actions

處理非同步操作，然後 commit mutation：

```javascript
actions: {
  // 登入
  async login({ commit }, credentials) {
    try {
      const response = await api.auth.login(credentials)
      const { token, user } = response.data
      
      localStorage.setItem('token', token)
      commit('SET_USER', user)
      
      return user
    } catch (error) {
      throw error
    }
  },
  
  // 登出
  async logout({ commit }) {
    try {
      await api.auth.logout()
      localStorage.removeItem('token')
      commit('SET_USER', null)
      commit('CLEAR_CART')
    } catch (error) {
      throw error
    }
  },
  
  // 載入商品
  async fetchProducts({ commit }) {
    try {
      const response = await api.products.getProducts()
      commit('SET_PRODUCTS', response.data)
    } catch (error) {
      throw error
    }
  },
  
  // 加入購物車（可能需要呼叫 API）
  async addToCart({ commit, state }, product) {
    // 如果已登入，同步到伺服器
    if (state.user) {
      await api.cart.addItem({
        productId: product.id,
        quantity: 1
      })
    }
    
    commit('ADD_TO_CART', product)
  },
  
  // 建立訂單
  async createOrder({ commit, state }, orderData) {
    try {
      const response = await api.orders.createOrder({
        ...orderData,
        items: state.cartItems
      })
      
      // 訂單建立成功，清空購物車
      commit('CLEAR_CART')
      
      return response.data
    } catch (error) {
      throw error
    }
  }
}
```

呼叫 action：

```javascript
// 方式 1：dispatch
await this.$store.dispatch('login', { username, password })

// 方式 2：使用 mapActions
import { mapActions } from 'vuex'

export default {
  methods: {
    ...mapActions(['login', 'logout', 'addToCart']),
    
    async handleLogin() {
      try {
        await this.login({
          username: this.username,
          password: this.password
        })
        this.$router.push('/')
      } catch (error) {
        alert('登入失敗')
      }
    }
  }
}
```

## 模組化 Store

當應用程式變大時，將 Store 拆分成模組：

```javascript
// src/store/modules/cart.js
export default {
  namespaced: true,
  
  state: {
    items: []
  },
  
  getters: {
    itemCount(state) {
      return state.items.length
    },
    
    totalAmount(state) {
      return state.items.reduce((total, item) => {
        return total + (item.price * item.quantity)
      }, 0)
    }
  },
  
  mutations: {
    ADD_ITEM(state, product) {
      const existingItem = state.items.find(item => item.id === product.id)
      
      if (existingItem) {
        existingItem.quantity++
      } else {
        state.items.push({ ...product, quantity: 1 })
      }
    },
    
    UPDATE_QUANTITY(state, { productId, quantity }) {
      const item = state.items.find(item => item.id === productId)
      if (item) {
        item.quantity = quantity
      }
    },
    
    REMOVE_ITEM(state, productId) {
      const index = state.items.findIndex(item => item.id === productId)
      if (index > -1) {
        state.items.splice(index, 1)
      }
    },
    
    CLEAR(state) {
      state.items = []
    }
  },
  
  actions: {
    async addItem({ commit, rootState }, product) {
      // 如果已登入，同步到伺服器
      if (rootState.auth.user) {
        await api.cart.addItem({
          productId: product.id,
          quantity: 1
        })
      }
      
      commit('ADD_ITEM', product)
    },
    
    updateQuantity({ commit }, payload) {
      commit('UPDATE_QUANTITY', payload)
    },
    
    removeItem({ commit }, productId) {
      commit('REMOVE_ITEM', productId)
    }
  }
}
```

```javascript
// src/store/modules/auth.js
export default {
  namespaced: true,
  
  state: {
    user: null,
    token: localStorage.getItem('token')
  },
  
  getters: {
    isLoggedIn(state) {
      return state.user !== null
    }
  },
  
  mutations: {
    SET_USER(state, user) {
      state.user = user
    },
    
    SET_TOKEN(state, token) {
      state.token = token
    }
  },
  
  actions: {
    async login({ commit }, credentials) {
      const response = await api.auth.login(credentials)
      const { token, user } = response.data
      
      localStorage.setItem('token', token)
      commit('SET_TOKEN', token)
      commit('SET_USER', user)
      
      return user
    },
    
    async logout({ commit }) {
      await api.auth.logout()
      localStorage.removeItem('token')
      commit('SET_TOKEN', null)
      commit('SET_USER', null)
    },
    
    async fetchUser({ commit, state }) {
      if (!state.token) return
      
      try {
        const response = await api.auth.getCurrentUser()
        commit('SET_USER', response.data)
      } catch (error) {
        // token 無效，清除
        localStorage.removeItem('token')
        commit('SET_TOKEN', null)
      }
    }
  }
}
```

```javascript
// src/store/index.js
import Vue from 'vue'
import Vuex from 'vuex'
import cart from './modules/cart'
import auth from './modules/auth'

Vue.use(Vuex)

export default new Vuex.Store({
  modules: {
    cart,
    auth
  }
})
```

使用模組化的 Store：

```javascript
// 存取 state
this.$store.state.cart.items
this.$store.state.auth.user

// 呼叫 getter
this.$store.getters['cart/itemCount']
this.$store.getters['auth/isLoggedIn']

// commit mutation
this.$store.commit('cart/ADD_ITEM', product)

// dispatch action
this.$store.dispatch('cart/addItem', product)
this.$store.dispatch('auth/login', credentials)

// 使用 map helpers
import { mapState, mapGetters, mapActions } from 'vuex'

export default {
  computed: {
    ...mapState('cart', ['items']),
    ...mapState('auth', ['user']),
    ...mapGetters('cart', ['itemCount', 'totalAmount']),
    ...mapGetters('auth', ['isLoggedIn'])
  },
  methods: {
    ...mapActions('cart', ['addItem', 'removeItem']),
    ...mapActions('auth', ['login', 'logout'])
  }
}
```

## 實戰範例：完整購物車

### 導航列顯示購物車數量

```vue
<template>
  <nav>
    <router-link to="/">首頁</router-link>
    <router-link to="/products">商品</router-link>
    <router-link to="/cart">
      購物車 <span v-if="cartItemCount > 0">({{ cartItemCount }})</span>
    </router-link>
    <div v-if="isLoggedIn">
      <span>{{ user.name }}</span>
      <button @click="handleLogout">登出</button>
    </div>
    <router-link v-else to="/login">登入</router-link>
  </nav>
</template>

<script>
import { mapState, mapGetters, mapActions } from 'vuex'

export default {
  name: 'Navbar',
  computed: {
    ...mapState('auth', ['user']),
    ...mapGetters('cart', ['itemCount']),
    ...mapGetters('auth', ['isLoggedIn']),
    
    cartItemCount() {
      return this.itemCount
    }
  },
  methods: {
    ...mapActions('auth', ['logout']),
    
    async handleLogout() {
      await this.logout()
      this.$router.push('/login')
    }
  }
}
</script>
```

### 商品列表加入購物車

```vue
<template>
  <div class="product-list">
    <div v-for="product in products" :key="product.id" class="product-card">
      <h3>{{ product.name }}</h3>
      <p>NT$ {{ product.price }}</p>
      <button @click="handleAddToCart(product)">加入購物車</button>
    </div>
  </div>
</template>

<script>
import { mapActions } from 'vuex'

export default {
  name: 'ProductList',
  data() {
    return {
      products: []
    }
  },
  async created() {
    const response = await api.products.getProducts()
    this.products = response.data
  },
  methods: {
    ...mapActions('cart', ['addItem']),
    
    async handleAddToCart(product) {
      try {
        await this.addItem(product)
        alert('已加入購物車')
      } catch (error) {
        alert('加入失敗')
      }
    }
  }
}
</script>
```

### 購物車頁面

```vue
<template>
  <div class="cart-page">
    <h2>購物車</h2>
    
    <div v-if="items.length === 0">
      <p>購物車是空的</p>
      <router-link to="/products">去逛逛</router-link>
    </div>
    
    <div v-else>
      <div v-for="item in items" :key="item.id" class="cart-item">
        <h3>{{ item.name }}</h3>
        <p>單價：NT$ {{ item.price }}</p>
        
        <div class="quantity-control">
          <button @click="decreaseQuantity(item)">-</button>
          <span>{{ item.quantity }}</span>
          <button @click="increaseQuantity(item)">+</button>
        </div>
        
        <p>小計：NT$ {{ item.price * item.quantity }}</p>
        <button @click="handleRemove(item.id)">移除</button>
      </div>
      
      <div class="cart-summary">
        <h3>總金額：NT$ {{ totalAmount }}</h3>
        <button @click="handleCheckout">結帳</button>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState, mapGetters, mapActions } from 'vuex'

export default {
  name: 'CartPage',
  computed: {
    ...mapState('cart', ['items']),
    ...mapGetters('cart', ['totalAmount']),
    ...mapGetters('auth', ['isLoggedIn'])
  },
  methods: {
    ...mapActions('cart', ['updateQuantity', 'removeItem']),
    
    increaseQuantity(item) {
      this.updateQuantity({
        productId: item.id,
        quantity: item.quantity + 1
      })
    },
    
    decreaseQuantity(item) {
      if (item.quantity > 1) {
        this.updateQuantity({
          productId: item.id,
          quantity: item.quantity - 1
        })
      }
    },
    
    handleRemove(productId) {
      if (confirm('確定要移除此商品嗎？')) {
        this.removeItem(productId)
      }
    },
    
    handleCheckout() {
      if (!this.isLoggedIn) {
        this.$router.push('/login?redirect=/checkout')
        return
      }
      
      this.$router.push('/checkout')
    }
  }
}
</script>
```

## 持久化購物車

```javascript
// src/store/plugins/cartPersistence.js
export default function createCartPersistencePlugin() {
  return (store) => {
    // 初始化時從 localStorage 載入
    const savedCart = localStorage.getItem('cart')
    if (savedCart) {
      store.commit('cart/SET_ITEMS', JSON.parse(savedCart))
    }
    
    // 監聽 state 變化，同步到 localStorage
    store.subscribe((mutation, state) => {
      if (mutation.type.startsWith('cart/')) {
        localStorage.setItem('cart', JSON.stringify(state.cart.items))
      }
    })
  }
}
```

```javascript
// src/store/index.js
import createCartPersistencePlugin from './plugins/cartPersistence'

export default new Vuex.Store({
  modules: {
    cart,
    auth
  },
  plugins: [createCartPersistencePlugin()]
})
```

## 小結

Vuex 的核心概念：

1. **State**：集中存放狀態
2. **Getters**：計算衍生狀態
3. **Mutations**：同步修改狀態
4. **Actions**：處理非同步操作
5. **Modules**：模組化管理

使用時機：
- 多個元件需要共享的狀態（如購物車、使用者資訊）
- 需要在多個地方修改的狀態
- 元件之間的關係很複雜

類似後端的 Service 層 + Repository 層，集中管理應用程式的資料流向。
