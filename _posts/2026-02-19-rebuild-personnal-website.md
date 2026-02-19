---
layout: post
title: "隔了快五年，重新翻修自己的個人網站"
date: 2026-02-19 12:00:00 +0800
categories: [技術筆記, 前端開發]
tags: [Vue 3, Quasar Framework, i18n, 個人網站, GitHub Pages]
---

## 事情的起頭

上一次認真動自己的網站，是 2021 年的事了。那時候做了一個 demo 頁面給客戶看，順便把個人頁面掛上去，但說實話根本沒花什麼心思。中間 2024 年小改了一版，也就微調而已。

一直到最近，覺得自己都在幫別人蓋系統、帶團隊，結果回頭看自己的網站，長得像五年前的東西（因為它確實是），實在說不過去。趁著過年後比較有空，決定整個砍掉重練。

## 為什麼要重寫而不是修修補補

舊版的問題很多：

- 原本是直接打包好的靜態檔丟上 GitHub Pages，html/css/js 散落一地，根本沒有工程化
- 之前還塞了一個 demo 資料夾放客戶的 demo 頁面在裡面，公私混在一起
- 完全沒有 RWD，手機看起來慘不忍睹
- 沒有多語系、沒有深色模式、沒有 SEO，什麼都沒有
- 想改點東西就得去翻打包後的程式碼，根本不會想去碰

所以這次的目標很明確：**用現代前端框架整個重寫，當作一個正經的專案來做。**

## 技術選型：為什麼選 Quasar + Vue 3

考慮過幾個方案：

1. **純 HTML + Tailwind**：太陽春，我想要元件化管理
2. **Next.js / Nuxt.js**：對個人網站來說太重了
3. **Quasar + Vue 3**：我工作上本來就很熟 Vue，Quasar 自帶大量 UI 元件、支援 RWD、深色模式開箱即用

最後選了 Quasar，主要原因是**不用自己造輪子**。像 Header 的 RWD 選單、深色模式切換、表單驗證這些東西，Quasar 通通有現成元件。我只想把時間花在內容跟設計上，不想花在刻基本功能。

## 整體架構

整個專案的結構長這樣：

```
src/
├── pages/           # 各個頁面
│   ├── Home.vue         # 首頁（主打頁面，最複雜）
│   ├── profile/         # 關於我 / 履歷
│   ├── work/            # 服務項目
│   ├── portfolio/       # 作品集
│   └── TechArchitecture.vue  # 技術架構展示
├── i18n/            # 多語系
│   ├── zh-tw/           # 繁體中文
│   └── en-us/           # 英文
├── components/      # 共用元件
├── layouts/         # 版面（Header + Footer）
├── router/          # 路由設定
├── stores/          # Pinia 狀態管理
├── utils/           # 工具（SEO 結構化資料）
└── css/             # 全域樣式
```

不算多複雜，但對一個個人網站來說夠用了。

## 這次改版做了哪些事

### 1. 完整的多語系支援（vue-i18n）

所有文字都抽到 `i18n/` 資料夾，分成 `zh-tw` 跟 `en-us` 兩個語系。每個頁面有自己的翻譯檔：`home.js`、`services.js`、`techarch.js`、`portfolio.js`。

切換語系的元件也做了，在 Header 上可以直接選語言。做這個最麻煩的不是技術，是**翻譯**。中文寫好再翻英文，英文翻完覺得語意怪怪的再回來改中文，來來回回搞了不少時間。

### 2. 深色模式

Quasar 內建 `$q.dark.mode` 就能切換，我在 CSS 裡用 CSS Variables 處理不同模式下的顏色：

```scss
body {
    color: var(--font-color-primary);
    background-color: var(--background-color);
}
```

其實這段不難，難的是確保所有頁面在深色模式下看起來都正常。每個卡片、每個背景漸層都得測過。

### 3. 首頁大改版

首頁是花最多時間的地方，大概佔了整個專案 40% 的工時。裡面塞了：

