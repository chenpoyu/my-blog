---
layout: post
title: "Vue 過濾器與插槽"
date: 2020-04-26 09:30:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, Filters, Slots]
render_with_liquid: false
---

開發購物車時需要在模板中格式化資料，以及建立可重用的元件佈局。Vue 提供了過濾器（Filters）和插槽（Slots）來處理這些需求。

## 過濾器 Filters

過濾器用於格式化文字，在模板中使用管道符號 `|` 串接。

### 定義過濾器

區域過濾器：

```javascript
export default {
  filters: {
    uppercase(value) {
      return value.toUpperCase()
    },
    
    currency(value) {
      return `NT$ ${value.toLocaleString()}`
    }
  }
}
```

全域過濾器：

```javascript
// main.js
Vue.filter('currency', (value) => {
  return `NT$ ${value.toLocaleString()}`
})

Vue.filter('date', (value) => {
  const date = new Date(value)
  return date.toLocaleDateString('zh-TW')
})
```

### 使用過濾器

```vue
<template>
  <div>
    <!-- 單一過濾器 -->
    <p>{{ product.name | uppercase }}</p>
    <p>{{ product.price | currency }}</p>
    
    <!-- 串接多個過濾器 -->
    <p>{{ message | filterA | filterB }}</p>
    
    <!-- 過濾器可以接收參數 -->
    <p>{{ price | currency('USD') }}</p>
  </div>
</template>
```

### 常用過濾器範例

```javascript
// 金額格式化
Vue.filter('currency', (value, symbol = 'NT$') => {
  return `${symbol} ${value.toLocaleString()}`
})

// 日期格式化
Vue.filter('date', (value, format = 'short') => {
  const date = new Date(value)
  
  if (format === 'short') {
    return date.toLocaleDateString('zh-TW')
  } else if (format === 'long') {
    return date.toLocaleDateString('zh-TW', {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
      weekday: 'long'
    })
  } else {
    return date.toLocaleString('zh-TW')
  }
})

// 截斷文字
Vue.filter('truncate', (value, length = 50) => {
  if (value.length <= length) {
    return value
  }
  return value.substring(0, length) + '...'
})

// 訂單狀態
Vue.filter('orderStatus', (value) => {
  const statusMap = {
    pending: '待處理',
    processing: '處理中',
    shipped: '已出貨',
    delivered: '已送達',
    cancelled: '已取消'
  }
  return statusMap[value] || value
})
```

使用範例：

```vue
<template>
  <div class="order-list">
    <div v-for="order in orders" :key="order.id">
      <h3>訂單 #{{ order.orderNumber }}</h3>
      <p>日期：{{ order.createdAt | date }}</p>
      <p>狀態：{{ order.status | orderStatus }}</p>
      <p>金額：{{ order.totalAmount | currency }}</p>
    </div>
  </div>
</template>
```

## 插槽 Slots

插槽讓父元件可以向子元件傳遞模板內容，用於建立靈活的可重用元件。

### 基本插槽

定義有插槽的元件：

```vue
<!-- components/Card.vue -->
<template>
  <div class="card">
    <div class="card-header">
      <slot name="header"></slot>
    </div>
    <div class="card-body">
      <slot></slot>
    </div>
    <div class="card-footer">
      <slot name="footer"></slot>
    </div>
  </div>
</template>
```

使用插槽：

```vue
<template>
  <Card>
    <template v-slot:header>
      <h2>商品資訊</h2>
    </template>
    
    <p>這是商品內容</p>
    <p>價格：NT$ 1000</p>
    
    <template v-slot:footer>
      <button>加入購物車</button>
    </template>
  </Card>
</template>
```

簡寫語法：

```vue
<Card>
  <template #header>
    <h2>商品資訊</h2>
  </template>
  
  <p>預設插槽內容</p>
  
  <template #footer>
    <button>加入購物車</button>
  </template>
</Card>
```

### 預設內容

```vue
<!-- components/Button.vue -->
<template>
  <button class="custom-button">
    <slot>預設按鈕文字</slot>
  </button>
</template>
```

