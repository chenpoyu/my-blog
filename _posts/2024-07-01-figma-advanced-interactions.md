---
layout: post
title: "Figma 進階互動：Drag、Variables 與複雜動畫"
date: 2024-07-01 16:55:00 +0800
categories: [設計, UI/UX]
tags: [Figma, Interaction, Variables]
---

前幾週學了基本的 Prototype 和 Smart Animate，這週來挑戰更進階的互動：拖曳（Drag）、變數（Variables）、以及一些複雜的互動模式。

這些功能可以讓 prototype 更接近真實產品的體驗。

## Drag 互動

Figma 支援拖曳互動，可以做 Slider、Swipe、拖動排序等效果。

### Horizontal Drag（橫向拖曳）

做一個音量滑桿（Slider）。

**設定**：

1. 畫一個滑軌（灰色長條）
2. 畫一個滑塊（圓形，藍色）
3. 選中滑塊，切換到 Prototype tab
4. Trigger: `On drag`
5. Action: `None`（不跳頁，只是拖動）
6. Overflow behavior: `Horizontal constraint`
7. 設定拖動範圍（Min: 0, Max: 200）

預覽，可以左右拖動滑塊，但不會超出設定的範圍。

### Vertical Drag（垂直拖曳）

做一個下拉重新整理（Pull to refresh）。

**設定**：

1. 畫一個內容區域
2. 畫一個 loading 圖示在頂部（Y: -50，一開始看不到）
3. 選中內容區域，On drag → None
4. Overflow behavior: `Vertical constraint`
5. 限制往下拖（Min: -100）

拖動時 loading 圖示會出現。

實際產品還要偵測拖動距離觸發重新整理，但 Figma prototype 做不到真正的邏輯判斷，只能模擬視覺效果。

### 卡片滑動（Swipe）

做一個左右滑動切換卡片的效果（類似 Tinder）。

**準備三個 Frame**：

- Frame 1: 卡片在中間
- Frame 2: 卡片往右滑出（X: 400）
- Frame 3: 卡片往左滑出（X: -400）

**設定互動**：

Frame 1 的卡片：
- On drag → 
  - Drag right → Navigate to Frame 2, Smart animate
  - Drag left → Navigate to Frame 3, Smart animate

但 Figma 的 Drag 沒有方向判斷，只能用 `While dragging` 模擬。

另一種做法是用 Component variant：

- Variant 1: Default（中間）
- Variant 2: Swiped Right（右邊）
- Variant 3: Swiped Left（左邊）

然後用手勢觸發切換（雖然不完美）。

## Variables（變數）

Variables 是 Figma 的新功能（2023 年推出），可以儲存和管理資料，讓 prototype 更動態。

### 建立 Variables

1. 右側面板切換到 `Local variables`
2. 點 `Create variable`
3. 命名：比如 `Counter`
4. Type: Number（其他類型：String, Boolean, Color）
5. 設定初始值：0

### 使用 Variables

做一個計數器：顯示數字，點按鈕會加 1。

**設定**：

1. 畫一個文字顯示數字
2. 選中文字，在內容輸入：`{Counter}`（用大括號引用變數）
3. 畫一個「+1」按鈕
4. 設定互動：
   - On click
   - Action: `Set variable`
   - Variable: Counter
   - Value: `Counter + 1`（表達式）

預覽，點按鈕，數字會增加。

### Variable Modes

Variables 可以有多個 Mode，類似主題切換。

比如建立一個 Color variable：`Primary Color`

Modes:
- Light mode: `#007AFF`
- Dark mode: `#0A84FF`

切換 Mode，所有用到 `Primary Color` 的元素都會改變。

用途：
- 主題切換（Light/Dark）
- 語言切換（中文/英文）
- 品牌變體（Brand A/Brand B）

### 條件顯示

利用 Boolean variable 控制元素顯示/隱藏。

建立變數 `ShowDetails` (Boolean, default: false)

有個「詳細資訊」區域，設定：
- Visible when: `ShowDetails == true`

按鈕：
- On click → Set variable `ShowDetails` = `!ShowDetails`（切換）

點按鈕，詳細資訊會顯示/隱藏。

## 實戰：購物車

做一個有互動的購物車 prototype。

**需求**：
- 顯示商品列表
- 每個商品可以調整數量（+/-）
- 顯示總價
- 數量 = 0 時商品消失

**設定 Variables**：

- `Item1_Quantity` (Number, 1)
- `Item2_Quantity` (Number, 1)
- `Item3_Quantity` (Number, 2)
- `TotalPrice` (Number, 計算公式)

