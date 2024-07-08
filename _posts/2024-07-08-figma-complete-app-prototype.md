---
layout: post
title: "Figma 實戰：打造完整的 App Prototype"
date: 2024-07-08 11:20:00 +0800
categories: [設計, UI/UX]
tags: [Figma, Prototype, 實戰專案]
---

學了五週 Figma，這週來做一個完整的 prototype，把學到的東西整合起來。

專案是一個待辦事項 App（Todo App），功能包括：
- 顯示任務列表
- 新增/編輯/刪除任務
- 標記完成
- 篩選（全部/進行中/已完成）
- 深色模式切換

## 規劃與架構

做之前先規劃一下：

**畫面（Frames）**：
1. 首頁（任務列表）
2. 新增任務
3. 編輯任務
4. 任務詳情

**Components**：
- Task Item（任務項目）
- Button（按鈕）
- Input（輸入框）
- Tab Bar（篩選）
- Header（標題列）

**Variables**：
- `TaskCount`：任務數量
- `DarkMode`：深色模式開關

**互動**：
- 點任務 → 任務詳情
- 勾選 → 標記完成，動畫
- 左滑 → 刪除
- 點「+」→ 新增任務
- 切換篩選
- 切換深色模式

## 建立 Design System

先建立可重用的元件。

### Colors（Color Variables）

建立 Color Variables，支援 Light/Dark 兩種 Mode：

| Variable | Light Mode | Dark Mode |
|----------|------------|-----------|
| Background | #FFFFFF | #1C1C1E |
| Surface | #F2F2F7 | #2C2C2E |
| Primary | #007AFF | #0A84FF |
| Text/Primary | #000000 | #FFFFFF |
| Text/Secondary | #8E8E93 | #8E8E93 |
| Border | #E5E5EA | #38383A |

### Text Styles

- Heading/Large: 28px, Bold
- Heading/Medium: 20px, Semibold
- Body/Regular: 16px, Regular
- Body/Small: 14px, Regular
- Caption: 12px, Regular

### Button Component

建立 Button Component，Variants：

**Type**：
- Primary（藍底白字）
- Secondary（白底藍字）
- Danger（紅底白字）
- Text（無背景）

**Size**：
- Small（32px height）
- Medium（40px height）
- Large（48px height）

**State**：
- Default
- Disabled（灰色，透明度 50%）

用 Auto Layout 包裝，設定：
- Padding: 8px 16px（Small）, 12px 24px（Medium）, 16px 32px（Large）
- Gap: 8px（文字和圖示間距）
- Corner radius: 8px

### Input Component

輸入框 Component：

**State**：
- Default
- Focused（藍色邊框）
- Error（紅色邊框）

Auto Layout：
- Padding: 12px 16px
- Height: 48px
- Corner radius: 8px
- Border: 1px

### Task Item Component

任務項目是核心元件。

**結構**：
- Checkbox（圓形，未完成；打勾，已完成）
- Title（任務標題）
- Time（建立時間）
- Status Tag（選填，如「緊急」）

**Variants**：

`Status`：
- Todo（未完成，白色背景）
- Done（已完成，灰色背景，文字刪除線）

Auto Layout：
- Direction: Horizontal
- Padding: 16px
- Gap: 12px
- Fill: Surface color

## 首頁設計

### Header

- 左邊：標題「待辦事項」
- 右邊：深色模式切換按鈕、設定按鈕

用 Auto Layout，Fill width。

### Tab Bar（篩選）

三個 Tab：全部、進行中、已完成

做成 Component，Variant = Tab1/Tab2/Tab3

Active 的 Tab 有藍色底線（用 Frame + 藍色 Fill，高度 2px）。

### Task List

用 Auto Layout，Direction: Vertical，Gap: 8px。

放 3-4 個 Task Item instances：
- 買菜（Todo）
- 寫報告（Todo）
- 運動（Done）
- 讀書（Todo）

### FAB（浮動按鈕）

右下角圓形藍色按鈕，「+」圖示。

設定 Fixed position when scrolling。

## 新增任務畫面

從底部滑出的 Sheet（或另一個 Frame）。

**內容**：
- 標題「新增任務」
- 輸入框（任務標題）
- 輸入框（備註，可選）
- 日期選擇（簡化，用假資料）
- 優先順序（低/中/高）
- 按鈕：取消、新增

## Prototype 互動

開始連接互動。

### 首頁 → 新增任務

FAB 按鈕：
- On click → Open overlay → New Task Frame
- Position: Bottom
- Animation: Move in from Bottom, 300ms

New Task 的「取消」：
- On click → Close overlay

New Task 的「新增」：
- On click → Close overlay
- （實際 app 會新增到列表，prototype 就省略）

### 任務完成動畫

Task Item Component：

在 Todo variant，Checkbox：
- On click → Change to Done variant, Smart animate, 300ms

在 Done variant，Checkbox：
- On click → Change to Todo variant, Smart animate, 300ms

Smart Animate 會自動處理：
- 背景顏色變化（白 → 灰）
- Checkbox 變化（空 → 勾）
- 文字刪除線出現

### 任務詳情

Task Item：
- On click（點任務本身，不是 checkbox）→ Navigate to Task Detail Frame
- Animation: Move in from Right, 300ms

Task Detail 的返回按鈕：
- On click → Back, Move out to Right, 300ms

### 左滑刪除

Task Item：
- On drag left → 顯示刪除按鈕（紅色，垃圾桶圖示）
- 可以用 Component variant 或另一個 Frame

Drag 設定：
- Horizontal constraint
- Min: -80（左滑 80px 顯示刪除按鈕）

