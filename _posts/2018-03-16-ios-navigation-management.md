---
layout: post
title: "iOS 導航與畫面管理"
date: 2018-03-16
categories: [Swift, iOS]
tags: [iOS, Navigation, UINavigationController, UITabBarController]
---

## Navigation Controller

UINavigationController 管理階層式的畫面導航，類似網頁的前進後退。

### 基本使用

```swift
// 在 AppDelegate 或 SceneDelegate 設定
let viewController = ViewController()
let navigationController = UINavigationController(rootViewController: viewController)
window?.rootViewController = navigationController
```

### Push 和 Pop

```swift
class FirstViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        title = "First Screen"
        
        let button = UIButton(frame: CGRect(x: 100, y: 200, width: 200, height: 50))
        button.setTitle("Go to Second", for: .normal)
        button.setTitleColor(.blue, for: .normal)
        button.addTarget(self, action: #selector(goToSecond), for: .touchUpInside)
        view.addSubview(button)
    }
    
    @objc func goToSecond() {
        let secondVC = SecondViewController()
        navigationController?.pushViewController(secondVC, animated: true)
    }
}

class SecondViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Second Screen"
        view.backgroundColor = .white
    }
    
    @objc func goBack() {
        navigationController?.popViewController(animated: true)
    }
    
    @objc func goToRoot() {
        navigationController?.popToRootViewController(animated: true)
    }
}
```

### Navigation Bar 設定

```swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 標題
        title = "Home"
        
        // 大標題模式 (iOS 11+)
        navigationController?.navigationBar.prefersLargeTitles = true
        navigationItem.largeTitleDisplayMode = .always
        
        // 左側按鈕
        navigationItem.leftBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .add,
            target: self,
            action: #selector(addTapped)
        )
        
        // 右側按鈕
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            title: "Edit",
            style: .plain,
            target: self,
            action: #selector(editTapped)
        )
        
        // 多個右側按鈕
        let editButton = UIBarButtonItem(barButtonSystemItem: .edit, target: self, action: #selector(editTapped))
        let shareButton = UIBarButtonItem(barButtonSystemItem: .action, target: self, action: #selector(shareTapped))
        navigationItem.rightBarButtonItems = [editButton, shareButton]
    }
    
    @objc func addTapped() {
        print("Add tapped")
    }
    
    @objc func editTapped() {
        print("Edit tapped")
    }
    
    @objc func shareTapped() {
        print("Share tapped")
    }
}
```

### 自訂 Back 按鈕

```swift
class FirstViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 方法一：修改文字
        navigationItem.backBarButtonItem = UIBarButtonItem(
            title: "Back",
            style: .plain,
            target: nil,
            action: nil
        )
    }
}

class SecondViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 方法二：完全自訂
        let backButton = UIBarButtonItem(
            title: "< Custom",
            style: .plain,
            target: self,
            action: #selector(backTapped)
        )
        navigationItem.leftBarButtonItem = backButton
        
        // 隱藏預設的 back 按鈕
        navigationItem.hidesBackButton = true
    }
    
    @objc func backTapped() {
        navigationController?.popViewController(animated: true)
    }
}
```

### 外觀自訂

```swift
// 在 AppDelegate 或第一個 ViewController 設定
func setupNavigationBar() {
    let appearance = UINavigationBar.appearance()
    appearance.barTintColor = .blue
    appearance.tintColor = .white
    appearance.titleTextAttributes = [
        .foregroundColor: UIColor.white,
        .font: UIFont.boldSystemFont(ofSize: 18)
    ]
    appearance.isTranslucent = false
}
```

## Tab Bar Controller

UITabBarController 管理平行的多個畫面，類似桌面應用的分頁。

### 基本使用

```swift
func setupTabBarController() {
    let homeVC = HomeViewController()
    homeVC.tabBarItem = UITabBarItem(tabBarSystemItem: .favorites, tag: 0)
    
    let searchVC = SearchViewController()
    searchVC.tabBarItem = UITabBarItem(tabBarSystemItem: .search, tag: 1)
    
    let profileVC = ProfileViewController()
    profileVC.tabBarItem = UITabBarItem(tabBarSystemItem: .contacts, tag: 2)
    
    let tabBarController = UITabBarController()
    tabBarController.viewControllers = [homeVC, searchVC, profileVC]
    
    window?.rootViewController = tabBarController
}
```

