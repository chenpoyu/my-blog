---
layout: post
title: "iOS App 架構模式：從 MVC 到 VIPER"
date: 2018-07-13
categories: [iOS, Architecture]
tags: [MVC, MVVM, VIPER, Architecture]
---

這週研究 iOS app 的架構模式，發現架構設計對專案的可維護性影響巨大。從 Java 的分層架構轉到 iOS，需要理解不同的架構模式和適用場景。

## iOS 開發中的 Massive View Controller 問題

剛開始寫 iOS 時，我把所有邏輯都寫在 ViewController：

```swift
class ProductListViewController: UIViewController {
    @IBOutlet weak var tableView: UITableView!
    
    var products: [Product] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 網路請求
        URLSession.shared.dataTask(with: URL(string: "https://api.example.com/products")!) { data, _, error in
            guard let data = data else { return }
            
            // JSON 解析
            let decoder = JSONDecoder()
            self.products = try! decoder.decode([Product].self, from: data)
            
            // 更新 UI
            DispatchQueue.main.async {
                self.tableView.reloadData()
            }
        }.resume()
    }
    
    // TableView DataSource
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return products.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        let product = products[indexPath.row]
        
        // Cell 設定邏輯
        cell.textLabel?.text = product.name
        cell.detailTextLabel?.text = "$\(product.price)"
        
        return cell
    }
    
    // TableView Delegate
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let product = products[indexPath.row]
        
        // 導航邏輯
        let detailVC = ProductDetailViewController()
        detailVC.product = product
        navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

這個 ViewController 混雜了：
- 網路請求
- 資料解析
- UI 更新
- 使用者互動處理
- 導航邏輯

結果檔案越來越長，動輒上千行，難以測試和維護。這就是 **Massive View Controller** 問題。

在 Java 專案中，我們有清楚的分層：Controller → Service → DAO，每層職責明確。iOS 需要類似的架構思維。

## MVC：Apple 的官方架構

Apple 推薦的 MVC（Model-View-Controller）架構：

```
View ←→ Controller ←→ Model
```

理論上：
- **Model**：資料和業務邏輯
- **View**：UI 顯示
- **Controller**：協調 Model 和 View

但實際上，iOS 的 ViewController 同時負責 Controller 和 View 的職責，因為 View 的生命週期由 ViewController 管理。

**改進版的 MVC**：

```swift
// Model
struct Product: Decodable {
    let id: Int
    let name: String
    let price: Double
}

// Service（負責網路請求和資料處理）
class ProductService {
    func fetchProducts(completion: @escaping (Result<[Product], Error>) -> Void) {
        guard let url = URL(string: "https://api.example.com/products") else {
            completion(.failure(NSError(domain: "Invalid URL", code: 0)))
            return
        }
        
        URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NSError(domain: "No data", code: 0)))
                return
            }
            
            do {
                let products = try JSONDecoder().decode([Product].self, from: data)
                completion(.success(products))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}

// Controller
class ProductListViewController: UIViewController {
    @IBOutlet weak var tableView: UITableView!
    
    private let service = ProductService()
    private var products: [Product] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        loadProducts()
    }
    
    private func loadProducts() {
        service.fetchProducts { [weak self] result in
            switch result {
            case .success(let products):
                self?.products = products
                DispatchQueue.main.async {
                    self?.tableView.reloadData()
                }
            case .failure(let error):
                self?.showError(error)
            }
        }
    }
    
    private func showError(_ error: Error) {
        let alert = UIAlertController(title: "Error", message: error.localizedDescription, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }
}

extension ProductListViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return products.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        let product = products[indexPath.row]
        cell.textLabel?.text = product.name
        cell.detailTextLabel?.text = "$\(product.price)"
        return cell
    }
}
```

這個版本把網路請求邏輯移到 `ProductService`，ViewController 只負責協調。但 ViewController 還是很肥，而且**難以測試**（因為依賴 UIKit）。

## MVVM：可測試的架構

MVVM（Model-View-ViewModel）把展示邏輯從 ViewController 中抽離：

```
View ← ViewModel ← Model
```

**關鍵概念**：
- **ViewModel** 不依賴 UIKit，純 Swift 實作
- **ViewModel** 負責展示邏輯和狀態管理
- **View** 只負責顯示 ViewModel 提供的資料

```swift
// ViewModel
class ProductListViewModel {
    private let service: ProductService
    private(set) var products: [Product] = []
    
    // 用閉包通知 View 更新
    var onProductsUpdated: (() -> Void)?
    var onError: ((Error) -> Void)?
    
    init(service: ProductService = ProductService()) {
        self.service = service
    }
    