刪除按鈕：
- On click → Task Item 消失（用 Smart Animate, Opacity 0）

### Tab 切換

Tab Bar Component：

Tab1（全部）：
- On click → Change to Tab1 variant, Smart animate

Tab2（進行中）：
- On click → Change to Tab2 variant, Smart animate
  - （同時切換到另一個 Frame，只顯示未完成任務）

Tab3（已完成）：
- On click → Change to Tab3 variant, Smart animate
  - （切換到只顯示已完成任務的 Frame）

下方的藍色指示器會滑動到對應 Tab（Smart Animate 自動補間）。

### 深色模式切換

Header 的切換按鈕（月亮/太陽圖示）：

- On click → Set variable `DarkMode` = `!DarkMode`
- 切換 Color Variables Mode（Light ↔ Dark）

所有使用 Color Variables 的元素會自動變色。

實際操作：
1. 選中整個 Frame
2. 在 Variables 面板選擇 Mode
3. 所有顏色會切換

但在 Prototype 動態切換比較複雜，可以簡化：

準備兩個版本的首頁（Light / Dark），切換按鈕：
- On click → Navigate to Dark Mode Frame, Smart animate, 500ms

Smart Animate 會處理所有顏色變化，很流暢。

## 加入微互動

一些小細節讓 prototype 更生動。

**按鈕按下效果**：
- On mouse down → Scale 95%, Smart animate, 50ms
- On mouse up → Scale 100%, Smart animate, 50ms

**Checkbox 動畫**：
- 勾選時，Checkbox 有個放大再縮小的彈跳效果
- 用 Scale（100% → 110% → 100%），多個 Frame 模擬

**任務新增成功**：
- 顯示 Toast 通知「任務已新增」
- 從頂部滑下，停留 2 秒，自動消失
- 用 After delay 觸發

**Loading 狀態**：
- 新增任務按下後，顯示 Loading（轉圈圈）
- After delay 1s → 關閉 Sheet

**空狀態**：
- 如果沒有任務，顯示空狀態插圖和文字「還沒有任務，點擊 + 新增」

## 組織與命名

專案變大，要整理好結構。

**Page 分類**：
- Design System（Components、Colors、Text Styles）
- Screens（所有畫面）
- Flows（Prototype 流程）

**Frame 命名**：
- `Home - All Tasks`
- `Home - In Progress`
- `Home - Completed`
- `Sheet - New Task`
- `Detail - Task`

**Component 命名**：
- `Button/Primary/Medium`
- `Input/Default`
- `Task/Todo`
- `Task/Done`

**Layer 命名**：
- 避免 `Rectangle 123`、`Group 45`
- 改成 `Background`、`Title`、`Icon`

## 測試與調整

完成後，仔細測試每個互動：

**流程測試**：
- 能否順利新增任務？
- 標記完成是否流暢？
- 篩選功能正常嗎？
- 返回按鈕正確嗎？

**動畫測試**：
- 速度是否合適？（太快太慢都不好）
- Easing 是否自然？
- 有沒有跳動或閃爍？

**視覺測試**：
- 間距是否一致？
- 對齊是否正確？
- 字體大小合適嗎？
- 顏色對比度夠嗎？

**裝置測試**：
- 在不同尺寸的 Frame 測試（iPhone、iPad）
- 確保 Auto Layout 正常運作

## 分享與反饋

完成後分享給別人試用：

1. 點 `Share` → `Get prototype link`
2. 勾選 `Show hotspot hints`（顯示可點擊區域）
3. 寄給團隊或朋友

收集反饋：
- 流程是否順暢？
- 有哪裡不清楚？
- 互動是否符合預期？
- 有沒有遺漏的功能？

根據反饋調整。

## 遇到的挑戰

**Variables 限制**：本來想用 Variables 動態管理任務列表，發現做不到。Figma 的 Variables 不能處理陣列和複雜物件，只能用多個 Frame 模擬不同狀態。

**左滑刪除**：Drag 互動不太好控制，需要多試幾次才能調整到理想的感覺。而且預覽時有時會卡卡的。

**深色模式切換**：動態切換 Variable Mode 在 Prototype 不太穩定，改用兩個 Frame 切換比較可靠。

**複雜度管理**：Prototype 連線很多，看起來像義大利麵，要花時間整理。用 Flow 功能分不同流程會清楚一些。

## 成果與心得

做完這個 prototype，對 Figma 的理解更深了。

**Component 真的很重要**：一開始花時間建立 Design System，後面改起來很輕鬆。改一個 Button Component，所有按鈕都更新。

**Smart Animate 很強大**：只要規劃好圖層名稱，動畫就會很流暢。標記完成的動畫、深色模式切換，都是 Smart Animate 自動處理。

**Prototype 的極限**：複雜的邏輯（表單驗證、資料處理、狀態管理）Figma 做不到。但對於展示設計概念、測試使用者流程，已經很夠用。

**與開發的差異**：Prototype 只是模擬，很多細節（錯誤處理、Loading、邊界情況）都簡化了。實際開發要考慮更多。

但 Prototype 的價值在於**快速驗證想法**，不用寫 code 就能看到效果，發現問題可以馬上改。比起先開發再修改，效率高很多。

這幾週學 Figma 的收穫不少。雖然我主要是工程師，但了解設計工具後，跟設計師溝通更順暢，也能更好地理解設計的思維。而且有時候要做個簡單的 mockup，自己就能搞定，不用麻煩別人。

Figma 是個值得學的工具，推薦給其他工程師試試看。
