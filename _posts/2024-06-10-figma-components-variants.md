---
layout: post
title: "Figma Components 與 Variants：建立設計系統"
date: 2024-06-10 15:30:00 +0800
categories: [設計, UI/UX]
tags: [Figma, Components, Design System]
---

上週做了簡單的登入畫面，這週來研究 Figma 的 Component。如果只是做單一畫面，不太需要 Component；但要做完整的 prototype，有很多重複的元素（按鈕、輸入框、卡片），每次都重畫太沒效率。

Component 就像程式裡的 function，定義一次，到處使用。

## Component 的概念

**Component（元件）**：可重用的設計元素。建立 Component 後，可以產生多個 Instance（實例）。

**Instance（實例）**：Component 的副本。修改主 Component，所有 Instance 會同步更新。

**Variant（變體）**：同一個 Component 的不同狀態或樣式。比如按鈕有 Primary、Secondary、Disabled 等狀態。

概念類似這樣：

```javascript
// Component 定義
function Button({ variant, text }) {
  return <button className={variant}>{text}</button>
}

// Instance 使用
<Button variant="primary" text="登入" />
<Button variant="secondary" text="取消" />
```

## 建立第一個 Component

來做一個按鈕 Component。

**Step 1：設計按鈕**

畫一個矩形（按鈕背景）：
- Width: 120
- Height: 40
- Fill: `#007AFF`
- Corner Radius: 8

加上文字「按鈕」，白色，置中。

選中矩形和文字，用 Auto Layout（`Shift+A`）包起來。調整 Padding：左右 24、上下 12。

**Step 2：建立 Component**

選中整個按鈕，按右鍵 → `Create component`（或快捷鍵 `Cmd+Option+K`）。

會看到圖層名稱前面出現紫色菱形圖示，代表這是 Component。

重新命名為 `Button/Primary`（用 `/` 可以建立階層結構）。

**Step 3：使用 Instance**

選中 Component，`Cmd+C` 複製，`Cmd+V` 貼上。

貼上的會是 Instance（圖層名稱前面是空心菱形）。

修改 Instance 的文字為「登入」。文字可以改，但樣式會跟著主 Component。

試著修改主 Component 的顏色為綠色，會看到所有 Instance 都變綠了。

## Variant 變體

按鈕通常有多種狀態：Primary、Secondary、Disabled 等。用 Variant 可以優雅地管理。

**Step 1：建立第一個 Variant**

選中 Button Component，右鍵 → `Add variant`。

會複製一個一模一樣的，兩個並排在一起。

**Step 2：修改不同狀態**

第一個保持藍色背景，命名 Property 為 `Type = Primary`。

第二個改成白色背景、藍色邊框、藍色文字，命名為 `Type = Secondary`。

再加一個 Variant，灰色背景、灰色文字，命名為 `Type = Disabled`。

**Step 3：使用 Variant**

拖一個 Button Instance 到畫布上。

在右側 Design 面板會看到 `Type` 下拉選單，可以選擇 Primary、Secondary、Disabled。

切換不同的 Type，按鈕外觀會自動改變。

## 多屬性 Variant

按鈕還可以有不同的大小：Small、Medium、Large。

選中 Button Component，在右側 `Variants` 面板點 `+` 新增屬性。

命名為 `Size`，值為 `Small`、`Medium`、`Large`。

Figma 會自動產生 3x3 = 9 個組合（3 種 Type x 3 種 Size）。

分別調整不同 Size 的：
- Small: Height 32, Font size 14, Padding 8/16
- Medium: Height 40, Font size 16, Padding 12/24
- Large: Height 48, Font size 18, Padding 16/32

使用時，可以在右側同時選擇 Type 和 Size。

## Boolean 屬性

有些屬性是 True/False，比如 Icon（有沒有圖示）。

新增屬性 `Icon`，類型選 `Boolean`。

在 `Icon = True` 的 Variant，加上一個圖示（可以從 Figma 的 Icon 外掛找）。

使用時勾選 `Icon`，按鈕就會顯示圖示。

## 覆寫與限制

Instance 可以覆寫某些屬性：
- **可以改**：文字內容、顏色、大小
- **不能改**：圓角、陰影、間距（這些跟著 Component）

如果想讓某個屬性不能覆寫，在 Component 選中該圖層，右鍵 → `Instance options` → 取消勾選 `Content`。

## Nested Components

Component 可以包含其他 Component（巢狀）。

比如做一個 Card Component，裡面包含：
- Image Component
- Text/Title Component
- Button Component

修改內部的 Button Component，所有用到它的 Card 都會更新。

