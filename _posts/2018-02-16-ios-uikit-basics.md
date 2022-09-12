---
layout: post
title: "iOS UIKit 基礎入門"
date: 2018-02-16
categories: [Swift, iOS]
tags: [iOS, UIKit, UI]
---

經過四週的 Swift 語法學習，這週終於開始接觸 iOS 開發的核心：**UIKit 框架**。

從 Java 開發的角度來看，UIKit 就像 Java 的 Swing 或 JavaFX，提供整套 UI 元件和事件處理機制。但 UIKit 是為觸控裝置設計的，互動模式和 Swing 的滑鼠點擊很不一樣。

一開始會覺得有點陌生，但其實很多概念是相通的：View 就是 Component，ViewController 就是 Controller，Delegate 就是 Listener。理解這些對應關係後，學起來就快很多。

## UIKit 框架概述

UIKit 是 iOS 開發的核心框架，提供應用程式的視覺元素和互動功能。所有與 UI 相關的類別都以 `UI` 開頭，像 UIView、UIViewController、UIButton 等。

### 應用程式結構

```swift
import UIKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?
    
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        return true
    }
}
```

`AppDelegate` 是應用程式的進入點，處理應用程式生命週期事件。

### 應用程式生命週期

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
    func applicationDidBecomeActive(_ application: UIApplication) {
        print("App became active")
    }
    
    func applicationWillResignActive(_ application: UIApplication) {
        print("App will resign active")
    }
    
    func applicationDidEnterBackground(_ application: UIApplication) {
        print("App entered background")
    }
    
    func applicationWillEnterForeground(_ application: UIApplication) {
        print("App will enter foreground")
    }
    
    func applicationWillTerminate(_ application: UIApplication) {
        print("App will terminate")
    }
}
```

## UIViewController

ViewController 是 MVC 架構中的 Controller，管理畫面和邏輯。

### 基本 ViewController

```swift
import UIKit

class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        view.backgroundColor = .white
        print("View did load")
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        print("View will appear")
    }
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        print("View did appear")
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        print("View will disappear")
    }
    
    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        print("View did disappear")
    }
}
```

### 生命週期順序

```
viewDidLoad
viewWillAppear
viewDidAppear
viewWillDisappear
viewDidDisappear
```

## UIView

UIView 是所有視覺元素的基礎類別。

### 建立和配置 View

```swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let redView = UIView()
        redView.backgroundColor = .red
        redView.frame = CGRect(x: 50, y: 100, width: 200, height: 150)
        view.addSubview(redView)
        
        let blueView = UIView()
        blueView.backgroundColor = .blue
        blueView.frame = CGRect(x: 100, y: 200, width: 150, height: 100)
        view.addSubview(blueView)
    }
}
```

### Frame 與 Bounds

```swift
let view = UIView()
view.frame = CGRect(x: 50, y: 100, width: 200, height: 150)

print(view.frame)   // (50, 100, 200, 150) - 相對於父 view
print(view.bounds)  // (0, 0, 200, 150) - 自己的座標系統
print(view.center)  // (150, 175) - 中心點
```

- `frame`：在父 view 座標系統中的位置和大小
- `bounds`：在自己座標系統中的大小
- `center`：在父 view 座標系統中的中心點

## UILabel

顯示文字的元件。

```swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let label = UILabel()
        label.text = "Hello, iOS"
        label.font = UIFont.systemFont(ofSize: 24)
        label.textColor = .black
        label.textAlignment = .center
        label.numberOfLines = 0  // 0 表示不限行數
        label.frame = CGRect(x: 50, y: 100, width: 300, height: 100)
        
        view.addSubview(label)
    }
}
```

### 多行文字

```swift
let label = UILabel()
label.text = "This is a very long text that will span multiple lines"
label.numberOfLines = 0
label.lineBreakMode = .byWordWrapping
label.frame = CGRect(x: 20, y: 100, width: 280, height: 200)
```

## UIButton

按鈕元件。

```swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let button = UIButton(type: .system)
        button.setTitle("Tap Me", for: .normal)
        button.setTitle("Tapped", for: .highlighted)
        button.frame = CGRect(x: 100, y: 200, width: 200, height: 50)
        button.addTarget(self, action: #selector(buttonTapped), for: .touchUpInside)
        
        view.addSubview(button)
    }
    
    @objc func buttonTapped() {
        print("Button was tapped")
    }
}
```

### 自訂按鈕外觀

```swift
let button = UIButton()
button.setTitle("Submit", for: .normal)
button.setTitleColor(.white, for: .normal)
button.backgroundColor = .blue
button.layer.cornerRadius = 8
button.frame = CGRect(x: 100, y: 300, width: 200, height: 50)
```

## UITextField

文字輸入欄位。

```swift
class ViewController: UIViewController, UITextFieldDelegate {
    let textField = UITextField()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        textField.placeholder = "Enter your name"
        textField.borderStyle = .roundedRect
        textField.frame = CGRect(x: 50, y: 150, width: 300, height: 40)
        textField.delegate = self
        