    func loadProducts() {
        service.fetchProducts { [weak self] result in
            switch result {
            case .success(let products):
                self?.products = products
                self?.onProductsUpdated?()
            case .failure(let error):
                self?.onError?(error)
            }
        }
    }
    
    // 展示邏輯
    func numberOfProducts() -> Int {
        return products.count
    }
    
    func product(at index: Int) -> Product {
        return products[index]
    }
    
    func displayText(for index: Int) -> (title: String, subtitle: String) {
        let product = products[index]
        return (product.name, "$\(product.price)")
    }
}

// ViewController（變得更輕量）
class ProductListViewController: UIViewController {
    @IBOutlet weak var tableView: UITableView!
    
    private let viewModel = ProductListViewModel()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        bindViewModel()
        viewModel.loadProducts()
    }
    
    private func bindViewModel() {
        viewModel.onProductsUpdated = { [weak self] in
            DispatchQueue.main.async {
                self?.tableView.reloadData()
            }
        }
        
        viewModel.onError = { [weak self] error in
            DispatchQueue.main.async {
                self?.showError(error)
            }
        }
    }
    
    private func showError(_ error: Error) {
        let alert = UIAlertController(title: "Error", message: error.localizedDescription, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }
}

extension ProductListViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return viewModel.numberOfProducts()
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        let (title, subtitle) = viewModel.displayText(for: indexPath.row)
        cell.textLabel?.text = title
        cell.detailTextLabel?.text = subtitle
        return cell
    }
}
```

**MVVM 的優點**：
1. **可測試性高**：ViewModel 不依賴 UIKit，可以純 Swift 測試

```swift
class ProductListViewModelTests: XCTestCase {
    func testLoadProducts() {
        let mockService = MockProductService()
        let viewModel = ProductListViewModel(service: mockService)
        
        let expectation = self.expectation(description: "Products loaded")
        
        viewModel.onProductsUpdated = {
            expectation.fulfill()
        }
        
        viewModel.loadProducts()
        
        waitForExpectations(timeout: 1) { _ in
            XCTAssertEqual(viewModel.numberOfProducts(), 2)
        }
    }
}
```

2. **職責分離**：ViewController 只負責 UI，ViewModel 負責邏輯
3. **重用性**：ViewModel 可以用在不同的 View（比如 iPhone 和 iPad 不同的 UI）

**MVVM 的缺點**：
- 資料綁定需要手動實作（用閉包或 RxSwift）
- 複雜畫面的 ViewModel 還是會很肥

## MVVM + Coordinator：處理導航邏輯

MVVM 的一個問題是 ViewController 還要處理導航：

```swift
func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    let product = viewModel.product(at: indexPath.row)
    let detailVC = ProductDetailViewController()
    detailVC.product = product
    navigationController?.pushViewController(detailVC, animated: true)
}
```

**Coordinator 模式**把導航邏輯抽離出來：

```swift
protocol Coordinator {
    func start()
}

class ProductCoordinator: Coordinator {
    private let navigationController: UINavigationController
    
    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }
    
    func start() {
        let viewModel = ProductListViewModel()
        let viewController = ProductListViewController(viewModel: viewModel)
        viewController.onProductSelected = { [weak self] product in
            self?.showProductDetail(product)
        }
        navigationController.pushViewController(viewController, animated: true)
    }
    
    private func showProductDetail(_ product: Product) {
        let detailViewModel = ProductDetailViewModel(product: product)
        let detailVC = ProductDetailViewController(viewModel: detailViewModel)
        navigationController.pushViewController(detailVC, animated: true)
    }
}

// ViewController 不再處理導航
class ProductListViewController: UIViewController {
    var onProductSelected: ((Product) -> Void)?
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let product = viewModel.product(at: indexPath.row)
        onProductSelected?(product)  // 通知 Coordinator
    }
}
```

這類似 Java Spring MVC 的架構思維：Controller 不直接處理視圖導航，而是回傳邏輯視圖名稱，由 ViewResolver 處理。

## VIPER：超清晰的職責分離

VIPER 是 iOS 中最複雜的架構模式：

- **View**：UIViewController，只負責顯示
- **Interactor**：業務邏輯
- **Presenter**：展示邏輯，連接 View 和 Interactor
- **Entity**：資料模型
- **Router**：導航邏輯

```swift
// Entity
struct Product {
    let id: Int
    let name: String
    let price: Double
}

// Interactor（業務邏輯）
protocol ProductListInteractorInput {
    func fetchProducts()
}

protocol ProductListInteractorOutput: AnyObject {
    func didFetchProducts(_ products: [Product])
    func didFailFetchProducts(_ error: Error)
}

