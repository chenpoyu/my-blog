---
layout: post
title: "Interface Builder 與 Storyboard"
date: 2018-02-23
categories: [Swift, iOS]
tags: [iOS, Interface Builder, Storyboard, IBOutlet]
---

這週學習用 **Interface Builder** 設計 UI。在 Java Swing 開發中，我用過 Eclipse 的 WindowBuilder，能視覺化建立 UI 並生成程式碼。Interface Builder 的概念類似，但更強大。

最大的差異是：
- **Swing**：GUI Builder 生成 Java 程式碼，編輯 UI 就是編輯程式碼
- **iOS**：Interface Builder 生成 .storyboard 或 .xib 檔案（XML 格式），程式碼和 UI 分離

這種設計有利有弊。好處是 UI 與邏輯分離、視覺化編輯不會弄亂程式碼。壞處是 merge conflict 比較難處理，而且學習 Auto Layout 需要時間。

## Interface Builder 簡介

Interface Builder 是 Xcode 內建的視覺化 UI 設計工具，類似 Android 的 Layout Editor。

### Storyboard vs XIB

- **Storyboard**：可以包含多個 ViewController 和畫面轉場
- **XIB**：單一 View 或 ViewController 的介面檔案

大多數專案使用 Storyboard，Main.storyboard 是預設的主要介面檔案。

## 基本操作

### 開啟 Storyboard

在 Xcode 專案導航器中點擊 `Main.storyboard`，會開啟 Interface Builder。

### 編輯器區域

- **Canvas**：畫面設計區域
- **Document Outline**：顯示 View 階層
- **Inspectors**：右側屬性面板
- **Object Library**：UI 元件庫

### 新增 UI 元件

1. 開啟 Object Library (右下角 +)
2. 拖曳元件到 Canvas
3. 在 Attributes Inspector 調整屬性

## IBOutlet 與 IBAction

### IBOutlet

連結 Storyboard 中的 UI 元件到程式碼：

```swift
class ViewController: UIViewController {
    @IBOutlet weak var nameLabel: UILabel!
    @IBOutlet weak var emailTextField: UITextField!
    @IBOutlet weak var submitButton: UIButton!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        nameLabel.text = "Welcome"
        emailTextField.placeholder = "Enter email"
    }
}
```

### 建立 IBOutlet 連結

1. 開啟 Assistant Editor (右上角兩個圓圈圖示)
2. 按住 Control，從 Storyboard 拖曳元件到程式碼
3. 選擇 Outlet，輸入名稱
4. 連結會自動產生

### IBAction

連結 UI 事件到方法：

```swift
class ViewController: UIViewController {
    @IBOutlet weak var resultLabel: UILabel!
    
    @IBAction func buttonTapped(_ sender: UIButton) {
        resultLabel.text = "Button was tapped"
    }
    
    @IBAction func switchChanged(_ sender: UISwitch) {
        if sender.isOn {
            view.backgroundColor = .white
        } else {
            view.backgroundColor = .gray
        }
    }
}
```

### 建立 IBAction 連結

1. 開啟 Assistant Editor
2. 按住 Control，從按鈕拖曳到程式碼
3. 選擇 Action，輸入名稱，選擇事件類型
4. 方法會自動產生

## 常用 UI 元件設定

### UILabel

在 Attributes Inspector 中：
- Text：設定文字內容
- Color：文字顏色
- Font：字型大小和樣式
- Alignment：對齊方式
- Lines：行數 (0 表示不限)

### UIButton

- Type：System, Custom, Detail, etc.
- Title：按鈕文字
- Font：字型
- Text Color：文字顏色
- Background：背景顏色或圖片

### UITextField

- Placeholder：提示文字
- Border Style：邊框樣式
- Keyboard Type：鍵盤類型
- Return Key：Return 鍵顯示文字
- Secure Text Entry：密碼輸入

### UISwitch

