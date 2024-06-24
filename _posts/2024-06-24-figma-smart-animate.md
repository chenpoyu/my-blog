---
layout: post
title: "Figma Smart Animate：打造流暢的動畫"
date: 2024-06-24 14:15:00 +0800
categories: [設計, UI/UX]
tags: [Figma, Smart Animate, Animation]
---

上週學了 Prototype 的基本互動，這週來研究 Smart Animate。這是 Figma 最強大的動畫功能，可以自動在兩個狀態之間產生過渡動畫。

概念類似 CSS transition，你只要定義起始和結束狀態，Figma 會自動計算中間的動畫過程。

## Smart Animate 的原理

Smart Animate 會比對兩個 Frame 之間相同名稱的圖層，自動產生補間動畫（Tween）。

動畫屬性包括：
- 位置（X, Y）
- 大小（Width, Height）
- 旋轉（Rotation）
- 透明度（Opacity）
- 圓角（Corner Radius）
- 顏色（Fill）

只要圖層名稱相同，Figma 就知道是同一個物件，會自動補間。

## 第一個 Smart Animate

做一個簡單的動畫：方塊從左移到右。

**Frame 1**：

畫一個正方形（100x100），藍色，命名為 `Box`。

位置在畫面左側（X: 50）。

**Frame 2**：

複製整個 Frame（`Cmd+D`）。

把方塊移到右側（X: 270）。

**設定互動**：

Frame 1 的方塊：
- On click → Navigate to → Frame 2
- Animation: **Smart animate**
- Duration: 500ms
- Easing: Ease out

預覽，點方塊，它會流暢地從左滑到右。

關鍵是兩個 Frame 的方塊都叫 `Box`，Figma 認得是同一個物件。

## 多屬性動畫

Smart Animate 可以同時改變多個屬性。

**Frame 1**：
- Box: 100x100, 藍色, 圓角 0, 位置 (50, 200)

**Frame 2**：
- Box: 150x150, 紅色, 圓角 75（圓形）, 位置 (200, 150)

用 Smart Animate 連接，預覽時方塊會：
- 移動
- 放大
- 變色（藍 → 紅）
- 變圓

全部同時發生，很流暢。

## 旋轉動畫

旋轉要注意設定正確的角度。

**Frame 1**：
- Box: Rotation 0°

**Frame 2**：
- Box: Rotation 180°

Smart Animate 連接，方塊會旋轉 180 度。

如果要連續旋轉（比如 loading），可以做多個 Frame：
- Frame 1: 0°
- Frame 2: 90°
- Frame 3: 180°
- Frame 4: 270°
- Frame 5: 360°

用 After delay 自動觸發下一個 Frame，形成循環。

## 透明度與淡入淡出

改變 Opacity 可以做淡入淡出效果。

**Frame 1**：
- Box: Opacity 100%

**Frame 2**：
- Box: Opacity 0%

Smart Animate 連接，方塊會淡出消失。

也可以做「出現」動畫：
- Frame 1: Opacity 0%（不可見）
- Frame 2: Opacity 100%（顯示）

## Component Variant 切換

Smart Animate 配合 Component Variant 超好用。

做一個按鈕 Component：

**Variant 1: State = Default**
- Size: 120x40
- Fill: 藍色

**Variant 2: State = Pressed**
- Size: 120x36（稍微壓扁）
- Fill: 深藍色

設定互動：
- On press → Change to → State: Pressed, Smart animate, 100ms
- On mouse up → Change to → State: Default, Smart animate, 100ms

預覽，按下按鈕會有壓下的動畫效果。

## 實戰：卡片展開動畫

做一個常見的效果：點卡片，展開顯示詳細內容。

**Frame 1：列表**

三張卡片（Card1, Card2, Card3）垂直排列，每張：
- Size: 320x120
- 顯示標題、簡短描述
- 圖層名稱：Card1, Card2, Card3

**Frame 2：展開**

Card2 展開：
- Size: 320x400（變高）
- 顯示完整內容、更多文字、按鈕
- 圖層名稱：Card2（保持相同）

Card1 和 Card3 往上下移動，讓出空間。

**設定互動**：

Frame 1 的 Card2：
- On click → Navigate to → Frame 2
- Smart animate, 400ms, Ease in out

Frame 2 的關閉按鈕：
- On click → Back
- Smart animate, 400ms

預覽，點卡片會展開，其他卡片會移動讓出位置，很流暢。

## Match layers（配對圖層）

如果圖層名稱不同，但想要動畫，可以手動配對。

比如：
- Frame 1: 圖層叫 `Button 1`
- Frame 2: 圖層叫 `Button Primary`

正常 Smart Animate 認不出是同一個物件。

解決方法：統一命名。或者在 Prototype 設定裡用 `Match layers` 手動配對（進階功能，通常不需要）。

建議還是保持圖層名稱一致。

## 物件消失與出現

如果 Frame 1 有 `Box`，Frame 2 沒有，Smart Animate 會讓 Box 淡出消失。

如果 Frame 1 沒有 `Circle`，Frame 2 有，Circle 會淡入出現。

利用這個特性可以做很多效果。

