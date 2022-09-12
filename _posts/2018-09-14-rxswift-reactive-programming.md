---
layout: post
title: "Reactive Programming：RxSwift 入門與實戰"
date: 2018-09-14
categories: [iOS, Reactive Programming]
tags: [RxSwift, Reactive, FRP, Asynchronous]
---

這週研究 RxSwift，響應式程式設計（Reactive Programming）的 Swift 實作。這是從 Java 的 RxJava 移植過來的概念，對非同步程式和資料流處理很有幫助。

## 為什麼需要 Reactive Programming

傳統的非同步程式碼容易變得複雜：

```swift
// 傳統做法：callback hell
fetchUser { user in
    fetchPosts(for: user) { posts in
        fetchComments(for: posts.first!) { comments in
            updateUI(with: comments)
        }
    }
}

// 或用 GCD
DispatchQueue.global().async {
    let user = self.fetchUser()
    let posts = self.fetchPosts(for: user)
    let comments = self.fetchComments(for: posts.first!)
    
    DispatchQueue.main.async {
        self.updateUI(with: comments)
    }
}
```

**問題**：
- 錯誤處理分散各處
- 難以取消或重試
- 多個非同步操作的組合很複雜

## RxSwift 的核心概念

### Observable：資料流

Observable 是可觀察的資料序列：

```swift
import RxSwift

// 建立 Observable
let numbers = Observable.of(1, 2, 3, 4, 5)

// 訂閱（Subscribe）
numbers.subscribe(onNext: { number in
    print(number)
})

// 輸出：1, 2, 3, 4, 5
```

**從各種來源建立 Observable**：

```swift
// 從陣列
let array = Observable.from([1, 2, 3, 4, 5])

// 從單一值
let single = Observable.just(42)

// 從非同步操作
let observable = Observable<String>.create { observer in
    DispatchQueue.global().async {
        let result = performNetworkRequest()
        observer.onNext(result)
        observer.onCompleted()
    }
    
    return Disposables.create()
}

// 從 Timer
let timer = Observable<Int>.interval(.seconds(1), scheduler: MainScheduler.instance)
```

### Observer：觀察者

Observer 訂閱 Observable 並處理事件：

```swift
observable.subscribe(
    onNext: { value in
        print("Got value: \(value)")
    },
    onError: { error in
        print("Got error: \(error)")
    },
    onCompleted: {
        print("Completed")
    },
    onDisposed: {
        print("Disposed")
    }
)
```

### DisposeBag：管理訂閱

避免記憶體洩漏，用 DisposeBag 管理訂閱：

```swift
class ViewController: UIViewController {
    let disposeBag = DisposeBag()
    
    func setupBindings() {
        observable
            .subscribe(onNext: { value in
                print(value)
            })
            .disposed(by: disposeBag)  // 當 ViewController 釋放時自動取消訂閱
    }
}
```

## Operators：強大的資料處理

### Map：轉換資料

```swift
Observable.of(1, 2, 3, 4, 5)
    .map { $0 * 2 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

// 輸出：2, 4, 6, 8, 10
```

### Filter：過濾資料

```swift
Observable.of(1, 2, 3, 4, 5)
    .filter { $0 % 2 == 0 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

// 輸出：2, 4
```

### FlatMap：處理巢狀 Observable

```swift
// 使用者輸入 → 搜尋 API → 結果
searchTextField.rx.text
    .orEmpty
    .debounce(.milliseconds(300), scheduler: MainScheduler.instance)  // 等待 300ms 沒輸入
    .distinctUntilChanged()  // 只在文字真的改變時觸發
    .flatMapLatest { query -> Observable<[SearchResult]> in
        if query.isEmpty {
            return .just([])
        }
        return searchAPI(query: query)
    }
    .observe(on: MainScheduler.instance)
    .subscribe(onNext: { results in
        self.updateUI(with: results)
    })
    .disposed(by: disposeBag)
```

**關鍵 operators**：
- `debounce`：延遲執行，避免頻繁觸發
- `distinctUntilChanged`：只在值改變時發送
- `flatMapLatest`：自動取消前一個請求，只保留最新的