```swift
class ViewController: UIViewController {
    @IBOutlet weak var notificationSwitch: UISwitch!
    
    @IBAction func switchValueChanged(_ sender: UISwitch) {
        if sender.isOn {
            print("Notifications enabled")
        } else {
            print("Notifications disabled")
        }
    }
}
```

### UISlider

```swift
class ViewController: UIViewController {
    @IBOutlet weak var volumeSlider: UISlider!
    @IBOutlet weak var volumeLabel: UILabel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        volumeSlider.minimumValue = 0
        volumeSlider.maximumValue = 100
        volumeSlider.value = 50
    }
    
    @IBAction func sliderValueChanged(_ sender: UISlider) {
        volumeLabel.text = String(format: "%.0f", sender.value)
    }
}
```

## Auto Layout 基礎

Auto Layout 讓 UI 能適應不同螢幕大小和方向。

### Constraints

約束定義 View 之間的關係：

- Leading：左邊界
- Trailing：右邊界
- Top：上邊界
- Bottom：下邊界
- Width：寬度
- Height：高度
- CenterX：水平置中
- CenterY：垂直置中

### 新增 Constraints

方法一：Control + 拖曳
1. 按住 Control，從一個 View 拖曳到另一個 View
2. 選擇約束類型

方法二：使用底部按鈕
- Align：對齊
- Add New Constraints：新增間距、寬高約束
- Resolve Auto Layout Issues：解決約束問題

### 常見約束組合

```
置中按鈕：
- Center X in Safe Area
- Center Y in Safe Area
- Width: 200
- Height: 50

全螢幕 View：
- Leading: 0
- Trailing: 0
- Top: 0
- Bottom: 0

固定邊距：
- Leading: 20
- Trailing: 20
- Top: 50
- Height: 40
```

### 更新 Constraints

```swift
class ViewController: UIViewController {
    @IBOutlet weak var boxTopConstraint: NSLayoutConstraint!
    
    @IBAction func moveUp() {
        boxTopConstraint.constant -= 20
        UIView.animate(withDuration: 0.3) {
            self.view.layoutIfNeeded()
        }
    }
}
```

## Stack View

UIStackView 簡化佈局，自動排列子 View。

### Horizontal Stack View

```
三個按鈕水平排列：
1. 選取三個按鈕
2. 點擊底部 Embed In Stack
3. 設定 Axis: Horizontal
4. Distribution: Fill Equally
5. Spacing: 10
```

### Vertical Stack View

```
表單欄位垂直排列：
1. 將 Label 和 TextField 放入 Stack View
2. Axis: Vertical
3. Alignment: Fill
4. Distribution: Fill
5. Spacing: 8
```

### Stack View 屬性

- **Axis**：Horizontal 或 Vertical
- **Distribution**：子 View 的大小分配方式
  - Fill：填滿，根據 content hugging 調整
  - Fill Equally：平均分配
  - Fill Proportionally：按比例分配
  - Equal Spacing：固定間距
  - Equal Centering：中心點等距
- **Alignment**：垂直於 axis 的對齊
- **Spacing**：間距

## Segue 與畫面轉場

### 建立 Segue

1. 按住 Control，從按鈕拖曳到目標 ViewController
2. 選擇 Segue 類型：
   - Show：Push navigation
   - Show Detail：Detail view
   - Present Modally：Modal present
   - Present as Popover：Popover

### 程式碼觸發 Segue

```swift
class ViewController: UIViewController {
    @IBAction func showDetail() {
        performSegue(withIdentifier: "showDetail", sender: self)
    }
    
    override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
        if segue.identifier == "showDetail" {
            if let detailVC = segue.destination as? DetailViewController {
                detailVC.data = "Some data"
            }
        }
    }
}
```

### Unwind Segue

從子畫面回到父畫面：

```swift
// 在父 ViewController
@IBAction func unwindToMain(segue: UIStoryboardSegue) {
    print("Returned to main")
}

// 在 Storyboard 中，從子畫面的按鈕 Control + 拖曳到 Exit icon
```