實際操作：

**Step 1：建立 Card**

畫一個 Frame（`F`），320x200。

加上：
- 圓角矩形當圖片占位（灰色，100x100）
- 標題文字「卡片標題」
- 內容文字「這是卡片內容...」
- 之前做的 Button Instance

用 Auto Layout 排版，設定間距和 Padding。

**Step 2：建立 Component**

選中整個 Card，`Cmd+Option+K` 建立 Component。

命名為 `Card/Default`。

**Step 3：使用**

拖一個 Card Instance。可以改文字內容，Button 也可以切換 Variant。

如果改了主 Button Component 的顏色，所有 Card 裡的按鈕都會跟著變。

## Component Library

可以把常用的 Component 整理成 Library，在不同檔案裡使用。

**建立 Library**：

1. 建立一個新檔案，命名為 `Design System`
2. 把所有 Component 放進去（Button、Input、Card 等）
3. 點右上角 `Share` → `Publish components`
4. 寫個更新說明，點 `Publish`

**使用 Library**：

1. 在其他檔案，點工具列的 `Assets` 面板（左上角書本圖示）
2. 點書本圖示旁的下拉選單 → `Enable libraries`
3. 勾選 `Design System`
4. 在 Assets 面板就能看到所有 Component，直接拖到畫布上

Library 更新後，使用的檔案會收到通知，可以選擇更新到最新版本。

## 實戰：建立一套按鈕系統

整理一下需求：

- **Type**: Primary, Secondary, Outline, Text
- **Size**: Small, Medium, Large
- **State**: Default, Hover, Pressed, Disabled
- **Icon**: True, False

如果每個組合都做，會有 4x3x4x2 = 96 個 Variant，太多了。

實務上可以簡化：State 用 Prototype 的互動處理，不用做成 Variant。

所以只需要：4 Type x 3 Size x 2 Icon = 24 個 Variant。

建立一個 `Button` Component，新增屬性：
- Type: Primary, Secondary, Outline, Text
- Size: Small, Medium, Large
- Icon: Boolean

逐一設計 24 種組合。

完成後，所有畫面都能用這套按鈕，保持一致性。

## 顏色與樣式管理

Figma 可以定義 Color Styles 和 Text Styles，類似 CSS 變數。

**Color Styles**：

1. 選中一個有顏色的物件
2. 右側 Fill 點 `Style` 圖示（四個點）
3. 點 `+` 建立新 Style
4. 命名：`Primary/Blue`、`Neutral/Gray-100` 等

之後選顏色就選 Style，不用每次輸入色碼。

改 Style 的顏色，所有用到的地方都會更新。

**Text Styles**：

1. 選中文字
2. 右側 Text 區域點 Style 圖示
3. 建立 Style：`Heading/H1`、`Body/Regular`、`Caption` 等

定義好字體、大小、行高、字重等。

## Auto Layout 進階

Auto Layout 可以設定：

**Resizing**：
- `Hug contents`：大小根據內容調整（類似 `auto`）
- `Fixed`：固定大小
- `Fill container`：填滿父容器（類似 `width: 100%`）

**Alignment**：
- 水平對齊：左、中、右
- 垂直對齊：上、中、下

**Spacing**：
- `Space between`：平均分配空間
- `Packed`：緊密排列

按鈕的 Auto Layout 設定：
- Direction: Horizontal
- Padding: 12px 24px
- Spacing: 8px（文字和圖示之間）
- Resizing: Hug contents

這樣文字長短改變，按鈕會自動調整寬度。

## 實用技巧

**快速建立多個 Variant**：按住 `Option` 拖曳 Variant，可以快速複製。

**批量修改**：選中多個 Variant，一次修改共同屬性（如圓角、陰影）。

**組織結構**：用 `/` 建立階層，比如 `Button/Primary/Large`。在 Assets 面板會顯示成資料夾結構。

**描述文件**：Component 可以加 Description，說明用法和注意事項。在 Assets 面板會顯示。

## 心得

Component 和 Variant 是 Figma 的核心功能，搞懂這個才能有效率地設計。

一開始可能會覺得建立 Component 很麻煩，直接複製貼上比較快。但當專案變大，有幾十個畫面，要改一個按鈕顏色，用 Component 只要改一個地方；不用 Component 就要改幾十個地方，容易漏掉。

Component 的思維跟寫程式很像，都是「不要重複」、「可維護性」。對工程師來說應該不難理解。

下週想研究 Figma 的 Prototype 功能，做一些互動效果，讓設計不只是靜態圖片，而是可以點擊操作的原型。
