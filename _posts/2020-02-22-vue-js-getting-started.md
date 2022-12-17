---
layout: post
title: "Vue.js 初探：從後端工程師的角度"
date: 2020-02-22 10:30:00 +0800
categories: [前端, JavaScript]
tags: [Vue.js, JavaScript, 前端框架]
---

最近專案需要使用 Vue.js 開發前端頁面，對於習慣寫後端的我來說，前端框架是相對陌生的領域。雖然之前有學過 Swift 和基本的 jQuery，但 Vue.js 的開發思維還是需要重新理解。

> 本文環境：**Vue 2.6.x** + **Node.js 12.x** + **npm**

## 為什麼是 Vue.js

專案需求是開發購物車與商品展示的電商功能，包含登入、瀏覽商品、購物車操作、下單等功能。選擇 Vue.js 的理由：

1. **漸進式框架**：可以從簡單的頁面開始，不需要一開始就建立複雜架構
2. **易學習曲線**：對於有 jQuery 基礎的人來說，模板語法相對直觀
3. **完整生態系**：Vue Router 處理路由、Vuex 管理狀態，官方套件齊全

## 環境準備

### 安裝 Node.js 與 Vue CLI

```bash
# 檢查 Node.js 版本
node --version
# v12.18.0

# 安裝 Vue CLI
npm install -g @vue/cli

# 驗證安裝
vue --version
# @vue/cli 4.5.11
```

### 建立第一個專案

```bash
# 使用 Vue CLI 建立專案
vue create shopping-mall

# 選擇 Vue 2
# 選擇預設配置即可
```

CLI 會詢問一些配置問題，先選擇預設的 Vue 2 即可。專案結構類似這樣：

```
shopping-mall/
├── public/
│   └── index.html
├── src/
│   ├── assets/        # 靜態資源
│   ├── components/    # 元件
│   ├── App.vue        # 根元件
│   └── main.js        # 入口檔案
├── package.json
└── vue.config.js
```

## Vue 的核心概念

從後端角度理解 Vue：

### 1. 元件 (Component)

類似後端的類別，每個元件封裝自己的邏輯、樣式和模板。

```vue
<template>
  <div class="product-item">
    <h3>{{ productName }}</h3>
    <p>價格：{{ price }} 元</p>
  </div>
</template>

<script>
export default {
  name: 'ProductItem',
  data() {
    return {
      productName: '商品名稱',
      price: 1000
    }
  }
}
</script>

<style scoped>
.product-item {
  border: 1px solid #ccc;
  padding: 10px;
}
</style>
```

元件分為三個部分：
- **template**：HTML 模板，類似 JSP 或 Thymeleaf
- **script**：JavaScript 邏輯，類似 Controller
- **style**：CSS 樣式

### 2. 響應式資料 (Reactive Data)

Vue 最核心的特性，資料改變時，畫面會自動更新。

```javascript
export default {
  data() {
    return {
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++  // 畫面會自動更新
    }
  }
}
```

對比後端：類似 Observer Pattern，但由框架自動處理。

### 3. 指令 (Directives)

Vue 提供的特殊 HTML 屬性，用來綁定資料或處理事件。

```html
<!-- v-bind：綁定屬性 -->
<img v-bind:src="imageUrl">
<!-- 縮寫 -->
<img :src="imageUrl">

<!-- v-on：綁定事件 -->
<button v-on:click="handleClick">點擊</button>
<!-- 縮寫 -->
<button @click="handleClick">點擊</button>

<!-- v-if：條件渲染 -->
<div v-if="isLoggedIn">歡迎回來</div>
<div v-else>請先登入</div>

<!-- v-for：列表渲染 -->
<ul>
  <li v-for="item in products" :key="item.id">
    {{ item.name }}
  </li>
</ul>
```

## 第一個範例：商品列表

建立一個簡單的商品列表頁面：

```vue
<template>
  <div class="product-list">
    <h2>商品列表</h2>
    <div v-for="product in products" :key="product.id" class="product-card">
      <h3>{{ product.name }}</h3>
      <p>價格：NT$ {{ product.price }}</p>
      <p>庫存：{{ product.stock }}</p>
      <button @click="addToCart(product)">加入購物車</button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ProductList',
  data() {
    return {
      products: [
        { id: 1, name: '筆記型電腦', price: 35000, stock: 5 },
        { id: 2, name: '無線滑鼠', price: 890, stock: 20 },
        { id: 3, name: '機械鍵盤', price: 2500, stock: 10 }
      ]
    }
  },
  methods: {
    addToCart(product) {
      console.log('加入購物車：', product.name)
      // 後續會實作購物車邏輯
    }
  }
}
</script>

<style scoped>
.product-list {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.product-card {
  border: 1px solid #ddd;
  padding: 15px;
  margin-bottom: 15px;
  border-radius: 4px;
}

.product-card button {
  background-color: #42b983;
  color: white;
  border: none;
  padding: 8px 16px;
  cursor: pointer;
  border-radius: 4px;
}

.product-card button:hover {
  background-color: #359268;
}
</style>
```

## 執行專案

```bash
# 開發模式
npm run serve

# 瀏覽器開啟 http://localhost:8080
```

修改程式碼後，瀏覽器會自動重新載入（Hot Reload），這點比起傳統的 jQuery 開發方便很多。

## 與 jQuery 的差異

過去使用 jQuery 的做法：

```javascript
// jQuery 方式：手動操作 DOM
$('#product-list').html('');
products.forEach(product => {
  $('#product-list').append(`
    <div class="product">
      <h3>${product.name}</h3>
      <p>${product.price}</p>
    </div>
  `);
});
```

Vue 的方式：

```vue
<!-- 宣告式渲染，不直接操作 DOM -->
<div v-for="product in products" :key="product.id">
  <h3>{{ product.name }}</h3>
  <p>{{ product.price }}</p>
</div>
```

Vue 是「資料驅動」的思維，只需要關注資料的變化，畫面會自動更新。jQuery 則是「命令式」，需要手動操作 DOM。

## 小結

第一次接觸 Vue.js，最大的感受是：

1. **元件化思維**：類似後端的模組化，每個功能獨立封裝
2. **響應式很神奇**：資料改變畫面就跟著變，不需要手動更新 DOM
3. **學習曲線還算友善**：模板語法類似 HTML，JavaScript 邏輯也不複雜

接下來幾週會持續學習更多 Vue 特性，包括元件通訊、路由管理、狀態管理等，目標是完成購物車專案。