## 實作登入畫面

### Storyboard 設計

```
佈局結構：
Container View (全螢幕)
  Stack View (Vertical, center)
    Logo ImageView (Height: 100)
    Spacer View (Height: 40)
    Username TextField
    Password TextField (Secure)
    Spacer View (Height: 20)
    Login Button
    Register Button
```

### ViewController 程式碼

```swift
class LoginViewController: UIViewController {
    @IBOutlet weak var usernameTextField: UITextField!
    @IBOutlet weak var passwordTextField: UITextField!
    @IBOutlet weak var loginButton: UIButton!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        setupUI()
    }
    
    func setupUI() {
        loginButton.layer.cornerRadius = 8
        loginButton.backgroundColor = .systemBlue
        
        usernameTextField.borderStyle = .roundedRect
        passwordTextField.borderStyle = .roundedRect
        passwordTextField.isSecureTextEntry = true
    }
    
    @IBAction func loginTapped(_ sender: UIButton) {
        guard let username = usernameTextField.text, !username.isEmpty,
              let password = passwordTextField.text, !password.isEmpty else {
            showAlert(message: "Please enter username and password")
            return
        }
        
        if validateCredentials(username: username, password: password) {
            performSegue(withIdentifier: "showHome", sender: self)
        } else {
            showAlert(message: "Invalid credentials")
        }
    }
    
    func validateCredentials(username: String, password: String) -> Bool {
        return username == "admin" && password == "1234"
    }
    
    func showAlert(message: String) {
        let alert = UIAlertController(title: "Error", message: message, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }
}
```

## Size Classes

Size Classes 讓 UI 能適應不同裝置。

### 基本概念

- **Compact Width**：iPhone 直立、iPhone 橫放的分割視圖
- **Regular Width**：iPad、iPhone Plus 橫放
- **Compact Height**：iPhone 橫放
- **Regular Height**：iPhone 直立、iPad

### 使用 Size Classes

1. 在 Interface Builder 底部點擊 "View as: iPhone 8"
2. 選擇不同裝置或方向
3. 點擊 "Vary for Traits" 設定特定 Size Class 的約束

## 最佳實踐

### 命名規範

```swift
// Outlet 命名
@IBOutlet weak var titleLabel: UILabel!
@IBOutlet weak var submitButton: UIButton!
@IBOutlet weak var emailTextField: UITextField!

// Action 命名
@IBAction func submitButtonTapped(_ sender: UIButton)
@IBAction func cancelButtonTapped(_ sender: UIButton)
```

### 組織結構

```swift
class ViewController: UIViewController {
    // MARK: - Outlets
    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var searchBar: UISearchBar!
    
    // MARK: - Properties
    var data: [String] = []
    
    // MARK: - Lifecycle
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
    }
    
    // MARK: - Setup
    private func setupUI() {
        // UI 設定
    }
    
    // MARK: - Actions
    @IBAction func refreshTapped(_ sender: UIBarButtonItem) {
        // 處理刷新
    }
}
```

### Outlet Collection

當有多個相似元件時：

```swift
class ViewController: UIViewController {
    @IBOutlet var starButtons: [UIButton]!
    
    @IBAction func starTapped(_ sender: UIButton) {
        guard let index = starButtons.firstIndex(of: sender) else { return }
        updateStars(rating: index + 1)
    }
    
    func updateStars(rating: Int) {
        for (index, button) in starButtons.enumerated() {
            button.isSelected = index < rating
        }
    }
}
```

## 小結

Interface Builder 和 Storyboard 提供視覺化的 UI 設計方式。IBOutlet 連結元件到程式碼，IBAction 處理使用者互動。Auto Layout 和 Stack View 讓佈局能適應不同螢幕。Segue 管理畫面轉場。雖然視覺化工具方便，但許多專案仍選擇純程式碼建立 UI，以提升團隊協作效率。

下週將學習 Table View 和 Collection View，這是 iOS 最常用的資料呈現方式。