### CombineLatest：組合多個 Observable

```swift
let username = usernameTextField.rx.text.orEmpty
let password = passwordTextField.rx.text.orEmpty

Observable.combineLatest(username, password) { username, password in
    return username.count >= 3 && password.count >= 6
}
.subscribe(onNext: { isValid in
    self.loginButton.isEnabled = isValid
})
.disposed(by: disposeBag)
```

只要任一個輸入改變，就重新計算登入按鈕是否該啟用。

## 實戰案例：搜尋功能

**需求**：使用者輸入關鍵字，即時顯示搜尋結果

**傳統做法的問題**：
- 每次輸入都發請求，浪費資源
- 需要手動管理請求取消
- 處理載入狀態和錯誤很複雜

**RxSwift 解決方案**：

```swift
class SearchViewController: UIViewController {
    @IBOutlet weak var searchBar: UISearchBar!
    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var activityIndicator: UIActivityIndicatorView!
    
    let disposeBag = DisposeBag()
    let viewModel = SearchViewModel()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupBindings()
    }
    
    func setupBindings() {
        // 搜尋輸入
        let searchText = searchBar.rx.text
            .orEmpty
            .debounce(.milliseconds(300), scheduler: MainScheduler.instance)
            .distinctUntilChanged()
        
        // 搜尋結果
        let searchResults = searchText
            .flatMapLatest { query -> Observable<[SearchResult]> in
                if query.isEmpty {
                    return .just([])
                }
                return self.viewModel.search(query: query)
                    .catch { error in
                        print("Search error: \(error)")
                        return .just([])
                    }
            }
            .share(replay: 1)  // 共享結果，避免重複請求
        
        // 綁定到 TableView
        searchResults
            .bind(to: tableView.rx.items(cellIdentifier: "Cell")) { index, result, cell in
                cell.textLabel?.text = result.title
            }
            .disposed(by: disposeBag)
        
        // 載入指示器
        searchText
            .map { !$0.isEmpty }
            .bind(to: activityIndicator.rx.isAnimating)
            .disposed(by: disposeBag)
        
        // 選擇結果
        tableView.rx.modelSelected(SearchResult.self)
            .subscribe(onNext: { result in
                self.showDetail(for: result)
            })
            .disposed(by: disposeBag)
    }
}

class SearchViewModel {
    func search(query: String) -> Observable<[SearchResult]> {
        return Observable.create { observer in
            let task = URLSession.shared.dataTask(with: URL(string: "https://api.example.com/search?q=\(query)")!) { data, response, error in
                if let error = error {
                    observer.onError(error)
                    return
                }
                
                guard let data = data else {
                    observer.onNext([])
                    observer.onCompleted()
                    return
                }
                
                do {
                    let results = try JSONDecoder().decode([SearchResult].self, from: data)
                    observer.onNext(results)
                    observer.onCompleted()
                } catch {
                    observer.onError(error)
                }
            }
            
            task.resume()
            
            return Disposables.create {
                task.cancel()  // 取消訂閱時自動取消請求
            }
        }
    }
}
```

**優點**：
- 自動處理去抖動（debounce）
- 自動取消舊請求
- 錯誤處理集中
- 程式碼簡潔易讀

## MVVM 與 RxSwift

RxSwift 和 MVVM 是天生一對：