### 自訂 Tab Bar Item

```swift
let homeVC = HomeViewController()
homeVC.tabBarItem = UITabBarItem(
    title: "Home",
    image: UIImage(named: "home"),
    selectedImage: UIImage(named: "home_selected")
)

let searchVC = SearchViewController()
searchVC.tabBarItem = UITabBarItem(
    title: "Search",
    image: UIImage(named: "search"),
    tag: 1
)
```

### Badge

```swift
class HomeViewController: UIViewController {
    func updateNotifications(count: Int) {
        if count > 0 {
            tabBarItem.badgeValue = "\(count)"
        } else {
            tabBarItem.badgeValue = nil
        }
    }
}
```

### Tab Bar 外觀

```swift
func setupTabBarAppearance() {
    let appearance = UITabBar.appearance()
    appearance.barTintColor = .white
    appearance.tintColor = .blue
    appearance.unselectedItemTintColor = .gray
}
```

### Tab Bar Controller Delegate

```swift
class AppTabBarController: UITabBarController, UITabBarControllerDelegate {
    override func viewDidLoad() {
        super.viewDidLoad()
        delegate = self
    }
    
    func tabBarController(_ tabBarController: UITabBarController,
                         didSelect viewController: UIViewController) {
        print("Selected tab: \(viewController.title ?? "")")
    }
    
    func tabBarController(_ tabBarController: UITabBarController,
                         shouldSelect viewController: UIViewController) -> Bool {
        // 可以攔截切換
        if viewController is ProfileViewController {
            // 檢查登入狀態
            if !isLoggedIn {
                showLoginAlert()
                return false
            }
        }
        return true
    }
}
```

## 組合 Navigation 和 Tab Bar

```swift
func setupApp() {
    // 建立 Navigation Controllers
    let homeNav = UINavigationController(rootViewController: HomeViewController())
    homeNav.tabBarItem = UITabBarItem(title: "Home", image: UIImage(named: "home"), tag: 0)
    
    let searchNav = UINavigationController(rootViewController: SearchViewController())
    searchNav.tabBarItem = UITabBarItem(title: "Search", image: UIImage(named: "search"), tag: 1)
    
    let profileNav = UINavigationController(rootViewController: ProfileViewController())
    profileNav.tabBarItem = UITabBarItem(title: "Profile", image: UIImage(named: "profile"), tag: 2)
    
    // 建立 Tab Bar Controller
    let tabBarController = UITabBarController()
    tabBarController.viewControllers = [homeNav, searchNav, profileNav]
    
    window?.rootViewController = tabBarController
}
```

## Modal Presentation

### Present 和 Dismiss

```swift
class ViewController: UIViewController {
    @objc func showModal() {
        let modalVC = ModalViewController()
        modalVC.modalPresentationStyle = .fullScreen
        present(modalVC, animated: true)
    }
}

class ModalViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let closeButton = UIBarButtonItem(
            barButtonSystemItem: .close,
            target: self,
            action: #selector(closeTapped)
        )
        navigationItem.leftBarButtonItem = closeButton
    }
    
    @objc func closeTapped() {
        dismiss(animated: true)
    }
}
```

### Presentation Style

```swift
let vc = ViewController()

// 全螢幕 (預設)
vc.modalPresentationStyle = .fullScreen

// 表單
vc.modalPresentationStyle = .formSheet

// 當前內容
vc.modalPresentationStyle = .currentContext

// Popover (iPad)
vc.modalPresentationStyle = .popover
```

### Presentation Delegate

```swift
class ParentViewController: UIViewController, UIAdaptivePresentationControllerDelegate {
    func showSheet() {
        let sheetVC = SheetViewController()
        sheetVC.presentationController?.delegate = self
        present(sheetVC, animated: true)
    }
    
    func presentationControllerDidDismiss(_ presentationController: UIPresentationController) {
        print("Sheet was dismissed")
    }
    
    func presentationControllerShouldDismiss(_ presentationController: UIPresentationController) -> Bool {
        // 可以阻止關閉
        return true
    }
}
```

## 資料傳遞

### Forward Passing (向前傳遞)

```swift
class ListViewController: UIViewController {
    func showDetail(item: String) {
        let detailVC = DetailViewController()
        detailVC.item = item
        navigationController?.pushViewController(detailVC, animated: true)
    }
}

class DetailViewController: UIViewController {
    var item: String?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        title = item
    }
}
```

