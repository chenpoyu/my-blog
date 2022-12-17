---
layout: post
title: "Vue 混入與自訂指令"
date: 2020-04-19 16:50:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Mixins, Directives]
render_with_liquid: false
---

開發購物車專案時發現有些邏輯在多個元件中重複出現，例如格式化日期、權限檢查等。Vue 提供了混入（Mixins）和自訂指令（Custom Directives）來重用程式碼。

## Mixins 混入

混入可以將重複的邏輯抽取出來，在多個元件中重用。類似後端的抽象類別或工具類別。

### 基本用法

建立混入檔案：

```javascript
// src/mixins/formatting.js
export default {
  methods: {
    formatPrice(price) {
      return `NT$ ${price.toLocaleString()}`
    },
    
    formatDate(dateString) {
      const date = new Date(dateString)
      return date.toLocaleDateString('zh-TW', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit'
      })
    },
    
    formatDateTime(dateString) {
      const date = new Date(dateString)
      return date.toLocaleString('zh-TW')
    }
  }
}
```

在元件中使用：

```vue
<template>
  <div>
    <p>價格：{{ formatPrice(product.price) }}</p>
    <p>上架日期：{{ formatDate(product.createdAt) }}</p>
  </div>
</template>

<script>
import formattingMixin from '@/mixins/formatting'

export default {
  mixins: [formattingMixin],
  data() {
    return {
      product: {
        price: 35000,
        createdAt: '2020-04-19T10:00:00'
      }
    }
  }
}
</script>
```

### 生命週期混入

```javascript
// src/mixins/pageTracking.js
export default {
  mounted() {
    console.log(`進入頁面：${this.$options.name}`)
    // 發送 GA 事件
    if (window.gtag) {
      window.gtag('event', 'page_view', {
        page_title: this.$options.name,
        page_path: this.$route.path
      })
    }
  },
  
  beforeDestroy() {
    console.log(`離開頁面：${this.$options.name}`)
  }
}
```

### 資料和計算屬性混入

```javascript
// src/mixins/auth.js
import { mapGetters } from 'vuex'

export default {
  computed: {
    ...mapGetters('auth', ['isLoggedIn', 'user'])
  },
  
  methods: {
    requireLogin() {
      if (!this.isLoggedIn) {
        this.$router.push({
          path: '/login',
          query: { redirect: this.$route.fullPath }
        })
        return false
      }
      return true
    },
    
    hasRole(role) {
      return this.user && this.user.roles.includes(role)
    }
  }
}
```

使用：

```vue
<script>
import authMixin from '@/mixins/auth'

export default {
  mixins: [authMixin],
  methods: {
    handleCheckout() {
      if (!this.requireLogin()) {
        return
      }
      
      if (!this.hasRole('buyer')) {
        alert('沒有購買權限')
        return
      }
      
      // 執行結帳
      this.$router.push('/checkout')
    }
  }
}
</script>
```

### 選項合併規則

當混入和元件有相同選項時：

```javascript
const mixin = {
  data() {
    return {
      message: 'from mixin',
      count: 100
    }
  },
  methods: {
    foo() {
      console.log('mixin foo')
    }
  },
  mounted() {
    console.log('mixin mounted')
  }
}

export default {
  mixins: [mixin],
  data() {
    return {
      message: 'from component',  // 元件的優先
      value: 200
    }
  },
  methods: {
    foo() {
      console.log('component foo')  // 元件的優先
    },
    bar() {
      console.log('component bar')
    }
  },
  mounted() {
    console.log('component mounted')  // 都會執行，mixin 先
  }
}

// 結果：
// data: { message: 'from component', count: 100, value: 200 }
// methods: { foo: component版本, bar: component版本 }
// mounted 輸出：
//   "mixin mounted"
//   "component mounted"
```

## 自訂指令

指令可以直接操作 DOM 元素，類似 jQuery plugin 的概念。

### 全域指令

```javascript
// main.js
Vue.directive('focus', {
  // 當綁定元素插入到 DOM 時
  inserted(el) {
    el.focus()
  }
})
```

使用：

```vue
<template>
  <input v-focus>
</template>
```

### 區域指令

```vue
<script>
export default {
  directives: {
    focus: {
      inserted(el) {
        el.focus()
      }
    }
  }
}
</script>
```

### 帶參數的指令

```javascript
// 自訂顏色指令
Vue.directive('color', {
  bind(el, binding) {
    el.style.color = binding.value
  },
  update(el, binding) {
    el.style.color = binding.value
  }
})
```

使用：

```vue
<template>
  <p v-color="'red'">紅色文字</p>
  <p v-color="textColor">動態顏色</p>
</template>

<script>
export default {
  data() {
    return {
      textColor: 'blue'
    }
  }
}
</script>
```

### 權限控制指令

```javascript
// src/directives/permission.js
export default {
  inserted(el, binding, vnode) {
    const { value } = binding
    const roles = vnode.context.$store.getters['auth/user']?.roles || []
    
    if (value && !roles.includes(value)) {
      // 移除元素
      el.parentNode && el.parentNode.removeChild(el)
    }
  }
}
```

```javascript
// main.js
import permission from './directives/permission'
Vue.directive('permission', permission)
```

使用：

```vue
<template>
  <div>
    <button v-permission="'admin'">管理功能</button>
    <button v-permission="'seller'">上架商品</button>
  </div>
</template>
```

