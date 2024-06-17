---
layout: post
title: "Figma Prototype 基礎：讓設計動起來"
date: 2024-06-17 09:40:00 +0800
categories: [設計, UI/UX]
tags: [Figma, Prototype, Interaction]
---

前兩週學了 Figma 的基礎和 Component，這週終於要進入重點：Prototype。

做 prototype 就是我學 Figma 的主要目的。靜態設計圖只能看，但 prototype 可以實際操作，點按鈕會跳到下一頁、輸入表單、看到動畫效果等。這樣可以在開發前就測試使用者流程，發現問題。

## Prototype 模式

Figma 右側有三個 tab：Design、Prototype、Inspect。

切換到 **Prototype** tab，介面會變成專門做互動的模式。

## 第一個互動：頁面跳轉

來做一個簡單的流程：登入頁 → 首頁。

**準備兩個 Frame**：

1. Frame 1：登入畫面（之前做的）
2. Frame 2：首頁（隨便放一些內容）

**建立互動**：

1. 切換到 Prototype tab
2. 選中登入畫面的「登入」按鈕
3. 會看到按鈕右側出現一個小圓點
4. 從圓點拖曳一條線到首頁 Frame
5. 會出現互動設定面板

設定：
- **Trigger（觸發）**: On click（點擊時）
- **Action（動作）**: Navigate to（導航到）
- **Destination（目標）**: 首頁 Frame
- **Animation（動畫）**: Instant（瞬間）或 Dissolve（淡入淡出）

**預覽**：

點右上角的播放按鈕（或 `Cmd+Enter`），會開啟預覽視窗。

點登入按鈕，就會跳到首頁。成功！

## 常用的 Trigger

**On click**：點擊時（最常用）

**On hover**：滑鼠移上去時（網頁常用，手機沒有 hover）

**On press**：按下時（還沒放開）

**On drag**：拖曳時（比如滑動 slider）

**After delay**：延遲後自動觸發（比如 Splash screen 3 秒後跳到首頁）

**Mouse enter / Mouse leave**：滑鼠移入/移出

**Key/Gamepad**：鍵盤或遊戲手把（比較少用）

## 常用的 Action

**Navigate to**：跳到另一個 Frame（最常用）

**Open overlay**：打開浮層（比如 Modal、下拉選單）

**Swap with**：替換成另一個元素（比如切換 Component variant）

**Back**：返回上一頁

**Close overlay**：關閉浮層

**Open link**：開啟外部連結（實際網址）

## 動畫效果

Navigate 和 Overlay 可以設定動畫：

**Instant**：瞬間切換（無動畫）

**Dissolve**：淡入淡出

**Smart animate**：智能動畫（Figma 自動計算，稍後詳述）

**Move in / Move out**：從某個方向滑入/滑出
- Direction: Left, Right, Top, Bottom

**Push**：推擠效果，舊畫面被新畫面推出去

**Slide in / Slide out**：滑入/滑出（有背景遮罩）

每個動畫可以設定：
- **Duration**：持續時間（300ms 預設）
- **Easing**：緩動效果（Ease in, Ease out, Linear 等）

## 實戰：登入流程

做一個完整的登入流程：

1. **歡迎頁**：After delay 2s → 登入頁
2. **登入頁**：點「登入」→ 首頁
3. **登入頁**：點「註冊」→ 註冊頁
4. **註冊頁**：點「返回」→ 登入頁
5. **首頁**：點「登出」→ 登入頁

建立這些 Frame，用 Prototype 連起來。

設定動畫：
- 歡迎頁 → 登入頁：Dissolve
- 登入頁 ↔ 註冊頁：Move in/out from Right
- 登入/登出：Dissolve

預覽，整個流程就能操作了。

## Overlay（浮層）

Overlay 用來做 Modal、Alert、下拉選單等浮在畫面上的元素。

**做一個 Alert**：

建立一個 Frame（200x150），放在畫布旁邊（不是在手機 Frame 裡）。

內容：
- 半透明黑色背景（模擬遮罩）
- 白色卡片
- 標題「確認刪除？」
- 兩個按鈕：「取消」、「確定」

**設定互動**：

在首頁加一個「刪除」按鈕：
- On click → Open overlay → Alert Frame
- Position: Center
- Close when clicking outside: 勾選

Alert 的「取消」按鈕：
- On click → Close overlay

Alert 的「確定」按鈕：
- On click → Close overlay
- 然後可以再 Navigate 到其他頁面