使用：

```vue
<!-- 使用預設內容 -->
<Button></Button>
<!-- 渲染：<button>預設按鈕文字</button> -->

<!-- 自訂內容 -->
<Button>加入購物車</Button>
<!-- 渲染：<button>加入購物車</button> -->
```

### 作用域插槽

子元件可以將資料傳遞給父元件的插槽。

```vue
<!-- components/ProductList.vue -->
<template>
  <div class="product-list">
    <div v-for="product in products" :key="product.id">
      <slot :product="product" :index="index">
        <!-- 預設內容 -->
        <h3>{{ product.name }}</h3>
        <p>{{ product.price }}</p>
      </slot>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    products: {
      type: Array,
      required: true
    }
  }
}
</script>
```

使用作用域插槽：

```vue
<template>
  <ProductList :products="products">
    <template v-slot="{ product, index }">
      <div class="custom-product-card">
        <span class="badge">{{ index + 1 }}</span>
        <h3>{{ product.name }}</h3>
        <p class="price">NT$ {{ product.price }}</p>
        <button @click="addToCart(product)">加入購物車</button>
      </div>
    </template>
  </ProductList>
</template>
```

簡寫：

```vue
<ProductList :products="products" v-slot="{ product }">
  <h3>{{ product.name }}</h3>
  <p>{{ product.price }}</p>
</ProductList>
```

## 實戰範例：可重用元件

### 通用卡片元件

```vue
<!-- components/BaseCard.vue -->
<template>
  <div class="base-card" :class="cardClass">
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

<script>
export default {
  name: 'BaseCard',
  props: {
    variant: {
      type: String,
      default: 'default',
      validator: (value) => {
        return ['default', 'primary', 'danger', 'success'].includes(value)
      }
    }
  },
  computed: {
    cardClass() {
      return `card-${this.variant}`
    }
  }
}
</script>

<style scoped>
.base-card {
  border: 1px solid #ddd;
  border-radius: 4px;
  overflow: hidden;
}

.card-header {
  padding: 15px;
  background-color: #f5f5f5;
  border-bottom: 1px solid #ddd;
}

.card-body {
  padding: 15px;
}

.card-footer {
  padding: 15px;
  background-color: #f5f5f5;
  border-top: 1px solid #ddd;
}

.card-primary {
  border-color: #42b983;
}

.card-danger {
  border-color: #f56c6c;
}
</style>
```

使用：

```vue
<template>
  <div>
    <BaseCard variant="primary">
      <template #header>
        <h3>商品詳情</h3>
      </template>
      
      <p>商品名稱：筆記型電腦</p>
      <p>價格：{{ 35000 | currency }}</p>
      
      <template #footer>
        <button>加入購物車</button>
      </template>
    </BaseCard>
  </div>
</template>
```

### 通用對話框元件

```vue
<!-- components/Modal.vue -->
<template>
  <transition name="modal">
    <div v-if="visible" class="modal-overlay" @click="handleOverlayClick">
      <div class="modal-container" @click.stop>
        <div class="modal-header">
          <slot name="header">
            <h3>{{ title }}</h3>
          </slot>
          <button class="close-button" @click="close">×</button>
        </div>
        
        <div class="modal-body">
          <slot></slot>
        </div>
        
        <div class="modal-footer">
          <slot name="footer">
            <button @click="close">關閉</button>
          </slot>
        </div>
      </div>
    </div>
  </transition>
</template>

<script>
export default {
  name: 'Modal',
  props: {
    visible: {
      type: Boolean,
      default: false
    },
    title: {
      type: String,
      default: ''
    },
    closeOnClickOutside: {
      type: Boolean,
      default: true
    }
  },
  methods: {
    close() {
      this.$emit('close')
    },
    
    handleOverlayClick() {
      if (this.closeOnClickOutside) {
        this.close()
      }
    }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-container {
  background-color: white;
  border-radius: 8px;
  max-width: 500px;
  width: 90%;
  max-height: 90vh;
  overflow: auto;
}

.modal-header {
  padding: 20px;
  border-bottom: 1px solid #ddd;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-body {
  padding: 20px;
}

.modal-footer {
  padding: 20px;
  border-top: 1px solid #ddd;
  text-align: right;
}

.close-button {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
}

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s;
}

.modal-enter,
.modal-leave-to {
  opacity: 0;
}
</style>
```