## 實戰：Tab 切換動畫

做一個 Tab 切換，有滑動效果。

**Frame 1: Tab 1 active**

- Tab bar: Tab1（藍色）, Tab2（灰色）, Tab3（灰色）
- 下方指示器（Indicator）在 Tab1 下方
- 內容區域顯示 Content1

**Frame 2: Tab 2 active**

- Tab bar: Tab1（灰色）, Tab2（藍色）, Tab3（灰色）
- Indicator 移到 Tab2 下方
- 內容區域顯示 Content2

**Frame 3: Tab 3 active**

類推。

設定互動：
- 點 Tab1 → Frame 1, Smart animate
- 點 Tab2 → Frame 2, Smart animate
- 點 Tab3 → Frame 3, Smart animate

關鍵是 Indicator 在三個 Frame 都叫相同名稱（比如 `Indicator`），位置不同。

預覽時點 Tab，指示器會滑動到對應位置，內容也會切換。

## 進階：路徑動畫

Smart Animate 也能做曲線路徑動畫，但需要一些技巧。

Figma 沒有直接定義路徑的功能，但可以用多個 Frame 模擬。

比如讓物件沿著圓形軌跡移動：
- Frame 1: 位置 (100, 100) - 0°
- Frame 2: 位置 (150, 80) - 45°
- Frame 3: 位置 (180, 100) - 90°
- ...（12 個 Frame 形成圓）

用 After delay 自動播放，形成循環。

不過這很費工，實務上比較少這樣做。

## While pressing（按住）

可以做按住時的動畫效果。

**Frame 1: Button Default**

按鈕正常大小。

**Frame 2: Button Pressed**

按鈕稍微縮小（95%）。

設定互動：
- While pressing → Navigate to → Frame 2, Smart animate, 100ms
- 放開時自動回到 Frame 1

這樣按住按鈕，它會縮小；放開就恢復。

## Easing（緩動）

Smart Animate 的 Easing 會影響動畫的感覺：

**Linear**：等速，機械感

**Ease in**：慢速開始，加速結束（物體開始移動）

**Ease out**：快速開始，減速結束（物體停止）

**Ease in and out**：慢速開始和結束（最自然）

**Custom bezier**：自訂曲線（進階）

常用的選擇：
- 移動物體：Ease in out
- 淡入淡出：Ease out
- 彈出效果：Ease out（或用 Spring）

## Spring（彈簧）

Figma 也支援彈簧動畫（Spring），更有彈性感。

在 Easing 選擇 `Spring`，可以調整：
- **Stiffness（硬度）**：數值越大，彈得越快
- **Damping（阻尼）**：數值越大，越不會彈
- **Mass（質量）**：數值越大，越有重量感

試試不同參數，找到喜歡的效果。

## 實戰：抽屜選單動畫

做一個從左側滑出的側邊選單（Drawer）。

**Frame 1: Menu Closed**

選單在畫面外（X: -280，完全在左邊看不到）。

**Frame 2: Menu Open**

選單滑入（X: 0）。

另外加一個半透明黑色遮罩（Overlay），在 Frame 1 是 Opacity 0%，Frame 2 是 Opacity 50%。

設定互動：
- 點選單圖示 → Navigate to Frame 2, Smart animate, 300ms, Ease out
- 點遮罩 → Back, Smart animate, 300ms
- 點選單項目 → Navigate to 對應頁面

預覽，點圖示選單滑出，點外面收回。

## 注意事項

**圖層名稱要一致**：這是最重要的，不一致就沒動畫。

**不要過度使用**：太多動畫會讓人眼花撩亂，適度就好。

**持續時間**：通常 200-400ms 比較合適，太慢會拖泥帶水，太快會看不清。

**效能**：複雜的動畫在手機上可能卡頓，實際開發時要注意。

**測試不同裝置**：在不同尺寸的 Frame 測試，確保動畫在各種裝置都好看。

## 實用範例

**Loading 動畫**：三個點上下跳動
- 用三個圓（Dot1, Dot2, Dot3）
- 多個 Frame 定義不同的上下位置
- After delay 循環播放

**進度條**：寬度從 0% 到 100%
- Frame 1: Progress bar width 0
- Frame 2: Progress bar width 100%
- Smart animate

**通知彈出**：從上方滑下
- Frame 1: Notification Y: -100（在上方看不到）
- Frame 2: Notification Y: 20
- After delay 3s → Frame 1（自動消失）

**切換開關（Toggle）**：
- Frame 1: 開關在左，灰色
- Frame 2: 開關在右，藍色
- Smart animate

## 心得

Smart Animate 真的很強大，可以做出各種流暢的動畫效果。只要理解「相同名稱的圖層會自動補間」這個原則，就能做出很多變化。

跟寫 CSS animation 或 JavaScript 動畫比起來，Figma 的方式更視覺化，不用寫 code 就能快速原型化。

不過也要注意，prototype 的動畫只是展示用，實際開發時工程師還是要重新實作。但至少能清楚傳達你要的效果。

下週想研究 Figma 的進階互動，像是 Drag、Mouse events、Variables 等，讓 prototype 更接近真實產品的互動體驗。