```swift
class UserProfileViewModel {
    // Input
    let loadTrigger = PublishSubject<Void>()
    let refreshTrigger = PublishSubject<Void>()
    
    // Output
    let username: Observable<String>
    let email: Observable<String>
    let isLoading: Observable<Bool>
    let error: Observable<Error?>
    
    init(userService: UserService) {
        let activityIndicator = ActivityIndicator()
        
        let userData = Observable.merge(loadTrigger, refreshTrigger)
            .flatMapLatest {
                userService.fetchUserProfile()
                    .trackActivity(activityIndicator)
                    .catch { error in
                        return .just(User.empty)
                    }
            }
            .share(replay: 1)
        
        self.username = userData.map { $0.name }
        self.email = userData.map { $0.email }
        self.isLoading = activityIndicator.asObservable()
        self.error = Observable.empty()  // 簡化示例
    }
}

class UserProfileViewController: UIViewController {
    let disposeBag = DisposeBag()
    let viewModel = UserProfileViewModel(userService: UserService())
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Bind Input
        rx.viewDidAppear
            .map { _ in () }
            .bind(to: viewModel.loadTrigger)
            .disposed(by: disposeBag)
        
        refreshControl.rx.controlEvent(.valueChanged)
            .bind(to: viewModel.refreshTrigger)
            .disposed(by: disposeBag)
        
        // Bind Output
        viewModel.username
            .bind(to: usernameLabel.rx.text)
            .disposed(by: disposeBag)
        
        viewModel.email
            .bind(to: emailLabel.rx.text)
            .disposed(by: disposeBag)
        
        viewModel.isLoading
            .bind(to: activityIndicator.rx.isAnimating)
            .disposed(by: disposeBag)
    }
}
```

**好處**：
- ViewModel 完全不依賴 UIKit，易於測試
- 資料流向清楚：Input → ViewModel → Output
- 自動更新 UI，不用手動呼叫

## Subjects：既是 Observable 也是 Observer

### PublishSubject

```swift
let subject = PublishSubject<String>()

subject.subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

subject.onNext("Hello")  // 輸出：Hello
subject.onNext("World")  // 輸出：World
```

只發送訂閱之後的事件。

### BehaviorSubject

```swift
let subject = BehaviorSubject<String>(value: "Initial")

subject.onNext("Updated")

subject.subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 輸出：Updated（會收到最新值）

subject.onNext("New")  // 輸出：New
```

新訂閱者會立即收到最新值。

### ReplaySubject

```swift
let subject = ReplaySubject<String>.create(bufferSize: 2)

subject.onNext("1")
subject.onNext("2")
subject.onNext("3")

subject.subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
// 輸出：2, 3（最近的兩個值）
```

## Schedulers：控制執行緒

```swift
observable
    .subscribe(on: ConcurrentDispatchQueueScheduler(qos: .background))  // 在背景執行緒訂閱
    .observe(on: MainScheduler.instance)  // 在主執行緒觀察
    .subscribe(onNext: { value in
        // 更新 UI
    })
    .disposed(by: disposeBag)
```

**常用 Schedulers**：
- `MainScheduler.instance`：主執行緒
- `ConcurrentDispatchQueueScheduler`：背景執行緒
- `SerialDispatchQueueScheduler`：序列執行緒

## RxSwift vs 傳統方式

**傳統方式**：

```swift
class ViewController: UIViewController {
    var username: String = "" {
        didSet {
            validateForm()
        }
    }
    
    var password: String = "" {
        didSet {
            validateForm()
        }
    }
    
    func validateForm() {
        let isValid = username.count >= 3 && password.count >= 6
        loginButton.isEnabled = isValid
    }
}
```

**RxSwift 方式**：

```swift
Observable.combineLatest(
    usernameTextField.rx.text.orEmpty,
    passwordTextField.rx.text.orEmpty
) { username, password in
    return username.count >= 3 && password.count >= 6
}
.bind(to: loginButton.rx.isEnabled)
.disposed(by: disposeBag)
```

更簡潔，而且資料流向清楚。

## 學習心得

RxSwift 的學習曲線陡峭，一開始很難理解「everything is a stream」的概念。但掌握後，非同步程式變得更簡潔易讀。

**適合用 RxSwift 的場景**：
- 複雜的使用者輸入處理
- 多個非同步操作的組合
- 即時更新的 UI
- MVVM 架構的資料綁定

**不適合的場景**：
- 簡單的一次性請求
- 學習成本考量（團隊不熟悉）
- 效能極度敏感的場景（RxSwift 有些 overhead）

與 Java 的 RxJava 相比，RxSwift 的概念和 API 幾乎一樣，轉換很容易。但 Swift 的型別系統和閉包語法讓 Rx 程式碼更簡潔。

未來 Apple 可能會推出官方的響應式框架（傳聞中的 Combine），但 RxSwift 的概念和技巧還是很有價值的。
