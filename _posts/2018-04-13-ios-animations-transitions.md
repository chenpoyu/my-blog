---
layout: post
title: "iOS 動畫與轉場效果"
date: 2018-04-13
categories: [iOS, Swift]
tags: [Swift, iOS, Animation, UIView, Core Animation]
---

這週開始深入研究 iOS 的動畫系統。在 Java Web 開發中，動畫通常是前端的事情，後端很少處理。但在 iOS 原生開發中，動畫是提升使用者體驗的關鍵要素，Apple 對此提供了非常完整的支援。

## 為什麼動畫很重要

剛開始寫 iOS 時，我覺得動畫只是「錦上添花」的裝飾。但實際使用 iOS 系統 app 後發現，動畫不只是為了好看，更重要的是**提供視覺反饋和上下文連續性**。

比如當你點擊 app 圖示，它會先放大再進入應用；當你刪除照片，照片會先縮小淡出再消失。這些動畫告訴使用者「你的操作已經被系統接收並執行了」，而不是突然跳轉讓人不知所措。

在 Java Swing 開發中也有動畫，但通常要自己控制計時器和重繪邏輯，相當繁瑣。iOS 的動畫 API 則設計得非常人性化，大部分情況下只需要幾行程式碼。

## UIView 動畫的核心概念

iOS 動畫的基礎是 `UIView.animate` 系列方法。它的設計哲學很有趣：**你只需要描述「結束狀態」，系統會自動幫你計算中間過程**。

這個概念類似 Java 中的 Future 或 Promise，你不需要管理執行細節，只需要宣告最終結果。差別在於 UIView 動畫處理的是視覺屬性的變化，而不是非同步運算。

UIView 可以動畫化的屬性包括：
- **frame / bounds / center**：位置和大小變化
- **transform**：旋轉、縮放、傾斜
- **alpha**：透明度
- **backgroundColor**：背景顏色

注意這些都是「可動畫化」的屬性。不是所有屬性都能動畫，比如 `text` 或 `image` 就不行，因為它們沒有明確的「中間狀態」。

## 基本動畫實作

最簡單的動畫只需要三個元素：**持續時間、動畫內容、結束狀態**。

```swift
// 讓 view 在 0.5 秒內向右移動 100 點並淡出
UIView.animate(withDuration: 0.5) {
    self.myView.center.x += 100
    self.myView.alpha = 0.3
}
```

執行時，系統會自動在 0.5 秒內平滑地改變 center 和 alpha 的值。這背後使用了 Core Animation 框架，但 UIKit 已經幫我們封裝好了。

如果需要知道動畫何時結束，可以使用完成回呼：

```swift
UIView.animate(withDuration: 0.5, animations: {
    self.myView.center.x += 100
}) { finished in
    if finished {
        print("動畫完成")
        // 可以在這裡接續下一個動畫或操作
    }
}
```

這個 `finished` 參數很重要。如果動畫被中斷（比如使用者切換到其他 app），finished 會是 false。這讓我們能正確處理異常情況。

## 動畫曲線與時間函數

預設的動畫是線性變化，但這通常不自然。真實世界的物體移動會有加速和減速。iOS 提供了多種**緩動曲線（Easing Curve）**：

```swift
UIView.animate(
    withDuration: 0.5,
    delay: 0,
    options: [.curveEaseInOut],  // 開始慢、中間快、結束慢
    animations: {
        self.myView.center.y += 200
    }
)
```

常用的曲線選項：
- **curveLinear**：等速，很少用，感覺機械
- **curveEaseIn**：開始慢後來快，適合物體消失的動畫
- **curveEaseOut**：開始快後來慢，適合物體出現的動畫
- **curveEaseInOut**：兩端慢中間快，最自然也是預設值

Apple 建議大部分情況用 `curveEaseInOut`，因為它模擬了真實物理世界的慣性。

## 彈簧動畫

iOS 7 引入了更自然的彈簧動畫，模擬物理世界的彈性效果：

```swift
UIView.animate(
    withDuration: 0.6,
    delay: 0,
    usingSpringWithDamping: 0.6,    // 阻尼係數：0-1，越小彈性越大
    initialSpringVelocity: 0.5,     // 初始速度
    options: [],
    animations: {
        self.button.transform = CGAffineTransform(scaleX: 1.2, y: 1.2)
    }
)
```