**商品 1**：

- 名稱：商品 A
- 單價：100
- 數量顯示：`{Item1_Quantity}`
- 小計：`{Item1_Quantity * 100}`

按鈕：
- 「+」: Set variable `Item1_Quantity` = `Item1_Quantity + 1`
- 「-」: Set variable `Item1_Quantity` = `max(0, Item1_Quantity - 1)`

**總價**：

`{Item1_Quantity * 100 + Item2_Quantity * 150 + Item3_Quantity * 200}`

預覽，可以調整數量，總價會自動更新。

（限制：商品消失的邏輯 Figma 做不到，需要手動建立不同的 Frame 模擬）

## Mouse events

除了 Click 和 Hover，還有更細緻的滑鼠事件。

### Mouse Enter / Mouse Leave

滑鼠移入/移出時觸發，可以做懸停效果。

**圖片卡片**：
- Default: 無遮罩
- Hover: 半透明黑色遮罩 + 文字

設定：
- On mouse enter → Change to Hover variant, Smart animate
- On mouse leave → Change to Default variant, Smart animate

### Mouse Down / Mouse Up

滑鼠按下/放開，可以做按壓效果。

**按鈕**：
- On mouse down → Scale 95%, Smart animate, 50ms
- On mouse up → Scale 100%, Smart animate, 50ms

### While Hovering / While Pressing

持續觸發，適合做持續變化的效果。

**放大鏡效果**：
- While hovering → 圖片放大 110%

## 實戰：影片播放器

做一個簡單的影片播放器 prototype（只是 UI，不是真的影片）。

**Component Variants**：

- `State = Paused`：顯示播放按鈕
- `State = Playing`：顯示暫停按鈕，進度條會動

**Variables**：

- `IsPlaying` (Boolean, false)
- `Progress` (Number, 0) - 進度百分比

**互動**：

播放按鈕：
- On click
  - Set variable `IsPlaying` = true
  - Change to `State = Playing`

暫停按鈕：
- On click
  - Set variable `IsPlaying` = false
  - Change to `State = Paused`

進度條：
- Width: `{Progress}%`

播放時，可以用 After delay 模擬進度增加：
- After delay 1s → Set variable `Progress` = `Progress + 10`
- 循環到 100%

（實際上這樣做很複雜，需要很多 Frame，Figma 不太適合做這種真實的邏輯）

## Key / Gamepad

可以用鍵盤或遊戲手把觸發互動。

比如做一個簡單遊戲：
- 按方向鍵移動角色
- 按空白鍵跳躍

設定：
- Key: Arrow Right → Move character right
- Key: Arrow Left → Move character left
- Key: Space → Jump (Change to jump variant)

遊戲類 prototype 可以用這個功能。

## 組合互動

複雜的互動可能需要組合多個觸發。

**長按刪除**：

- While pressing (hold 1s) → Show delete confirm
- On click (quick tap) → Normal action

Figma 可以設定 Press duration。

**雙擊編輯**：

Figma 沒有內建雙擊，但可以用快速連續兩次 On click 模擬（不完美）。

## 限制與替代方案

Figma prototype 的限制：

**沒有真正的程式邏輯**：只能用 Variables 做簡單的計算和條件。

**無法處理真實資料**：不能連 API、資料庫。

**複雜互動很費工**：需要建立很多 Frame 和連線。

**效能有限**：太複雜的 prototype 會卡頓。

如果需要更真實的 prototype，可以考慮：

**ProtoPie**：更強的互動邏輯和感測器支援。

**Framer**：用 React 寫 prototype，完全的程式邏輯。

**直接寫 code**：用 React/Vue 做真實的 prototype。

但 Figma 的優勢是快速、直覺、協作方便，對大部分情況夠用。

## 心得

進階互動功能讓 Figma prototype 能做更多事情，但也發現極限在哪。

Variables 很實用，可以做一些簡單的動態效果。但複雜的邏輯（比如表單驗證、資料處理）就力不從心了。

Drag 互動蠻有趣，但設定起來有點麻煩，而且效果不如真實 app 流暢。

整體來說，Figma 適合做「高保真度的視覺原型」，讓大家看到大致的樣子和流程。真正的互動細節和邏輯，還是要等實際開發。

下週想整理一下這段時間學到的東西，做一個完整的 App prototype，把 Component、Prototype、Smart Animate、Variables 都整合起來，看能做出什麼樣的成果。
