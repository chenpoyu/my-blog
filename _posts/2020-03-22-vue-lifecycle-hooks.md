---
layout: post
title: "Vue 生命週期鉤子函數"
date: 2020-03-22 11:05:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Lifecycle, Hooks]
---

開發購物車功能時遇到一個問題：什麼時候該載入商品資料？什麼時候該清理計時器？Vue 的生命週期鉤子函數提供了在元件不同階段執行程式碼的機制。

## 生命週期圖解

Vue 元件從建立到銷毀會經歷幾個階段：

```
建立階段：
  beforeCreate  -> 實例初始化之後，資料觀測之前
  created       -> 實例建立完成，資料已設定

掛載階段：
  beforeMount   -> 掛載到 DOM 之前
  mounted       -> 掛載到 DOM 完成

更新階段：
  beforeUpdate  -> 資料更新時，DOM 重新渲染之前
  updated       -> DOM 重新渲染完成

銷毀階段：
  beforeDestroy -> 實例銷毀之前
  destroyed     -> 實例銷毀完成
```

類似後端的 Servlet 生命週期（init -> service -> destroy），但更細緻。

## 各階段詳解

### beforeCreate

實例剛初始化，資料觀測和事件配置還沒開始。

```javascript
export default {
  data() {
    return {
      message: 'Hello'
    }
  },
  beforeCreate() {
    console.log('beforeCreate')
    console.log(this.message)  // undefined
    console.log(this.$el)      // undefined
  }
}
```

使用場景：很少使用，通常在插件開發時才需要。

### created

實例建立完成，資料、computed、methods 都已設定，但還沒掛載到 DOM。

```javascript
export default {
  data() {
    return {
      products: []
    }
  },
  created() {
    console.log('created')
    console.log(this.products)  // []，可以存取
    console.log(this.$el)       // undefined，還沒掛載
    
    // 適合在這裡取得初始資料
    this.fetchProducts()
  },
  methods: {
    async fetchProducts() {
      const response = await fetch('/api/products')
      this.products = await response.json()
    }
  }
}
```

使用場景：
- 呼叫 API 取得初始資料
- 設定初始狀態
- 不需要操作 DOM 的初始化邏輯

### beforeMount

掛載到 DOM 之前，模板編譯完成但還沒渲染。

```javascript
beforeMount() {
  console.log('beforeMount')
  console.log(this.$el)  // undefined
}
```

使用場景：很少使用。

### mounted

元件已掛載到 DOM，可以存取 DOM 元素。

```javascript
export default {
  mounted() {
    console.log('mounted')
    console.log(this.$el)  // DOM 元素
    
    // 可以操作 DOM
    this.$refs.input.focus()
    
    // 整合第三方函式庫
    this.initChart()
  },
  methods: {
    initChart() {
      // 使用 Chart.js 繪製圖表
      const ctx = this.$refs.canvas.getContext('2d')
      // ...
    }
  }
}
```

使用場景：
- 需要操作 DOM
- 整合第三方 UI 函式庫（如 jQuery plugin、Chart.js）
- 設定事件監聽器

### beforeUpdate

資料更新時，DOM 重新渲染之前。

```javascript
export default {
  data() {
    return {
      count: 0
    }
  },
  beforeUpdate() {
    console.log('beforeUpdate')
    console.log('DOM 中的值：', this.$el.textContent)  // 舊值
    console.log('資料中的值：', this.count)           // 新值
  }
}
```

使用場景：
- 在 DOM 更新前存取舊的 DOM 狀態
- 很少使用

### updated

DOM 重新渲染完成。

```javascript
export default {
  data() {
    return {
      count: 0
    }
  },
  updated() {
    console.log('updated')
    console.log('DOM 已更新：', this.$el.textContent)
  }
}
```

使用場景：
- 需要在 DOM 更新後執行某些操作
- 注意：避免在這裡修改資料，會造成無限迴圈

### beforeDestroy

元件銷毀之前，實例仍然完全可用。

```javascript
export default {
  data() {
    return {
      timer: null
    }
  },
  created() {
    // 設定計時器
    this.timer = setInterval(() => {
      console.log('tick')
    }, 1000)
  },
  beforeDestroy() {
    console.log('beforeDestroy')
    
    // 清理計時器
    if (this.timer) {
      clearInterval(this.timer)
      this.timer = null
    }
    
    // 移除事件監聽器
    window.removeEventListener('resize', this.handleResize)
  }
}
```

使用場景：
- 清理計時器
- 移除事件監聽器
- 清理第三方函式庫實例

### destroyed

元件銷毀完成，所有綁定和監聽器都已移除。

```javascript
destroyed() {
  console.log('destroyed')
  // 元件已完全銷毀
}
```

使用場景：很少使用。

## 實戰範例：商品輪播