        view.addSubview(textField)
    }
    
    func textFieldShouldReturn(_ textField: UITextField) -> Bool {
        textField.resignFirstResponder()
        return true
    }
    
    func textFieldDidEndEditing(_ textField: UITextField) {
        if let text = textField.text {
            print("User entered: \(text)")
        }
    }
}
```

### 鍵盤處理

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    NotificationCenter.default.addObserver(
        self,
        selector: #selector(keyboardWillShow),
        name: UIResponder.keyboardWillShowNotification,
        object: nil
    )
    
    NotificationCenter.default.addObserver(
        self,
        selector: #selector(keyboardWillHide),
        name: UIResponder.keyboardWillHideNotification,
        object: nil
    )
}

@objc func keyboardWillShow(notification: Notification) {
    if let keyboardSize = (notification.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? NSValue)?.cgRectValue {
        print("Keyboard height: \(keyboardSize.height)")
    }
}

@objc func keyboardWillHide(notification: Notification) {
    print("Keyboard will hide")
}
```

## UIImageView

顯示圖片的元件。

```swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageView = UIImageView()
        imageView.image = UIImage(named: "photo")
        imageView.contentMode = .scaleAspectFit
        imageView.frame = CGRect(x: 50, y: 100, width: 300, height: 200)
        
        view.addSubview(imageView)
    }
}
```

### ContentMode

```swift
imageView.contentMode = .scaleAspectFit   // 保持比例，完整顯示
imageView.contentMode = .scaleAspectFill  // 保持比例，填滿
imageView.contentMode = .scaleToFill      // 不保持比例，拉伸填滿
imageView.contentMode = .center           // 置中，不縮放
```

## 事件處理

### Target-Action 模式

```swift
let button = UIButton()
button.addTarget(self, action: #selector(buttonTapped(_:)), for: .touchUpInside)

@objc func buttonTapped(_ sender: UIButton) {
    sender.backgroundColor = .green
}
```

### Gesture Recognizer

```swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let redView = UIView()
        redView.backgroundColor = .red
        redView.frame = CGRect(x: 100, y: 200, width: 200, height: 200)
        view.addSubview(redView)
        
        let tapGesture = UITapGestureRecognizer(target: self, action: #selector(handleTap))
        redView.addGestureRecognizer(tapGesture)
        redView.isUserInteractionEnabled = true
    }
    
    @objc func handleTap() {
        print("View tapped")
    }
}
```

### 多種手勢

```swift
// Tap
let tapGesture = UITapGestureRecognizer(target: self, action: #selector(handleTap))

// Long Press
let longPressGesture = UILongPressGestureRecognizer(target: self, action: #selector(handleLongPress))

// Swipe
let swipeGesture = UISwipeGestureRecognizer(target: self, action: #selector(handleSwipe))
swipeGesture.direction = .left

// Pan
let panGesture = UIPanGestureRecognizer(target: self, action: #selector(handlePan))

// Pinch
let pinchGesture = UIPinchGestureRecognizer(target: self, action: #selector(handlePinch))
```

## 實作簡單應用程式

### 計數器應用

```swift
class CounterViewController: UIViewController {
    var count = 0
    
    let countLabel = UILabel()
    let incrementButton = UIButton(type: .system)
    let decrementButton = UIButton(type: .system)
    let resetButton = UIButton(type: .system)
    
    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .white
        
        setupCountLabel()
        setupButtons()
    }
    
    func setupCountLabel() {
        countLabel.text = "\(count)"
        countLabel.font = UIFont.systemFont(ofSize: 48, weight: .bold)
        countLabel.textAlignment = .center
        countLabel.frame = CGRect(x: 0, y: 150, width: view.bounds.width, height: 60)
        view.addSubview(countLabel)
    }
    
    func setupButtons() {
        incrementButton.setTitle("+", for: .normal)
        incrementButton.titleLabel?.font = UIFont.systemFont(ofSize: 36)
        incrementButton.frame = CGRect(x: view.bounds.width - 120, y: 250, width: 80, height: 80)
        incrementButton.addTarget(self, action: #selector(increment), for: .touchUpInside)
        view.addSubview(incrementButton)
        
        decrementButton.setTitle("-", for: .normal)
        decrementButton.titleLabel?.font = UIFont.systemFont(ofSize: 36)
        decrementButton.frame = CGRect(x: 40, y: 250, width: 80, height: 80)
        decrementButton.addTarget(self, action: #selector(decrement), for: .touchUpInside)
        view.addSubview(decrementButton)
        
        resetButton.setTitle("Reset", for: .normal)
        resetButton.frame = CGRect(x: (view.bounds.width - 100) / 2, y: 350, width: 100, height: 40)
        resetButton.addTarget(self, action: #selector(reset), for: .touchUpInside)
        view.addSubview(resetButton)
    }
    
    @objc func increment() {
        count += 1
        updateLabel()
    }
    
    @objc func decrement() {
        count -= 1
        updateLabel()
    }
    
    @objc func reset() {
        count = 0
        updateLabel()
    }
    
    func updateLabel() {
        countLabel.text = "\(count)"
    }
}
```

## 與 Android 的比較

作為 Java 工程師，與 Android 開發的對比：

1. **生命週期**：類似 Android Activity，但更簡潔
2. **佈局方式**：iOS 可用程式碼或 Interface Builder，Android 用 XML
3. **事件處理**：iOS 用 Target-Action，Android 用 Listener
4. **UI 元件**：概念類似，但名稱和 API 不同

## 小結

UIKit 提供完整的 UI 開發能力。ViewController 管理畫面生命週期，各種 UIView 子類別提供不同的 UI 元件。事件處理使用 Target-Action 模式和 Gesture Recognizer。目前使用程式碼建立 UI，下週將學習 Interface Builder 和 Auto Layout。
