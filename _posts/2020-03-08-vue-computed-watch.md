---
layout: post
title: "Vue 計算屬性與監聽器"
date: 2020-03-08 09:15:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Computed, Watch]
render_with_liquid: false
---

在開發購物車時，發現需要頻繁計算總價、折扣後金額等衍生資料。如果每次都在 methods 中計算，程式碼會變得很冗長。Vue 提供了 computed 和 watch 來處理這類需求。

## Computed 計算屬性

計算屬性是基於其他資料計算出來的值，會自動快取，只有相依的資料改變時才會重新計算。

### 基本用法

```vue
<template>
  <div>
    <p>原價：NT$ {{ originalPrice }}</p>
    <p>折扣：{{ discount }}%</p>
    <p>售價：NT$ {{ finalPrice }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      originalPrice: 1000,
      discount: 20
    }
  },
  computed: {
    finalPrice() {
      return this.originalPrice * (1 - this.discount / 100)
    }
  }
}
</script>
```

### 購物車總價計算

```vue
<script>
export default {
  data() {
    return {
      cartItems: [
        { id: 1, name: '筆電', price: 35000, quantity: 1 },
        { id: 2, name: '滑鼠', price: 890, quantity: 2 }
      ],
      shippingFee: 100,
      discountCode: 'SAVE10'
    }
  },
  computed: {
    // 小計
    subtotal() {
      return this.cartItems.reduce((sum, item) => {
        return sum + (item.price * item.quantity)
      }, 0)
    },
    
    // 折扣金額
    discountAmount() {
      if (this.discountCode === 'SAVE10') {
        return this.subtotal * 0.1
      }
      return 0
    },
    
    // 總金額
    totalAmount() {
      return this.subtotal - this.discountAmount + this.shippingFee
    },
    
    // 免運門檻提示
    freeShippingHint() {
      const threshold = 1000
      if (this.subtotal >= threshold) {
        return '已符合免運資格'
      }
      const diff = threshold - this.subtotal
      return `再消費 NT$ ${diff} 即可免運`
    }
  }
}
</script>
```

### Computed vs Methods

```javascript
// Methods：每次呼叫都會執行
methods: {
  calculateTotal() {
    console.log('計算總價')  // 每次呼叫都會印出
    return this.cartItems.reduce((sum, item) => sum + item.price, 0)
  }
}

// Computed：有快取，相依資料不變就不會重新計算
computed: {
  total() {
    console.log('計算總價')  // 只有 cartItems 改變才會印出
    return this.cartItems.reduce((sum, item) => sum + item.price, 0)
  }
}
```

在模板中：

```vue
<template>
  <div>
    <!-- Methods 需要加括號，每次渲染都會執行 -->
    <p>{{ calculateTotal() }}</p>
    
    <!-- Computed 不需要括號，有快取機制 -->
    <p>{{ total }}</p>
  </div>
</template>
```

## Watch 監聽器

監聽資料變化，當資料改變時執行特定操作。適合用在需要執行非同步操作或複雜邏輯的場景。

### 基本用法

```javascript
export default {
  data() {
    return {
      searchKeyword: '',
      searchResults: []
    }
  },
  watch: {
    searchKeyword(newValue, oldValue) {
      console.log(`搜尋關鍵字從 ${oldValue} 變成 ${newValue}`)
      this.performSearch(newValue)
    }
  },
  methods: {
    performSearch(keyword) {
      // 執行搜尋 API 呼叫
      console.log('搜尋：', keyword)
    }
  }
}
```

### 深度監聽物件

```javascript
export default {
  data() {
    return {
      user: {
        name: 'John',
        address: {
          city: 'Taipei',
          district: 'Xinyi'
        }
      }
    }
  },
  watch: {
    // 淺層監聽：只監聽物件參考的改變
    user(newValue) {
      console.log('使用者物件被替換')
    },
    
    // 深度監聽：監聽物件內部屬性的改變
    user: {
      handler(newValue) {
        console.log('使用者資料有變動', newValue)
        // 儲存到 localStorage
        localStorage.setItem('user', JSON.stringify(newValue))
      },
      deep: true
    }
  }
}
```

### 監聽陣列變化

```javascript
export default {
  data() {
    return {
      cartItems: []
    }
  },
  watch: {
    cartItems: {
      handler(newItems) {
        console.log('購物車商品數量：', newItems.length)
        // 同步到後端
        this.syncCartToServer(newItems)
      },
      deep: true
    }
  },
  methods: {
    syncCartToServer(items) {
      // 呼叫 API 同步購物車資料
      console.log('同步購物車到伺服器')
    }
  }
}
```

