---
layout: post
title: "iOS 架構模式：MVC 與 MVVM"
date: 2018-06-08
categories: [iOS, Swift]
tags: [Swift, iOS, Architecture, MVC, MVVM, Design Pattern]
---

這週深入研究 iOS 的架構模式。在 Java 企業開發中，我們有成熟的分層架構：Controller、Service、DAO、Entity。但 iOS app 的架構思維不太一樣，需要重新理解。

## 為什麼架構很重要

剛開始寫 iOS 時，我把所有邏輯都塞在 ViewController 裡：網路請求、資料處理、UI 更新、商業邏輯全部混在一起。結果一個 ViewController 超過 1000 行，難以測試和維護。

這個問題在業界被戲稱為 **Massive View Controller**，是 iOS 開發的經典痛點。

好的架構能帶來：
- **可測試性**：商業邏輯和 UI 分離，容易寫單元測試
- **可維護性**：職責清楚，修改不會牽一髮動全身
- **可重用性**：元件獨立，能在不同地方重用
- **團隊協作**：不同人負責不同層次，減少衝突

這和 Java 的 Spring MVC 或分層架構的目標一致，但實踐方式不同。

## MVC：Apple 的官方模式

Apple 推薦的是 **MVC（Model-View-Controller）** 模式，但它和傳統 Web 開發的 MVC 有些不同。

### iOS MVC 的三個角色

**Model（模型）**
- 負責資料和商業邏輯
- 不知道 View 和 Controller 的存在
- 純粹的 Swift struct 或 class

**View（視圖）**
- 負責顯示和使用者互動
- 可以是 UIView、UIButton、自訂控制項
- 不包含商業邏輯，只處理顯示

**Controller（控制器）**
- 負責協調 Model 和 View
- 回應使用者操作，更新 Model
- 監聽 Model 變化，更新 View

理論上很美好，但實務上 **Controller 很容易變得臃腫**，因為：
- UIViewController 本身就是 View 的一部分
- 網路請求、資料轉換常常寫在 Controller
- UI 邏輯和商業邏輯難以分離

### MVC 實作範例

一個簡單的使用者資料展示：

```swift
// MARK: - Model
struct User {
    let id: Int
    let name: String
    let email: String
    let avatarURL: String?
}

class UserService {
    func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void) {
        // 網路請求邏輯
        let url = URL(string: "https://api.example.com/users/\(id)")!
        
        URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NSError(domain: "", code: -1)))
                return
            }
            
            do {
                let user = try JSONDecoder().decode(User.self, from: data)
                completion(.success(user))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}

// MARK: - View
class UserProfileView: UIView {
    let nameLabel = UILabel()
    let emailLabel = UILabel()
    let avatarImageView = UIImageView()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupUI()
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        setupUI()
    }
    
    private func setupUI() {
        addSubview(avatarImageView)
        addSubview(nameLabel)
        addSubview(emailLabel)
        
        // 設定 Auto Layout...
    }
    
    func configure(with user: User) {
        nameLabel.text = user.name
        emailLabel.text = user.email
        
        // 載入頭像
        if let urlString = user.avatarURL,
           let url = URL(string: urlString) {
            loadImage(from: url)
        }
    }
    
    private func loadImage(from url: URL) {
        URLSession.shared.dataTask(with: url) { [weak self] data, _, _ in
            if let data = data, let image = UIImage(data: data) {
                DispatchQueue.main.async {
                    self?.avatarImageView.image = image
                }
            }
        }.resume()
    }
}

// MARK: - Controller
class UserProfileViewController: UIViewController {
    private let profileView = UserProfileView()
    private let userService = UserService()
    private var user: User?
    
    override func loadView() {
        view = profileView
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        loadUserData()
    }
    
    private func loadUserData() {
        showLoadingIndicator()
        
        userService.fetchUser(id: 123) { [weak self] result in
            DispatchQueue.main.async {
                self?.hideLoadingIndicator()
                
                switch result {
                case .success(let user):
                    self?.user = user
                    self?.profileView.configure(with: user)
                    
                case .failure(let error):
                    self?.showError(error)
                }
            }
        }
    }
    
    private func showLoadingIndicator() {
        // 顯示載入動畫
    }
    
    private func hideLoadingIndicator() {
        // 隱藏載入動畫
    }
    
    private func showError(_ error: Error) {
        let alert = UIAlertController(
            title: "錯誤",
            message: error.localizedDescription,
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "確定", style: .default))
        present(alert, animated: true)
    }
}
```

