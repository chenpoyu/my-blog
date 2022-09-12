---
layout: post
title: "iOS 自訂 UI 控制項"
date: 2018-04-20
categories: [iOS, Swift]
tags: [Swift, iOS, Custom Control, UIControl, Drawing]
---

這週挑戰自己實作自訂 UI 控制項。在 Java Swing 中，自訂元件需要繼承 JComponent 並覆寫 paintComponent 方法，然後處理各種事件監聽器。iOS 的做法類似但更有系統性。

## 為什麼需要自訂控制項

UIKit 提供的標準控制項（UIButton、UISlider、UISwitch 等）已經很豐富，但有時設計稿會要求特殊的 UI 元素。比如：

- 設計師要求的特殊樣式開關
- 評分星星控制項
- 自訂的音量滑桿
- 特殊的進度指示器

這些無法用標準控制項輕鬆實現，就需要自己動手打造。更重要的是，**自訂控制項能封裝複雜的互動邏輯**，讓程式碼更模組化、可重用。

## UIControl 的角色

iOS 提供了 `UIControl` 這個基礎類別，專門用來建立可互動的控制項。它已經處理好觸控事件、目標-動作模式、狀態管理等基礎功能，我們只需要專注於：

1. **視覺呈現**：如何畫出控制項
2. **狀態回應**：使用者互動時如何反應
3. **數值管理**：如何儲存和通知數值變化

這個架構比 Java Swing 的事件監聽器模式更清晰。Swing 中每個事件類型都要加監聽器，而 UIControl 用統一的 target-action 機制處理。

## 實作範例：評分控制項

我決定實作一個星星評分控制項，從 0 到 5 星可選。這是很常見的需求，用來理解自訂控制項的完整流程很適合。

### 第一步：設計類別結構

```swift
class StarRatingControl: UIControl {
    // 可配置屬性
    var starCount: Int = 5              // 總共幾顆星
    var rating: Int = 0 {               // 目前評分
        didSet {
            updateStarButtons()
            sendActions(for: .valueChanged)  // 通知值改變
        }
    }
    var starSize: CGFloat = 44.0        // 星星大小
    var spacing: CGFloat = 8.0          // 間距
    
    // 內部 UI 元素
    private var starButtons: [UIButton] = []
}
```

這個設計中，`rating` 是最重要的屬性。當它改變時，我們要：
1. 更新星星的視覺狀態（填滿或空心）
2. 發送 `.valueChanged` 事件，讓外部知道值變了

`didSet` 屬性觀察器在這裡很有用，類似 Java 中的 setter 方法，但語法更簡潔。

### 第二步：初始化與建立子 View

```swift
override init(frame: CGRect) {
    super.init(frame: frame)
    setupStars()
}

required init?(coder: NSCoder) {
    super.init(coder: coder)
    setupStars()
}

private func setupStars() {
    // 建立指定數量的星星按鈕
    for index in 0..<starCount {
        let button = UIButton()
        
        // 設定星星圖片
        button.setImage(UIImage(systemName: "star"), for: .normal)
        button.setImage(UIImage(systemName: "star.fill"), for: .selected)
        
        // 設定顏色
        button.tintColor = .systemYellow
        
        // 設定尺寸
        button.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            button.widthAnchor.constraint(equalToConstant: starSize),
            button.heightAnchor.constraint(equalToConstant: starSize)
        ])
        
        // 加入點擊事件
        button.addTarget(self, action: #selector(starTapped(_:)), for: .touchUpInside)
        
        // 儲存 tag 方便識別是第幾顆星
        button.tag = index
        
        addSubview(button)
        starButtons.append(button)
    }
    
    layoutStars()
}
```

這裡用了 `UIImage(named:)` 來載入圖片資源。在實際專案中，你需要在 Assets.xcassets 加入 `star-filled` 和 `star-empty` 兩張圖片。或者可以用程式碼繪製星星，但這樣會比較複雜。

注意我們給每個 button 設定了 `tag`，這樣在點擊處理中就能知道使用者點了第幾顆星。

### 第三步：佈局星星

```swift
private func layoutStars() {
    var xOffset: CGFloat = 0
    
    for button in starButtons {
        button.frame = CGRect(
            x: xOffset,
            y: 0,
            width: starSize,
            height: starSize
        )
        xOffset += starSize + spacing
    }
}

override func layoutSubviews() {
    super.layoutSubviews()
    layoutStars()
}

override var intrinsicContentSize: CGSize {
    // 計算控制項本身需要的大小
    let width = CGFloat(starCount) * starSize + CGFloat(starCount - 1) * spacing
    return CGSize(width: width, height: starSize)
}
```

`intrinsicContentSize` 很重要，它告訴 Auto Layout 系統「這個控制項自己想要多大」。如果不實作這個，控制項在 Storyboard 或 Auto Layout 中會無法正確顯示。

這個概念類似 Java Swing 的 `getPreferredSize()`，但在 iOS 的約束系統中更關鍵。

### 第四步：處理使用者互動

```swift
@objc private func starTapped(_ sender: UIButton) {
    let index = sender.tag
    
    // 如果點擊已選中的星星，取消選擇；否則設定新評分
    if rating == index + 1 {
        rating = 0
    } else {
        rating = index + 1
    }
}

private func updateStarButtons() {
    for (index, button) in starButtons.enumerated() {
        button.isSelected = index < rating
    }
}
```

這個邏輯實現了「點擊第三顆星，前三顆都會亮起」的效果。而且如果再點一次已經選中的星星，會清除評分，這是常見的 UX 模式。

`enumerated()` 方法很方便，同時給我們索引和元素，類似 Java 中的 `IntStream.range().forEach()`。

### 第五步：在 ViewController 中使用