`damping` 參數控制彈簧的「硬度」：
- **接近 1.0**：幾乎沒有彈跳，很快穩定
- **0.5-0.7**：適度彈跳，感覺活潑但不過分
- **接近 0.0**：劇烈彈跳，像果凍一樣

實務上我發現 0.6-0.7 最常用，既有動態感又不會讓人分心。

## Transform 轉換

`transform` 屬性是動畫的精華，它能做到位移、旋轉、縮放：

```swift
// 放大 1.5 倍
view.transform = CGAffineTransform(scaleX: 1.5, y: 1.5)

// 旋轉 45 度（注意用弧度）
view.transform = CGAffineTransform(rotationAngle: .pi / 4)

// 平移
view.transform = CGAffineTransform(translationX: 100, y: 50)

// 組合多個轉換
view.transform = CGAffineTransform(scaleX: 1.5, y: 1.5)
    .rotated(by: .pi / 4)
    .translatedBy(x: 100, y: 50)
```

Transform 的妙處在於**不影響佈局**。即使 view 旋轉或縮放，它在佈局系統中的 frame 和 bounds 依然不變，不會影響其他 view 的位置。這在 Java Swing 中需要手動重新計算佈局，iOS 的這個設計省了很多麻煩。

要恢復原狀，使用 `.identity`：

```swift
view.transform = .identity  // 回到原始狀態
```

## 動畫序列與巢狀

有時需要依序執行多個動畫。可以在完成回呼中啟動下一個動畫：

```swift
// 先淡出
UIView.animate(withDuration: 0.3, animations: {
    self.messageLabel.alpha = 0
}) { _ in
    // 改變文字
    self.messageLabel.text = "新訊息"
    // 再淡入
    UIView.animate(withDuration: 0.3) {
        self.messageLabel.alpha = 1
    }
}
```

這種串接方式在邏輯上很清楚，但如果動畫步驟很多，會形成「回呼地獄」。更好的做法是使用 `animateKeyframes`（關鍵影格動畫），這個我下週再深入研究。

## 實際應用：按鈕點擊反饋

一個好的 UX 設計是按鈕被點擊時有視覺回饋。我實作了這個通用的按鈕動畫效果：

```swift
@IBAction func buttonTapped(_ sender: UIButton) {
    // 按下時縮小
    UIView.animate(
        withDuration: 0.1,
        animations: {
            sender.transform = CGAffineTransform(scaleX: 0.95, y: 0.95)
        }
    ) { _ in
        // 放開時回彈
        UIView.animate(
            withDuration: 0.3,
            delay: 0,
            usingSpringWithDamping: 0.5,
            initialSpringVelocity: 0.8,
            options: [],
            animations: {
                sender.transform = .identity
            }
        )
    }
    
    // 執行實際的按鈕功能
    performButtonAction()
}
```

這個效果模擬了真實按鈕被按下的感覺，使用者體驗會好很多。我在 Java Swing 專案中從來沒做過這種細節，但在 iOS 開發中這幾乎是標準作法。

## 轉場動畫

除了 view 本身的動畫，iOS 還提供了容器級的**轉場動畫（Transition）**。當 view 在容器中被加入或移除時，可以有過渡效果：

```swift
UIView.transition(
    with: containerView,
    duration: 0.5,
    options: .transitionFlipFromLeft,  // 翻轉效果
    animations: {
        // 移除舊 view
        self.oldView.removeFromSuperview()
        // 加入新 view
        self.containerView.addSubview(self.newView)
    }
)
```

常用的轉場選項：
- **transitionFlipFromLeft / Right**：翻牌效果
- **transitionCurlUp / Down**：捲頁效果
- **transitionCrossDissolve**：交叉淡入淡出

這些轉場特別適合用在內容切換的場景，比如圖片輪播、卡片切換等。

## 小結

這週學習動畫的過程中，最大的體會是 iOS 動畫設計的**宣告式特性**。你不需要自己寫計時器、計算插值、觸發重繪，只需要宣告「我要什麼結果」，系統會處理所有細節。

這和 Spring 的依賴注入有異曲同工之妙：開發者專注於「做什麼」而不是「怎麼做」，框架負責實現細節。

下週打算研究更進階的 Core Animation，尤其是 CALayer 和關鍵影格動畫，應該能做出更複雜的效果。