這個實作已經比較乾淨，把 View 獨立出來。但 **Controller 還是很重**，包含了網路請求處理、錯誤處理、UI 狀態管理。

## MVVM：更現代的選擇

**MVVM（Model-View-ViewModel）** 是 MVC 的改進版，在 iOS 社群越來越流行。

### MVVM 的四個角色

**Model**：和 MVC 一樣，純資料

**View**：UIView 和 UIViewController 都算 View，只負責顯示

**ViewModel**：
- 處理 View 的邏輯
- 包含 View 需要的資料（以屬性形式）
- 不依賴 UIKit，可以純粹用 Swift 實作

**Coordinator（可選）**：
- 負責導航流程
- 讓 ViewController 不需要知道其他 ViewController

MVVM 的優點是 **ViewModel 完全獨立於 UIKit**，可以輕鬆做單元測試。這在 Java 開發中類似 Service 層，不依賴 Servlet API。

### MVVM 實作範例

改寫上面的例子：

```swift
// MARK: - Model（不變）
struct User {
    let id: Int
    let name: String
    let email: String
    let avatarURL: String?
}

// MARK: - ViewModel
class UserProfileViewModel {
    // 可觀察的屬性（簡化版，實務上用 Combine 或 RxSwift）
    var user: User? {
        didSet {
            onUserUpdated?()
        }
    }
    
    var isLoading: Bool = false {
        didSet {
            onLoadingStateChanged?(isLoading)
        }
    }
    
    var errorMessage: String? {
        didSet {
            if let error = errorMessage {
                onError?(error)
            }
        }
    }
    
    // 回呼閉包
    var onUserUpdated: (() -> Void)?
    var onLoadingStateChanged: ((Bool) -> Void)?
    var onError: ((String) -> Void)?
    
    private let userService: UserService
    private let userID: Int
    
    init(userID: Int, userService: UserService = UserService()) {
        self.userID = userID
        self.userService = userService
    }
    
    func loadUser() {
        isLoading = true
        
        userService.fetchUser(id: userID) { [weak self] result in
            DispatchQueue.main.async {
                self?.isLoading = false
                
                switch result {
                case .success(let user):
                    self?.user = user
                    
                case .failure(let error):
                    self?.errorMessage = error.localizedDescription
                }
            }
        }
    }
    
    // View 需要的格式化資料
    var displayName: String {
        user?.name ?? "載入中..."
    }
    
    var displayEmail: String {
        user?.email ?? ""
    }
    
    var hasAvatar: Bool {
        user?.avatarURL != nil
    }
}

// MARK: - View/Controller
class UserProfileViewController: UIViewController {
    private let profileView = UserProfileView()
    private let viewModel: UserProfileViewModel
    
    init(viewModel: UserProfileViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("用 init(viewModel:) 初始化")
    }
    
    override func loadView() {
        view = profileView
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        bindViewModel()
        viewModel.loadUser()
    }
    
    private func bindViewModel() {
        // 綁定 ViewModel 的變化到 View
        viewModel.onUserUpdated = { [weak self] in
            guard let self = self, let user = self.viewModel.user else { return }
            self.profileView.configure(with: user)
        }
        
        viewModel.onLoadingStateChanged = { [weak self] isLoading in
            if isLoading {
                self?.showLoadingIndicator()
            } else {
                self?.hideLoadingIndicator()
            }
        }
        
        viewModel.onError = { [weak self] message in
            self?.showError(message)
        }
    }
    
    private func showLoadingIndicator() {
        // 顯示載入動畫
    }
    
    private func hideLoadingIndicator() {
        // 隱藏載入動畫
    }
    
    private func showError(_ message: String) {
        let alert = UIAlertController(
            title: "錯誤",
            message: message,
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "確定", style: .default))
        present(alert, animated: true)
    }
}
```