class ProductListInteractor: ProductListInteractorInput {
    weak var output: ProductListInteractorOutput?
    private let service: ProductService
    
    init(service: ProductService = ProductService()) {
        self.service = service
    }
    
    func fetchProducts() {
        service.fetchProducts { [weak self] result in
            switch result {
            case .success(let products):
                self?.output?.didFetchProducts(products)
            case .failure(let error):
                self?.output?.didFailFetchProducts(error)
            }
        }
    }
}

// Presenter（展示邏輯）
protocol ProductListPresenterInput {
    func viewDidLoad()
    func didSelectProduct(at index: Int)
}

protocol ProductListPresenterOutput: AnyObject {
    func showProducts(_ products: [(title: String, subtitle: String)])
    func showError(_ message: String)
}

class ProductListPresenter: ProductListPresenterInput {
    weak var view: ProductListPresenterOutput?
    var interactor: ProductListInteractorInput?
    var router: ProductListRouter?
    
    private var products: [Product] = []
    
    func viewDidLoad() {
        interactor?.fetchProducts()
    }
    
    func didSelectProduct(at index: Int) {
        let product = products[index]
        router?.navigateToDetail(product: product)
    }
}

extension ProductListPresenter: ProductListInteractorOutput {
    func didFetchProducts(_ products: [Product]) {
        self.products = products
        let displayData = products.map { (title: $0.name, subtitle: "$\($0.price)") }
        view?.showProducts(displayData)
    }
    
    func didFailFetchProducts(_ error: Error) {
        view?.showError(error.localizedDescription)
    }
}

// View（ViewController）
class ProductListViewController: UIViewController, ProductListPresenterOutput {
    var presenter: ProductListPresenterInput?
    private var displayData: [(title: String, subtitle: String)] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        presenter?.viewDidLoad()
    }
    
    func showProducts(_ products: [(title: String, subtitle: String)]) {
        self.displayData = products
        DispatchQueue.main.async {
            self.tableView.reloadData()
        }
    }
    
    func showError(_ message: String) {
        // 顯示錯誤訊息
    }
}

// Router（導航邏輯）
class ProductListRouter {
    weak var viewController: UIViewController?
    
    func navigateToDetail(product: Product) {
        let detailVC = ProductDetailBuilder.build(product: product)
        viewController?.navigationController?.pushViewController(detailVC, animated: true)
    }
}

// Builder（組裝模組）
class ProductListBuilder {
    static func build() -> UIViewController {
        let view = ProductListViewController()
        let presenter = ProductListPresenter()
        let interactor = ProductListInteractor()
        let router = ProductListRouter()
        
        view.presenter = presenter
        presenter.view = view
        presenter.interactor = interactor
        presenter.router = router
        interactor.output = presenter
        router.viewController = view
        
        return view
    }
}
```

**VIPER 的優點**：
- 職責極度分離，每個模組只做一件事
- 非常容易測試，每個模組都可以獨立測試
- 適合大型專案和團隊協作

**VIPER 的缺點**：
- 程式碼量大，簡單畫面也需要很多檔案
- 學習曲線陡峭
- 過度設計（overengineering）的風險

## 架構選擇的考量

**小型專案（個人 side project）**：
- 用改進版的 MVC 就夠了
- 把 Service 層抽離出來
- 不需要過度設計

**中型專案（小團隊）**：
- MVVM 是最佳選擇
- 配合 Coordinator 處理導航
- 測試重要的業務邏輯

**大型專案（多團隊）**：
- VIPER 或類似的清晰架構
- 每個模組獨立開發和測試
- 嚴格的 code review 和架構守則

## 與 Java 架構的對比

Java 企業開發的分層架構：

```
Controller → Service → DAO → Database
```

對應到 iOS：

```
ViewController → ViewModel/Presenter → Service → Network/Database
```

**關鍵差異**：
- Java 的 Controller 不處理 View，iOS 的 ViewController 必須處理
- Java 有成熟的 DI 框架（Spring），iOS 需要手動管理依賴
- Java 側重資料流，iOS 側重狀態管理和 UI 更新

## 學習心得

從 Java 轉到 iOS，最大的挑戰是理解**狀態管理**。Java 後端是無狀態的（stateless），每個請求獨立處理。iOS app 是有狀態的（stateful），需要管理 View 的狀態、資料的快取、使用者的互動。

架構的選擇沒有銀彈，重點是：
1. **職責分離**：每個模組只做一件事
2. **可測試性**：能獨立測試業務邏輯
3. **可維護性**：新人能快速理解程式碼結構

我現在的專案用 MVVM + Coordinator，平衡了複雜度和可維護性。未來如果專案更大，會考慮 VIPER 或 Clean Architecture。