### 點擊外部關閉指令

```javascript
// src/directives/clickOutside.js
export default {
  bind(el, binding) {
    el.clickOutsideEvent = (event) => {
      // 檢查點擊是否在元素外部
      if (!(el === event.target || el.contains(event.target))) {
        // 呼叫綁定的方法
        binding.value(event)
      }
    }
    document.addEventListener('click', el.clickOutsideEvent)
  },
  
  unbind(el) {
    document.removeEventListener('click', el.clickOutsideEvent)
  }
}
```

使用：

```vue
<template>
  <div class="dropdown">
    <button @click="isOpen = !isOpen">選單</button>
    <div v-show="isOpen" v-click-outside="closeDropdown" class="dropdown-menu">
      <a href="#">選項 1</a>
      <a href="#">選項 2</a>
    </div>
  </div>
</template>

<script>
import clickOutside from '@/directives/clickOutside'

export default {
  directives: {
    clickOutside
  },
  data() {
    return {
      isOpen: false
    }
  },
  methods: {
    closeDropdown() {
      this.isOpen = false
    }
  }
}
</script>
```

### 防抖指令

```javascript
// src/directives/debounce.js
export default {
  inserted(el, binding) {
    let timer
    el.addEventListener('input', () => {
      if (timer) {
        clearTimeout(timer)
      }
      timer = setTimeout(() => {
        binding.value()
      }, binding.arg || 500)
    })
  }
}
```

使用：

```vue
<template>
  <input 
    v-model="searchKeyword" 
    v-debounce:300="handleSearch"
    placeholder="搜尋商品"
  >
</template>

<script>
import debounce from '@/directives/debounce'

export default {
  directives: {
    debounce
  },
  data() {
    return {
      searchKeyword: ''
    }
  },
  methods: {
    handleSearch() {
      console.log('執行搜尋：', this.searchKeyword)
      // 呼叫 API
    }
  }
}
</script>
```

### 圖片懶載入指令

```javascript
// src/directives/lazyload.js
export default {
  inserted(el, binding) {
    const imgSrc = binding.value
    
    // 使用 Intersection Observer API
    const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          // 圖片進入視窗，載入真實圖片
          el.src = imgSrc
          // 停止觀察
          observer.unobserve(el)
        }
      })
    })
    
    // 開始觀察
    observer.observe(el)
    
    // 設定預設圖片
    el.src = 'data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7'
  }
}
```

使用：

```vue
<template>
  <div class="product-list">
    <div v-for="product in products" :key="product.id">
      <img v-lazyload="product.image" :alt="product.name">
      <h3>{{ product.name }}</h3>
    </div>
  </div>
</template>

<script>
import lazyload from '@/directives/lazyload'

export default {
  directives: {
    lazyload
  }
}
</script>
```

## 實戰範例：完整應用

### 全域混入與指令

```javascript
// main.js
import Vue from 'vue'
import App from './App.vue'
import store from './store'
import router from './router'

// 全域混入
import formattingMixin from './mixins/formatting'
Vue.mixin(formattingMixin)

// 全域指令
import permission from './directives/permission'
import clickOutside from './directives/clickOutside'
import lazyload from './directives/lazyload'

Vue.directive('permission', permission)
Vue.directive('click-outside', clickOutside)
Vue.directive('lazyload', lazyload)

new Vue({
  store,
  router,
  render: h => h(App)
}).$mount('#app')
```

### 載入狀態混入

```javascript
// src/mixins/loading.js
export default {
  data() {
    return {
      loading: false,
      error: null
    }
  },
  
  methods: {
    async withLoading(asyncFn) {
      this.loading = true
      this.error = null
      
      try {
        return await asyncFn()
      } catch (error) {
        console.error(error)
        this.error = error.message || '操作失敗'
        throw error
      } finally {
        this.loading = false
      }
    }
  }
}
```

使用：

```vue
<template>
  <div>
    <div v-if="loading">載入中...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- 內容 -->
    </div>
  </div>
</template>

<script>
import loadingMixin from '@/mixins/loading'
import api from '@/api'

export default {
  mixins: [loadingMixin],
  data() {
    return {
      products: []
    }
  },
  async created() {
    await this.withLoading(async () => {
      const response = await api.products.getProducts()
      this.products = response.data
    })
  }
}
</script>
```

## Mixins 的注意事項

Mixins 雖然方便，但使用時需要注意一些問題：

1. **命名衝突**：多個 mixin 可能有相同的屬性或方法
2. **來源不明確**：不容易追蹤方法來自哪個 mixin
3. **隱式依賴**：mixin 之間可能有隱式依賴關係

因此在使用 Mixins 時，建議：
- 明確的命名規範，避免衝突
- 適當的註解說明 mixin 的用途
- 避免過度使用，保持元件的清晰性

## 小結

Mixins 和自訂指令的使用時機：

**Mixins：**
- 重複的業務邏輯（格式化、權限檢查）
- 共用的生命週期處理
- 共用的計算屬性和方法

**自訂指令：**
- 需要直接操作 DOM
- 封裝 DOM 互動行為（拖曳、縮放）
- 與第三方函式庫整合

兩者都是程式碼重用的工具，但要避免過度使用，保持程式碼的可讀性和可維護性。
