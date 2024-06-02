---
layout: post
title: "Figma 初體驗：從零開始的設計工具"
date: 2024-06-03 10:25:00 +0800
categories: [設計, UI/UX]
tags: [Figma, Design, Prototype]
---

最近手上有個專案要做 prototype，想說趁這個機會學一下 Figma。以前都是工程師做工程師的事，設計交給設計師，但這次想自己動手試試看，了解一下設計師的世界。

## 為什麼選 Figma

市面上設計工具不少，Sketch、Adobe XD、Figma 都有人用。選 Figma 的原因很簡單：

**網頁版**：不用裝軟體，開瀏覽器就能用。Mac、Windows、Linux 都可以。

**協作方便**：就像 Google Docs 一樣，可以多人同時編輯，看到彼此的游標。

**免費版夠用**：個人使用免費版就很夠了，可以建立無限個檔案。

**社群資源多**：有很多現成的 UI Kit、Icon、Template 可以用。

**學習曲線平緩**：對工程師來說，Figma 的邏輯蠻直覺的。

## 註冊與介面

到 figma.com 註冊帳號，可以用 Google 登入。進去後會看到 Dashboard，類似檔案總管的概念。

點 `New design file` 建立一個新檔案。

介面分成幾個區域：

**左側 Layers**：顯示所有圖層，概念類似 Photoshop 的圖層面板。

**中間畫布**：主要工作區域，可以無限延伸。

**右側 Design**：屬性面板，調整選中物件的樣式、位置、大小等。

**頂部工具列**：各種繪圖工具、形狀工具、文字工具。

## 基本概念

Figma 有幾個核心概念要先搞懂：

**Frame**：類似畫板或容器，通常一個 Frame 代表一個畫面（手機畫面、網頁等）。

**Layer**：任何你放在畫布上的東西都是 Layer，包括形狀、文字、圖片。

**Group**：把多個 Layer 群組起來，方便管理。

**Component**：可重用的元件，類似程式的 function。

**Auto Layout**：自動排版功能，類似 CSS Flexbox。

## 第一個設計

來做個簡單的登入畫面練習。

**Step 1：建立 Frame**

按快捷鍵 `F`（或點工具列的 Frame 圖示），選擇 `iPhone 14 Pro`。會出現一個手機大小的 Frame。

**Step 2：加上背景**

選中 Frame，在右側 Design 面板的 Fill 設定顏色，比如淺灰色 `#F5F5F5`。

**Step 3：畫一個卡片**

按 `R` 建立矩形（Rectangle），拖出一個大約 320x400 的矩形。

在右側調整：
- Fill: 白色 `#FFFFFF`
- Corner Radius: 16（圓角）
- Drop shadow: 加上陰影（Effects → Drop shadow）

**Step 4：加上文字**

按 `T` 建立文字（Text），輸入「登入」。

調整：
- Font: 選一個好看的字體（我用 Inter）
- Size: 24
- Weight: Bold
- Fill: 深色 `#333333`

**Step 5：輸入框**

再畫一個矩形，當作輸入框：
- Width: 280
- Height: 48
- Fill: 淺灰 `#F0F0F0`
- Corner Radius: 8
- Stroke: 1px，淺灰色（可選）

加上 placeholder 文字「請輸入帳號」，顏色用灰色。

複製一個做密碼欄位（`Cmd+D`）。

**Step 6：按鈕**

畫一個矩形當按鈕：
- Width: 280
- Height: 48
- Fill: 藍色 `#007AFF`
- Corner Radius: 8

加上白色文字「登入」。

## 對齊與排版

Figma 有很好用的對齊工具。

選中多個物件（按住 Shift 點選），在頂部會出現對齊選項：
- Align left/center/right
- Align top/middle/bottom
- Distribute horizontally/vertically

或者用快捷鍵：
- `Alt+A`：對齊左邊
- `Alt+H`：水平置中
- `Alt+V`：垂直置中

## Auto Layout 初探

Auto Layout 是 Figma 的殺手功能，可以自動調整間距和排版。

選中輸入框和按鈕，按 `Shift+A` 建立 Auto Layout。

會看到它們自動排成一列（或一欄）。在右側可以調整：
- Direction: Horizontal（橫）或 Vertical（直）
- Spacing: 物件之間的間距
- Padding: 內邊距

改變間距，所有元素會自動調整，不用手動移動，超方便。

## 圖層命名

養成好習慣，圖層要命名。預設的 `Rectangle 1`、`Text 2` 看了很亂。

雙擊圖層名稱可以重新命名：
- `Background`
- `Login Card`
- `Title`
- `Input - Email`
- `Input - Password`
- `Button - Login`

之後要找東西方便很多。

## 快捷鍵

Figma 的快捷鍵很直覺，記幾個常用的：

**工具**：
- `V`：選取工具（Move）
- `F`：Frame
- `R`：矩形
- `O`：圓形
- `T`：文字
- `P`：筆刷（Pen）

**操作**：
- `Cmd+D`：複製並貼上（原位置）
- `Cmd+G`：群組
- `Cmd+Shift+G`：解散群組
- `Cmd+[` / `Cmd+]`：圖層往下/上移
- `Cmd+Shift+[` / `Cmd+Shift+]`：圖層移到最下/最上

**檢視**：
- `Cmd+0`：縮放到 100%
- `Cmd+1`：縮放到適合畫面
- `Space+拖曳`：移動畫布
- `Cmd+滾輪`：縮放

## 存檔與分享

Figma 是自動存檔的，不用按 Save。每次編輯都會即時同步。

要分享給別人看：
1. 點右上角 `Share`
2. 可以邀請特定人，或產生分享連結
3. 權限分成 `Can view`（只能看）和 `Can edit`（可編輯）

別人打開連結就能看到你的設計，還能加註解。

## 版本歷史

Figma 會自動保存版本歷史。

點選檔案名稱旁的下拉選單 → `Show version history`，可以看到所有修改記錄。

如果改壞了，可以回到之前的版本。也可以手動儲存重要的版本：`Save to version history`。

## 初步心得

第一次用 Figma，感覺蠻上手的。介面簡潔，工具不複雜，基本的矩形、圓形、文字就能做出不錯的畫面。

跟寫 code 比起來，設計是完全不同的思維。要考慮視覺平衡、顏色搭配、間距是否協調等。不過對工程師來說，Figma 的邏輯還算清楚，圖層就像 DOM 樹，Auto Layout 就像 Flexbox。

下週想研究 Component 和 Variant，看怎麼建立可重用的元件，這樣設計系統會更有效率。也想試試 Figma 的 Prototype 功能，畢竟這是我學 Figma 的主要目的。