```swift
class ViewController: UIViewController {
    let ratingControl = StarRatingControl()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 配置控制項
        ratingControl.starCount = 5
        ratingControl.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(ratingControl)
        
        // 設定約束
        NSLayoutConstraint.activate([
            ratingControl.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            ratingControl.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
        
        // 監聽值改變
        ratingControl.addTarget(
            self,
            action: #selector(ratingChanged(_:)),
            for: .valueChanged
        )
    }
    
    @objc func ratingChanged(_ sender: StarRatingControl) {
        print("新評分：\(sender.rating) 星")
        // 這裡可以執行任何需要的業務邏輯
    }
}
```

使用方式就像標準的 UIControl 一樣，用 `addTarget` 監聽事件。這正是自訂控制項的好處：**對外的介面和系統控制項完全一致**。

## 進階：支援 Interface Builder

如果要讓自訂控制項能在 Storyboard 中預覽和配置，需要加上 `@IBDesignable` 和 `@IBInspectable`：

```swift
@IBDesignable
class StarRatingControl: UIControl {
    
    @IBInspectable var starCount: Int = 5 {
        didSet {
            if starCount != oldValue {
                recreateStars()
            }
        }
    }
    
    @IBInspectable var rating: Int = 0 {
        didSet {
            updateStarButtons()
            sendActions(for: .valueChanged)
        }
    }
    
    @IBInspectable var starSize: CGFloat = 44.0 {
        didSet {
            if starSize != oldValue {
                layoutStars()
                invalidateIntrinsicContentSize()
            }
        }
    }
}
```

`@IBDesignable` 讓 Xcode 在 Interface Builder 中即時渲染控制項。`@IBInspectable` 讓屬性出現在屬性檢查器中，可以直接調整。

這在開發時很方便，不用每次都執行 app 才能看效果。但注意 `@IBDesignable` 會增加編譯時間，只在需要時使用。

## 繪製自訂圖形

如果要畫更複雜的圖形（不只是組合標準 view），需要覆寫 `draw(_:)` 方法：

```swift
override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else { return }
    
    // 設定繪圖屬性
    context.setFillColor(UIColor.systemBlue.cgColor)
    context.setStrokeColor(UIColor.white.cgColor)
    context.setLineWidth(2.0)
    
    // 繪製圓形進度
    let center = CGPoint(x: rect.midX, y: rect.midY)
    let radius = min(rect.width, rect.height) / 2 - 2
    
    let startAngle = -CGFloat.pi / 2  // 從上方開始
    let endAngle = startAngle + (CGFloat.pi * 2 * progress)
    
    // 畫背景圓
    context.addArc(
        center: center,
        radius: radius,
        startAngle: 0,
        endAngle: CGFloat.pi * 2,
        clockwise: false
    )
    context.setStrokeColor(UIColor.systemGray5.cgColor)
    context.strokePath()
    
    // 畫進度弧
    context.addArc(
        center: center,
        radius: radius,
        startAngle: startAngle,
        endAngle: endAngle,
        clockwise: false
    )
    context.strokePath()
}
```

Core Graphics 的繪圖 API 比 Java 2D 的更直觀。座標系統預設是左上角為原點，但可以透過 transform 調整。

記得當屬性改變需要重繪時，呼叫 `setNeedsDisplay()`：

```swift
var progress: CGFloat = 0 {
    didSet {
        setNeedsDisplay()  // 標記需要重繪
    }
}
```

## 效能考量

自訂繪製控制項時要注意效能。`draw(_:)` 方法不應該做複雜運算或配置記憶體，因為它可能被頻繁呼叫。

最佳實踐：
1. **快取不變的計算結果**：比如路徑、顏色
2. **只在必要時重繪**：用 `setNeedsDisplay()` 而不是直接呼叫 `draw(_:)`
3. **避免在 draw 中建立物件**：事先準備好所需的資源
4. **使用 CALayer**：複雜的靜態圖形可以用 layer 實現，效能更好

在我的 Java 經驗中，自訂繪製常常成為效能瓶頸。iOS 的繪製系統因為有硬體加速，通常不是問題，但還是要遵循最佳實踐。

## 輔助功能支援

一個好的自訂控制項應該支援輔助功能（Accessibility），讓視障使用者也能使用：

```swift
private func setupAccessibility() {
    isAccessibilityElement = true
    accessibilityTraits = .adjustable
    accessibilityLabel = "評分"
    updateAccessibilityValue()
}

private func updateAccessibilityValue() {
    accessibilityValue = "\(rating) 顆星，共 \(starCount) 顆星"
}

override func accessibilityIncrement() {
    if rating < starCount {
        rating += 1
    }
}

override func accessibilityDecrement() {
    if rating > 0 {
        rating -= 1
    }
}
```

這樣 VoiceOver 使用者可以透過上下滑動手勢來調整評分，而不需要精準點擊星星。

這在 Java 桌面應用開發中很少被重視，但 Apple 對輔助功能要求很嚴格，提交 App Store 時會檢查。

## 小結

這週實作自訂控制項的過程，讓我深刻體會到 UIKit 的**物件導向設計**。透過繼承 UIControl 並遵循既定的模式，自訂控制項能和系統控制項無縫整合。

關鍵要點：
1. **繼承 UIControl** 取得完整的觸控和事件處理能力
2. **使用 didSet** 在屬性改變時自動更新 UI
3. **實作 intrinsicContentSize** 讓 Auto Layout 正常運作
4. **用 target-action 通知外部** 值的改變
5. **支援輔助功能** 讓更多人能使用你的控制項

下週打算研究手勢辨識器（Gesture Recognizer），這能讓控制項支援更複雜的互動，像是拖曳、捏合、旋轉等。
