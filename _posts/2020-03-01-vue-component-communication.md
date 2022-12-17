---
layout: post
title: "Vue 元件通訊：Props 與 Emit"
date: 2020-03-01 14:20:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Component, Props, Emit]
render_with_liquid: false
---

上週學習了 Vue 的基本概念，這週要解決一個實際問題：如何讓元件之間互相溝通。在開發購物車功能時，商品列表元件需要傳遞資料給購物車元件，購物車元件也要通知父元件更新總金額。

## 元件通訊的場景

電商專案中常見的通訊需求：

```
App.vue (父元件)
├── ProductList.vue (子元件) -> 需要傳遞商品資料給父元件
├── ShoppingCart.vue (子元件) -> 需要接收父元件傳來的購物車資料
└── OrderSummary.vue (子元件) -> 需要顯示總金額
```

類似後端的方法呼叫，但前端元件之間不能直接呼叫，需要透過特定機制。

## Props：父傳子

Props 用於父元件傳遞資料給子元件，類似後端的建構函數參數或方法參數。

### 父元件傳遞資料

```vue
<!-- App.vue -->
<template>
  <div id="app">
    <ProductItem 
      :product-name="'筆記型電腦'"
      :price="35000"
      :stock="5"
    />
  </div>
</template>
```

### 子元件接收資料

```vue
<!-- ProductItem.vue -->
<template>
  <div class="product-item">
    <h3>{{ productName }}</h3>
    <p>價格：NT$ {{ price }}</p>
    <p>庫存：{{ stock }}</p>
  </div>
</template>

<script>
export default {
  name: 'ProductItem',
  props: {
    productName: {
      type: String,
      required: true
    },
    price: {
      type: Number,
      required: true
    },
    stock: {
      type: Number,
      default: 0
    }
  }
}
</script>
```

### Props 驗證

Vue 提供型別驗證，類似後端的參數驗證：

```javascript
props: {
  // 基本型別檢查
  name: String,
  age: Number,
  
  // 多種可能的型別
  priceOrName: [String, Number],
  
  // 必填的字串
  productId: {
    type: String,
    required: true
  },
  
  // 有預設值的數字
  quantity: {
    type: Number,
    default: 1
  },
  
  // 物件預設值需要用函數回傳
  product: {
    type: Object,
    default: () => ({
      id: 0,
      name: '',
      price: 0
    })
  },
  
  // 自訂驗證函數
  stock: {
    type: Number,
    validator: (value) => {
      return value >= 0  // 庫存不能是負數
    }
  }
}
```

## Emit：子傳父

子元件無法直接修改父元件的資料，需要透過事件通知父元件。類似後端的 Event 或 Callback。

### 子元件觸發事件

```vue
<!-- ProductItem.vue -->
<template>
  <div class="product-item">
    <h3>{{ product.name }}</h3>
    <p>價格：NT$ {{ product.price }}</p>
    <button @click="handleAddToCart">加入購物車</button>
  </div>
</template>

<script>
export default {
  name: 'ProductItem',
  props: {
    product: {
      type: Object,
      required: true
    }
  },
  methods: {
    handleAddToCart() {
      // 觸發自訂事件，傳遞商品資料
      this.$emit('add-to-cart', this.product)
    }
  }
}
</script>
```

### 父元件監聽事件

```vue
<!-- App.vue -->
<template>
  <div id="app">
    <ProductItem 
      v-for="product in products"
      :key="product.id"
      :product="product"
      @add-to-cart="addToCart"
    />
    <p>購物車商品數：{{ cartItemCount }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      products: [
        { id: 1, name: '筆記型電腦', price: 35000 },
        { id: 2, name: '無線滑鼠', price: 890 }
      ],
      cartItems: []
    }
  },
  computed: {
    cartItemCount() {
      return this.cartItems.length
    }
  },
  methods: {
    addToCart(product) {
      // 接收子元件傳來的資料
      this.cartItems.push({
        ...product,
        quantity: 1
      })
      console.log('購物車內容：', this.cartItems)
    }
  }
}
</script>
```

## 實戰範例：購物車系統

建立一個完整的購物車通訊範例：

### App.vue (父元件)