### Backward Passing (向後傳遞) - Delegate

```swift
protocol DetailViewControllerDelegate: AnyObject {
    func detailViewController(_ controller: DetailViewController, didUpdate item: String)
}

class DetailViewController: UIViewController {
    weak var delegate: DetailViewControllerDelegate?
    var item: String?
    
    @objc func saveTapped() {
        delegate?.detailViewController(self, didUpdate: "Updated Item")
        navigationController?.popViewController(animated: true)
    }
}

class ListViewController: UIViewController, DetailViewControllerDelegate {
    func showDetail() {
        let detailVC = DetailViewController()
        detailVC.delegate = self
        navigationController?.pushViewController(detailVC, animated: true)
    }
    
    func detailViewController(_ controller: DetailViewController, didUpdate item: String) {
        print("Received update: \(item)")
        // 更新 UI
    }
}
```

### Closure

```swift
class DetailViewController: UIViewController {
    var onSave: ((String) -> Void)?
    
    @objc func saveTapped() {
        onSave?("Updated Item")
        navigationController?.popViewController(animated: true)
    }
}

class ListViewController: UIViewController {
    func showDetail() {
        let detailVC = DetailViewController()
        detailVC.onSave = { [weak self] item in
            print("Received: \(item)")
            self?.updateList(with: item)
        }
        navigationController?.pushViewController(detailVC, animated: true)
    }
    
    func updateList(with item: String) {
        // 更新邏輯
    }
}
```

## 實作多層級導航應用

```swift
struct Category {
    let name: String
    let items: [String]
}

class CategoryListViewController: UIViewController {
    let tableView = UITableView()
    let categories = [
        Category(name: "Fruits", items: ["Apple", "Banana", "Orange"]),
        Category(name: "Vegetables", items: ["Carrot", "Broccoli", "Spinach"]),
        Category(name: "Drinks", items: ["Water", "Juice", "Coffee"])
    ]
    
    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Categories"
        
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        view.addSubview(tableView)
    }
}

extension CategoryListViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return categories.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        cell.textLabel?.text = categories[indexPath.row].name
        cell.accessoryType = .disclosureIndicator
        return cell
    }
}

extension CategoryListViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let category = categories[indexPath.row]
        let itemListVC = ItemListViewController()
        itemListVC.category = category
        navigationController?.pushViewController(itemListVC, animated: true)
    }
}

class ItemListViewController: UIViewController {
    let tableView = UITableView()
    var category: Category?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        title = category?.name
        
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        view.addSubview(tableView)
    }
}

extension ItemListViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return category?.items.count ?? 0
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        cell.textLabel?.text = category?.items[indexPath.row]
        return cell
    }
}

extension ItemListViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        guard let item = category?.items[indexPath.row] else { return }
        
        let detailVC = ItemDetailViewController()
        detailVC.itemName = item
        navigationController?.pushViewController(detailVC, animated: true)
    }
}

class ItemDetailViewController: UIViewController {
    var itemName: String?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        title = itemName
        view.backgroundColor = .white
        
        let label = UILabel()
        label.text = "Details for \(itemName ?? "")"
        label.textAlignment = .center
        label.frame = CGRect(x: 0, y: 200, width: view.bounds.width, height: 40)
        view.addSubview(label)
    }
}
```

## 深層連結

```swift
class DeepLinkManager {
    static func handle(url: URL, navigationController: UINavigationController) {
        // URL 格式: myapp://category/fruits/item/apple
        let pathComponents = url.pathComponents
        
        if pathComponents.count >= 3 && pathComponents[1] == "category" {
            let categoryName = pathComponents[2]
            
            let categoryVC = CategoryListViewController()
            navigationController.pushViewController(categoryVC, animated: false)
            
            if pathComponents.count >= 5 && pathComponents[3] == "item" {
                let itemName = pathComponents[4]
                
                let itemDetailVC = ItemDetailViewController()
                itemDetailVC.itemName = itemName
                navigationController.pushViewController(itemDetailVC, animated: true)
            }
        }
    }
}
```

## 小結

Navigation Controller 提供階層式導航，Tab Bar Controller 管理平行畫面。兩者可以組合使用，建構複雜的應用結構。理解資料傳遞機制對於建立互動式應用至關重要。Delegate 和 Closure 是常用的向後傳遞方式。

下週將學習網路請求和 JSON 解析，開始處理遠端資料。