### Immediate 選項

```javascript
watch: {
  // 元件載入時就執行一次
  userId: {
    handler(newId) {
      this.fetchUserData(newId)
    },
    immediate: true  // 立即執行
  }
}
```

## 實戰範例：商品搜尋與篩選

```vue
<template>
  <div class="product-search">
    <input 
      v-model="searchKeyword" 
      placeholder="搜尋商品"
    >
    
    <select v-model="selectedCategory">
      <option value="">全部分類</option>
      <option value="3C">3C 產品</option>
      <option value="書籍">書籍</option>
      <option value="服飾">服飾</option>
    </select>
    
    <div class="price-range">
      <input 
        v-model.number="priceRange.min" 
        type="number" 
        placeholder="最低價"
      >
      <input 
        v-model.number="priceRange.max" 
        type="number" 
        placeholder="最高價"
      >
    </div>
    
    <p>找到 {{ filteredProducts.length }} 項商品</p>
    
    <div v-for="product in filteredProducts" :key="product.id">
      <h3>{{ product.name }}</h3>
      <p>NT$ {{ product.price }}</p>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      searchKeyword: '',
      selectedCategory: '',
      priceRange: {
        min: 0,
        max: 100000
      },
      products: [
        { id: 1, name: '筆記型電腦', price: 35000, category: '3C' },
        { id: 2, name: 'JavaScript 權威指南', price: 880, category: '書籍' },
        { id: 3, name: '無線滑鼠', price: 890, category: '3C' },
        { id: 4, name: '運動外套', price: 1500, category: '服飾' }
      ]
    }
  },
  computed: {
    filteredProducts() {
      return this.products.filter(product => {
        // 關鍵字篩選
        const matchKeyword = product.name
          .toLowerCase()
          .includes(this.searchKeyword.toLowerCase())
        
        // 分類篩選
        const matchCategory = this.selectedCategory === '' || 
                             product.category === this.selectedCategory
        
        // 價格範圍篩選
        const matchPrice = product.price >= this.priceRange.min && 
                          product.price <= this.priceRange.max
        
        return matchKeyword && matchCategory && matchPrice
      })
    }
  },
  watch: {
    // 監聽搜尋關鍵字，記錄搜尋歷史
    searchKeyword(keyword) {
      if (keyword.length > 0) {
        this.recordSearchHistory(keyword)
      }
    },
    
    // 監聽篩選條件變化，發送分析事件
    filteredProducts(products) {
      console.log('篩選結果數量：', products.length)
      // 可以在這裡發送 GA 事件
    }
  },
  methods: {
    recordSearchHistory(keyword) {
      const history = JSON.parse(localStorage.getItem('searchHistory') || '[]')
      if (!history.includes(keyword)) {
        history.unshift(keyword)
        localStorage.setItem('searchHistory', JSON.stringify(history.slice(0, 10)))
      }
    }
  }
}
</script>
```

## Computed vs Watch 使用時機

### 使用 Computed

1. **需要根據現有資料計算新值**
2. **需要快取機制**
3. **同步操作**

```javascript
computed: {
  // 好的使用範例
  fullName() {
    return `${this.firstName} ${this.lastName}`
  },
  
  filteredList() {
    return this.items.filter(item => item.active)
  }
}
```

### 使用 Watch

1. **需要執行非同步操作**
2. **需要執行複雜或耗時的操作**
3. **需要在資料變化時觸發副作用**

```javascript
watch: {
  // 好的使用範例
  searchKeyword(newKeyword) {
    // 非同步 API 呼叫
    this.debouncedSearch(newKeyword)
  },
  
  cartItems: {
    handler(items) {
      // 同步到後端或 localStorage
      this.saveToLocalStorage(items)
    },
    deep: true
  }
}
```

## 小結

Computed 和 Watch 是 Vue 中處理資料變化的重要工具：

1. **Computed**：計算衍生資料，有快取，適合同步操作
2. **Watch**：監聽資料變化，執行副作用，適合非同步操作
3. **選擇原則**：能用 computed 就用 computed，需要非同步或複雜操作才用 watch

這兩個特性讓我們可以更優雅地處理資料邏輯，不需要在 methods 中寫一堆重複的計算程式碼。