預覽，點刪除會彈出 Alert，點外面或取消會關閉。

## Overlay 進階設定

**Position**：
- Top, Bottom, Left, Right, Center
- Manual（手動定位）

**Background**：
- None（無背景）
- Solid（純色遮罩）

**Close on click outside**：點擊外部是否關閉

**Add background behind overlay**：是否加背景遮罩

## Scrolling（滾動）

Frame 可以設定滾動行為。

做一個內容很長的頁面：

1. 建立 Frame（iPhone 14 Pro）
2. 放很多內容（超過螢幕高度）
3. 選中 Frame，右側 Prototype tab
4. Scrolling → Vertical scrolling

預覽時就能上下滾動。

可以設定：
- Horizontal scrolling（橫向滾動）
- Horizontal and vertical（雙向滾動）
- 也可以限制 Overflow behavior（Scroll 或 Clip）

## Fixed Position

某些元素要固定在畫面上，比如導航列、浮動按鈕。

選中導航列，在 Constraints 設定：
- Vertical: Top（固定在頂部）
- Horizontal: Left and Right（填滿寬度）

勾選 `Fix position when scrolling`。

這樣滾動時，導航列會固定不動。

## 互動狀態

按鈕通常有不同狀態：Default、Hover、Pressed。

如果用 Component Variant 做了這些狀態，可以用 Prototype 切換：

1. 選中按鈕 Instance
2. Prototype tab → Interactions
3. On press → Change to → Variant: Pressed
4. On mouse up → Change to → Variant: Default

或者：
5. On hover → Change to → Variant: Hover
6. On mouse leave → Change to → Variant: Default

這樣按鈕會有按下、滑過的視覺回饋。

## 實戰：下拉選單

做一個下拉選單（Dropdown）。

**準備 Component**：

建立 Dropdown Component，有兩個 Variant：
1. `State = Collapsed`（收起）：只顯示選中的值
2. `State = Expanded`（展開）：顯示所有選項

**設定互動**：

Collapsed variant：
- On click → Change to → State: Expanded

Expanded variant 的每個選項：
- On click → Change to → State: Collapsed
- （實際專案還要處理選中值的更新）

Expanded variant 的背景：
- On click → Change to → State: Collapsed（點外面收起）

預覽，點下拉選單會展開，點選項或外面會收起。

## 條件分支

有時候要根據條件做不同動作。Figma 沒有內建邏輯判斷，但可以用多個互動模擬。

比如：登入表單，帳號密碼正確 → 首頁，錯誤 → Alert。

做法：
- 準備兩個「登入」按鈕 Component variant：Success 和 Error
- 在測試時，根據要測試的情境選不同 variant

或者用 Figma 的變數功能（Variables，進階功能，之後再研究）。

## Prototype 設定

右側 Prototype tab 上方有 Flow 設定：

**Starting frame**：設定起始畫面（預覽時從這裡開始）

**Device**：選擇預覽裝置（iPhone、Android、Desktop 等）

**Background**：預覽背景顏色

可以建立多個 Flow，分別測試不同的使用者旅程（User Flow）。

## 分享 Prototype

做好 prototype 要給別人看：

1. 點右上角 `Share`
2. 選 `Get link` → `Anyone with the link can view`
3. 勾選 `Link to current page`
4. 複製連結寄給別人

別人開啟連結，右上角會有播放按鈕，點了就能操作 prototype。

也可以直接 `Share prototype`，會產生一個專門的 prototype 連結，更簡潔。

## Prototype 小技巧

**熱區提示**：預覽時按住 `Shift`，會顯示所有可點擊的區域（藍色高亮）。

**回到起點**：預覽時按 `R`，回到起始畫面。

**全螢幕**：預覽時按 `F`，全螢幕模式（沒有 Figma 的 UI）。

**顯示評論**：勾選 `Show prototype settings` → `Show comments`，預覽時能看到設計評論。

## 心得

Prototype 功能比想像中強大。基本的頁面跳轉、浮層、滾動都能做到，對於展示設計概念、測試使用者流程很夠用。

不過也有限制：
- 沒有真正的邏輯判斷（if/else）
- 不能處理真實資料
- 表單輸入只是視覺，不會真的記錄

但這也是 prototype 的定位：快速驗證概念，不是做真正的產品。

下週想研究 Smart Animate，據說可以做出很流暢的動畫效果，物件會自動補間動畫，應該蠻有趣的。