現在 **ViewController 變得很薄**，只負責：
1. 綁定 ViewModel 到 View
2. 顯示系統 UI（Alert、Loading）

所有商業邏輯都在 ViewModel，可以輕鬆測試：

```swift
func testLoadUser() {
    let mockService = MockUserService()
    let viewModel = UserProfileViewModel(userID: 1, userService: mockService)
    
    var userLoaded = false
    viewModel.onUserUpdated = {
        userLoaded = true
    }
    
    viewModel.loadUser()
    
    XCTAssertTrue(userLoaded)
    XCTAssertNotNil(viewModel.user)
    XCTAssertEqual(viewModel.displayName, "Test User")
}
```

這個測試**完全不需要 UIKit**，可以在任何環境執行，速度也快。

## 資料綁定：RxSwift 響應式程式設計

上面的 ViewModel 用閉包綁定，比較陽春。如果想要更優雅的資料綁定，可以考慮 **RxSwift**，這是 ReactiveX 的 Swift 實作，提供響應式程式設計：

```swift
import RxSwift
import RxCocoa

class UserProfileViewModel {
    // 用 BehaviorRelay 取代普通屬性
    let user = BehaviorRelay<User?>(value: nil)
    let isLoading = BehaviorRelay<Bool>(value: false)
    let errorMessage = PublishRelay<String>()
    
    private let userService: UserService
    private let userID: Int
    private let disposeBag = DisposeBag()
    
    init(userID: Int, userService: UserService = UserService()) {
        self.userID = userID
        self.userService = userService
    }
    
    func loadUser() {
        isLoading.accept(true)
        
        userService.fetchUser(id: userID) { [weak self] result in
            DispatchQueue.main.async {
                self?.isLoading.accept(false)
                
                switch result {
                case .success(let user):
                    self?.user.accept(user)
                case .failure(let error):
                    self?.errorMessage.accept(error.localizedDescription)
                }
            }
        }
    }
}

// 在 ViewController 中綁定
class UserProfileViewController: UIViewController {
    private let viewModel: UserProfileViewModel
    private let disposeBag = DisposeBag()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 觀察 user 變化
        viewModel.user
            .filter { $0 != nil }  // 過濾 nil
            .subscribe(onNext: { [weak self] user in
                self?.profileView.configure(with: user!)
            })
            .disposed(by: disposeBag)
        
        // 觀察 loading 狀態
        viewModel.isLoading
            .subscribe(onNext: { [weak self] isLoading in
                if isLoading {
                    self?.showLoadingIndicator()
                } else {
                    self?.hideLoadingIndicator()
                }
            })
            .disposed(by: disposeBag)
    }
}
```

RxSwift 的優點：
- **宣告式**：描述資料流而不是手動管理
- **自動記憶體管理**：disposeBag 釋放時自動取消訂閱
- **運算子豐富**：map、filter、debounce、throttle 等
- **跨平台**：ReactiveX 有 Java (RxJava)、JavaScript (RxJS) 等版本

這對我從 Java 轉過來特別親切，因為 RxJava 在 Android 開發很流行。概念是相通的。

## Coordinator 模式：管理導航流程

在 MVVM 中，ViewController 不應該知道其他 ViewController，否則耦合度太高。**Coordinator** 負責管理導航：