- **Hero 區塊**：大頭照 + 職稱 + CTA 按鈕
- **核心價值**：四張卡片呈現（創新驅動、品質至上、高效交付、持續成長）
- **專業服務**：架構設計、全端開發、團隊管理、技術顧問四大區塊
- **數據統計**：用 `CounterAnimation` 元件做數字跳動動畫（年資、專案數、團隊人數）
- **聯絡表單**：`ContactForm` 元件，整合了表單驗證

光 Home.vue 就寫了 1500 多行，說實話有點肥，但內容就是多，暫時沒想要拆更細。

### 4. 技術架構展示頁（Mermaid 圖表）

這個頁面是我個人覺得最有意思的。用 Mermaid.js 畫了四張架構圖：

- **微服務架構**：Azure API Management + App Service + Redis + SQL Server
- **物聯網架構**：AWS Lambda + Step Functions + API Gateway + S3
- **SSO 單一登入**：OAuth 2.0 + JWT 的認證流程
- **高併發系統**：CDN → Load Balancer → Redis → App Service → Message Queue → DB

這些都是我這十幾年實際做過的架構，不是隨便畫的示意圖。用 Mermaid 的好處是可以直接寫在 Vue 裡面，不用另外存圖片檔，改起來也方便。

### 5. 作品集頁面

做了分類展示：互動小遊戲、形象網頁、開源工具、UI 元件、全端專案。每個項目有狀態標籤（上線中 / 即將推出 / 開發中），用不同顏色的 Badge 標示。

老實說目前上線的只有這個網站本身，其他都還是 "Coming Soon" 的狀態。但框架先搭好，後面陸續做完就放上去。

### 6. SEO 處理

加了幾個基本的 SEO 措施：

- `robots.txt` 跟 `sitemap.xml`
- 每個路由的 `meta`（title、description、keywords）
- structuredData.js 裡面放了 JSON-LD 的 Schema.org 標記（Person、ProfessionalService、WebSite）

不求排名多前面，但至少 Google 搜得到。

### 7. GitHub Actions 自動部署

設了 GitHub Workflow，push 到 master 就自動 build 然後部署到 GitHub Pages。不用再本地打包完手動推上去了。

## 開發過程中踩到的坑

### Vite 衝突問題

Quasar 底下用的是 `@quasar/app-vite`，在某些套件版本組合下會出衝突。我記得花了一些時間在 debug 這個，最後是調整 Vite 相關的設定才解決的。

### i18n 巢狀物件的取值

`vue-i18n` 要取巢狀的翻譯物件（不是單純字串，是整個陣列或物件），不能用 `$t()`，要用 `tm()`（Translation Message）。像作品集的資料結構比較深：

```js
const sections = computed(() => {
  const s = tm('portfolioPage.sections')
  return s
})
```

這個搞了一下才弄懂，文件沒寫得很清楚。

### 舊檔案清理

舊版打包的檔案（`js/`、`css/`、`fonts/`、`demo/`、`profolio/`）全部要清掉，改成新的 Quasar 專案結構。那個 diff 看起來改了一百多個檔案，刪了兩萬多行，加了一萬六千多行，看起來很嚇人但其實主要是在移除舊的打包產物。

## 回頭看這次改版

從 2021 年的版本到現在，等於是從「把打包好的靜態檔丟上去」變成「一個有工程化結構的 Vue 專案」。該有的都補上了：

- 元件化架構（Vue 3 + Composition API）
- UI 框架（Quasar 2）
- 多語系（vue-i18n，中 / 英）
- 深色模式
- RWD 響應式設計
- SEO 基本功（meta、sitemap、structured data）
- CI/CD（GitHub Actions）
- 狀態管理（Pinia）
- 路由管理（Vue Router）

其實這些對一個個人網站來說有點 overkill，但既然要做就做完整。一方面是自己用起來方便，另一方面如果有人來看我的 GitHub，至少看到的是一個像樣的專案，而不是一堆散落的 HTML 檔。

隔了這麼久才更新，就是因為自己的東西永遠排在最後面。但做完之後其實蠻有成就感的，至少不用再心虛了。

下一步就是把作品集裡那些 "Coming Soon" 的東西一個一個做出來，希望不要再等五年。