使用：

```vue
<template>
  <div>
    <button @click="showModal = true">開啟對話框</button>
    
    <Modal :visible="showModal" @close="showModal = false" title="確認刪除">
      <p>確定要刪除此商品嗎？此操作無法復原。</p>
      
      <template #footer>
        <button @click="showModal = false">取消</button>
        <button @click="handleDelete" class="danger">刪除</button>
      </template>
    </Modal>
  </div>
</template>

<script>
export default {
  data() {
    return {
      showModal: false
    }
  },
  methods: {
    handleDelete() {
      console.log('刪除商品')
      this.showModal = false
    }
  }
}
</script>
```

### 表格元件

```vue
<!-- components/DataTable.vue -->
<template>
  <table class="data-table">
    <thead>
      <tr>
        <th v-for="column in columns" :key="column.key">
          {{ column.label }}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(row, index) in data" :key="index">
        <td v-for="column in columns" :key="column.key">
          <slot 
            :name="`cell-${column.key}`" 
            :row="row" 
            :value="row[column.key]"
            :index="index"
          >
            {{ row[column.key] }}
          </slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
export default {
  name: 'DataTable',
  props: {
    columns: {
      type: Array,
      required: true
    },
    data: {
      type: Array,
      required: true
    }
  }
}
</script>
```

使用：

```vue
<template>
  <DataTable :columns="columns" :data="orders">
    <template #cell-orderNumber="{ value }">
      <strong>{{ value }}</strong>
    </template>
    
    <template #cell-createdAt="{ value }">
      {{ value | date('long') }}
    </template>
    
    <template #cell-totalAmount="{ value }">
      <span class="price">{{ value | currency }}</span>
    </template>
    
    <template #cell-status="{ value }">
      <span :class="`badge badge-${value}`">
        {{ value | orderStatus }}
      </span>
    </template>
    
    <template #cell-actions="{ row }">
      <button @click="viewOrder(row.id)">查看</button>
      <button @click="cancelOrder(row.id)">取消</button>
    </template>
  </DataTable>
</template>

<script>
export default {
  data() {
    return {
      columns: [
        { key: 'orderNumber', label: '訂單編號' },
        { key: 'createdAt', label: '訂單日期' },
        { key: 'totalAmount', label: '金額' },
        { key: 'status', label: '狀態' },
        { key: 'actions', label: '操作' }
      ],
      orders: [
        {
          id: 1,
          orderNumber: 'ORD-001',
          createdAt: '2020-04-26T10:00:00',
          totalAmount: 35000,
          status: 'pending'
        }
      ]
    }
  }
}
</script>
```

## 過濾器 vs 方法 vs 計算屬性

```vue
<template>
  <div>
    <!-- 過濾器：適合簡單的格式化 -->
    <p>{{ price | currency }}</p>
    
    <!-- 方法：需要傳遞參數或複雜邏輯 -->
    <p>{{ formatPrice(price, 'USD') }}</p>
    
    <!-- 計算屬性：需要快取 -->
    <p>{{ formattedPrice }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      price: 35000
    }
  },
  computed: {
    formattedPrice() {
      return `NT$ ${this.price.toLocaleString()}`
    }
  },
  methods: {
    formatPrice(value, currency) {
      return `${currency} ${value.toLocaleString()}`
    }
  }
}
</script>
```

## 小結

過濾器和插槽的使用時機：

**過濾器：**
- 簡單的文字格式化
- 在模板中直接使用
- 保持邏輯簡單，複雜邏輯使用計算屬性

**插槽：**
- 建立可重用的元件佈局
- 需要自訂內容的元件
- 父子元件之間傳遞模板內容

這兩個特性讓元件更加靈活和可重用，是建構複雜 UI 的重要工具。