```swift
protocol Coordinator {
    var navigationController: UINavigationController { get }
    func start()
}

class UserCoordinator: Coordinator {
    let navigationController: UINavigationController
    
    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }
    
    func start() {
        showUserList()
    }
    
    func showUserList() {
        let viewModel = UserListViewModel()
        viewModel.onUserSelected = { [weak self] userID in
            self?.showUserDetail(userID: userID)
        }
        
        let viewController = UserListViewController(viewModel: viewModel)
        navigationController.pushViewController(viewController, animated: true)
    }
    
    func showUserDetail(userID: Int) {
        let viewModel = UserProfileViewModel(userID: userID)
        let viewController = UserProfileViewController(viewModel: viewModel)
        navigationController.pushViewController(viewController, animated: true)
    }
}
```

現在 ViewController 完全不知道導航邏輯，ViewModel 透過閉包告訴 Coordinator「使用者點了某個項目」，由 Coordinator 決定要導航到哪裡。

這個模式類似 Java Web 的 FrontController，統一處理路由和導航。

## 依賴注入

MVVM 的另一個好處是容易實作依賴注入：

```swift
protocol UserServiceProtocol {
    func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void)
}

class RealUserService: UserServiceProtocol {
    func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void) {
        // 真實的網路請求
    }
}

class MockUserService: UserServiceProtocol {
    func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void) {
        // 回傳假資料
        let user = User(id: 1, name: "Mock User", email: "mock@example.com", avatarURL: nil)
        completion(.success(user))
    }
}

// ViewModel 依賴介面而非實作
class UserProfileViewModel {
    private let userService: UserServiceProtocol
    
    init(userID: Int, userService: UserServiceProtocol) {
        self.userID = userID
        self.userService = userService
    }
}

// 使用時注入實作
let realViewModel = UserProfileViewModel(userID: 1, userService: RealUserService())
let mockViewModel = UserProfileViewModel(userID: 1, userService: MockUserService())
```

這和 Spring 的依賴注入完全一樣的思維：**依賴抽象而不是具體實作**。

## MVC vs MVVM 的選擇

什麼時候用 MVC：
- 小型、簡單的畫面
- 邏輯很少，主要是顯示
- 快速原型開發

什麼時候用 MVVM：
- 複雜的商業邏輯
- 需要高可測試性
- 多人協作的大型專案

我的經驗是：**專案初期用 MVC 快速迭代，穩定後逐步重構成 MVVM**。過早優化會拖慢開發速度。

## 資料夾結構

一個典型的 MVVM 專案結構：

```
MyApp/
├── App/
│   ├── AppDelegate.swift
│   └── SceneDelegate.swift
├── Models/
│   ├── User.swift
│   └── Post.swift
├── ViewModels/
│   ├── UserListViewModel.swift
│   └── UserProfileViewModel.swift
├── Views/
│   ├── UserListViewController.swift
│   ├── UserProfileViewController.swift
│   └── Components/
│       └── UserCell.swift
├── Services/
│   ├── NetworkService.swift
│   ├── UserService.swift
│   └── AuthService.swift
├── Coordinators/
│   └── UserCoordinator.swift
└── Utilities/
    ├── Extensions/
    └── Constants.swift
```

清楚的資料夾結構讓團隊成員快速找到檔案，這在 Java 專案中也是標準做法。

## 小結

這週研究架構模式，最大的收穫是理解了 **職責分離的重要性**。無論是 MVC 還是 MVVM，核心思想都是：

1. **Model**：專注於資料和邏輯
2. **View**：專注於顯示
3. **中間層**：協調兩者

這和 Java 的分層架構異曲同工。MVVM 引入 ViewModel 層，讓商業邏輯脫離 UI 框架，提升可測試性。

關鍵要點：
- **避免 Massive View Controller**：盡量精簡 ViewController
- **依賴注入**：讓元件易於測試和替換
- **單一職責**：每個類別只做一件事
- **循序漸進**：不要過早優化，根據需求選擇架構

下週計劃研究單元測試和 UI 測試，包括 XCTest、Mock、測試覆蓋率等。測試是確保程式碼品質的關鍵。