```vue
<template>
  <div class="product-carousel">
    <div class="carousel-item">
      <img :src="currentProduct.image" :alt="currentProduct.name">
      <h3>{{ currentProduct.name }}</h3>
      <p>NT$ {{ currentProduct.price }}</p>
    </div>
    <div class="carousel-indicators">
      <span 
        v-for="(product, index) in products" 
        :key="product.id"
        :class="{ active: index === currentIndex }"
        @click="goToSlide(index)"
      ></span>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ProductCarousel',
  data() {
    return {
      products: [],
      currentIndex: 0,
      timer: null,
      interval: 3000
    }
  },
  computed: {
    currentProduct() {
      return this.products[this.currentIndex] || {}
    }
  },
  created() {
    console.log('[created] 載入商品資料')
    this.fetchProducts()
  },
  mounted() {
    console.log('[mounted] 啟動自動輪播')
    this.startAutoPlay()
    
    // 滑鼠移入時暫停，移出時繼續
    this.$el.addEventListener('mouseenter', this.pauseAutoPlay)
    this.$el.addEventListener('mouseleave', this.startAutoPlay)
  },
  beforeDestroy() {
    console.log('[beforeDestroy] 清理資源')
    
    // 清除計時器
    this.pauseAutoPlay()
    
    // 移除事件監聽器
    this.$el.removeEventListener('mouseenter', this.pauseAutoPlay)
    this.$el.removeEventListener('mouseleave', this.startAutoPlay)
  },
  methods: {
    async fetchProducts() {
      // 模擬 API 呼叫
      await new Promise(resolve => setTimeout(resolve, 500))
      this.products = [
        { id: 1, name: '商品 A', price: 1000, image: '/img/a.jpg' },
        { id: 2, name: '商品 B', price: 2000, image: '/img/b.jpg' },
        { id: 3, name: '商品 C', price: 3000, image: '/img/c.jpg' }
      ]
    },
    
    startAutoPlay() {
      if (this.timer) return
      
      this.timer = setInterval(() => {
        this.nextSlide()
      }, this.interval)
    },
    
    pauseAutoPlay() {
      if (this.timer) {
        clearInterval(this.timer)
        this.timer = null
      }
    },
    
    nextSlide() {
      this.currentIndex = (this.currentIndex + 1) % this.products.length
    },
    
    goToSlide(index) {
      this.currentIndex = index
    }
  }
}
</script>
```

## 實戰範例：資料快取

```vue
<script>
export default {
  name: 'ProductDetail',
  data() {
    return {
      product: null,
      loading: false
    }
  },
  computed: {
    productId() {
      return this.$route.params.id
    }
  },
  created() {
    this.loadProduct()
  },
  watch: {
    // 路由參數變化時重新載入
    productId() {
      this.loadProduct()
    }
  },
  methods: {
    async loadProduct() {
      // 檢查 sessionStorage 是否有快取
      const cacheKey = `product_${this.productId}`
      const cached = sessionStorage.getItem(cacheKey)
      
      if (cached) {
        console.log('使用快取資料')
        this.product = JSON.parse(cached)
        return
      }
      
      // 沒有快取，從 API 載入
      this.loading = true
      try {
        const response = await fetch(`/api/products/${this.productId}`)
        this.product = await response.json()
        
        // 存入快取
        sessionStorage.setItem(cacheKey, JSON.stringify(this.product))
      } catch (error) {
        console.error('載入失敗', error)
      } finally {
        this.loading = false
      }
    }
  },
  beforeDestroy() {
    // 元件銷毀時可以選擇清除快取或保留
    // sessionStorage.removeItem(`product_${this.productId}`)
  }
}
</script>
```

## 常見錯誤

### 1. 在 created 中操作 DOM

```javascript
// 錯誤：created 時 DOM 還沒掛載
created() {
  this.$refs.input.focus()  // Error: Cannot read property 'focus' of undefined
}

// 正確：在 mounted 中操作
mounted() {
  this.$refs.input.focus()  // OK
}
```

### 2. 忘記清理資源

```javascript
// 錯誤：沒有清理計時器
created() {
  setInterval(() => {
    this.updateTime()
  }, 1000)
}

// 正確：在銷毀前清理
data() {
  return {
    timer: null
  }
},
created() {
  this.timer = setInterval(() => {
    this.updateTime()
  }, 1000)
},
beforeDestroy() {
  if (this.timer) {
    clearInterval(this.timer)
  }
}
```

### 3. 在 updated 中修改資料

```javascript
// 錯誤：造成無限迴圈
updated() {
  this.count++  // 會觸發 updated，又執行 this.count++，無限迴圈
}

// 正確：使用 watch 或其他鉤子
watch: {
  someData(newVal) {
    this.count++
  }
}
```

## 小結

生命週期鉤子函數的使用時機：

1. **created**：取得初始資料、設定初始狀態
2. **mounted**：操作 DOM、整合第三方函式庫
3. **beforeDestroy**：清理資源（計時器、事件監聽器）

理解生命週期對於寫出正確的 Vue 程式碼很重要，特別是涉及非同步操作、DOM 操作、資源清理時。