```vue
<template>
  <div id="app">
    <h1>購物商城</h1>
    
    <!-- 商品列表 -->
    <ProductList 
      :products="products"
      @add-to-cart="handleAddToCart"
    />
    
    <!-- 購物車 -->
    <ShoppingCart 
      :cart-items="cartItems"
      @update-quantity="handleUpdateQuantity"
      @remove-item="handleRemoveItem"
    />
  </div>
</template>

<script>
import ProductList from './components/ProductList.vue'
import ShoppingCart from './components/ShoppingCart.vue'

export default {
  name: 'App',
  components: {
    ProductList,
    ShoppingCart
  },
  data() {
    return {
      products: [
        { id: 1, name: '筆記型電腦', price: 35000, stock: 5 },
        { id: 2, name: '無線滑鼠', price: 890, stock: 20 },
        { id: 3, name: '機械鍵盤', price: 2500, stock: 10 }
      ],
      cartItems: []
    }
  },
  methods: {
    handleAddToCart(product) {
      const existingItem = this.cartItems.find(item => item.id === product.id)
      
      if (existingItem) {
        existingItem.quantity++
      } else {
        this.cartItems.push({
          ...product,
          quantity: 1
        })
      }
    },
    
    handleUpdateQuantity(payload) {
      const item = this.cartItems.find(item => item.id === payload.id)
      if (item) {
        item.quantity = payload.quantity
      }
    },
    
    handleRemoveItem(productId) {
      const index = this.cartItems.findIndex(item => item.id === productId)
      if (index > -1) {
        this.cartItems.splice(index, 1)
      }
    }
  }
}
</script>
```

### ProductList.vue (子元件)

```vue
<template>
  <div class="product-list">
    <h2>商品列表</h2>
    <div v-for="product in products" :key="product.id" class="product-card">
      <h3>{{ product.name }}</h3>
      <p>價格：NT$ {{ product.price }}</p>
      <p>庫存：{{ product.stock }}</p>
      <button 
        @click="$emit('add-to-cart', product)"
        :disabled="product.stock === 0"
      >
        {{ product.stock > 0 ? '加入購物車' : '已售完' }}
      </button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ProductList',
  props: {
    products: {
      type: Array,
      required: true
    }
  }
}
</script>
```

### ShoppingCart.vue (子元件)

```vue
<template>
  <div class="shopping-cart">
    <h2>購物車</h2>
    <div v-if="cartItems.length === 0">
      <p>購物車是空的</p>
    </div>
    <div v-else>
      <div v-for="item in cartItems" :key="item.id" class="cart-item">
        <h4>{{ item.name }}</h4>
        <p>單價：NT$ {{ item.price }}</p>
        <div class="quantity-control">
          <button @click="decreaseQuantity(item)">-</button>
          <span>{{ item.quantity }}</span>
          <button @click="increaseQuantity(item)">+</button>
        </div>
        <p>小計：NT$ {{ item.price * item.quantity }}</p>
        <button @click="$emit('remove-item', item.id)">移除</button>
      </div>
      <div class="cart-summary">
        <h3>總金額：NT$ {{ totalAmount }}</h3>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ShoppingCart',
  props: {
    cartItems: {
      type: Array,
      required: true
    }
  },
  computed: {
    totalAmount() {
      return this.cartItems.reduce((sum, item) => {
        return sum + (item.price * item.quantity)
      }, 0)
    }
  },
  methods: {
    increaseQuantity(item) {
      this.$emit('update-quantity', {
        id: item.id,
        quantity: item.quantity + 1
      })
    },
    
    decreaseQuantity(item) {
      if (item.quantity > 1) {
        this.$emit('update-quantity', {
          id: item.id,
          quantity: item.quantity - 1
        })
      }
    }
  }
}
</script>
```

## 注意事項

### 1. Props 是單向的

子元件不能直接修改 props：

```javascript
// 錯誤：不要直接修改 props
this.productName = 'new name'  // Vue 會警告

// 正確：透過 emit 通知父元件修改
this.$emit('update-name', 'new name')
```

### 2. 事件命名規則

使用 kebab-case 命名事件：

```javascript
// 好的命名
this.$emit('add-to-cart')
this.$emit('update-quantity')

// 避免使用 camelCase
this.$emit('addToCart')  // 在 HTML 中不易辨識
```

### 3. 複雜資料要深拷貝

傳遞物件或陣列時要注意參考問題：

```javascript
methods: {
  handleAddToCart(product) {
    // 使用展開運算子複製，避免直接修改原始資料
    this.cartItems.push({ ...product, quantity: 1 })
  }
}
```

## 小結

元件通訊是 Vue 開發的核心技能：

1. **Props**：父傳子，類似參數傳遞
2. **Emit**：子傳父，類似事件回呼
3. **單向資料流**：資料由上往下傳，事件由下往上傳

這種設計模式讓元件之間的關係更清晰，類似後端的分層架構，每一層都有明確的職責。下週會學習如何使用 Vuex 管理更複雜的狀態。
