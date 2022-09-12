---
layout: post
title: "iOS 單元測試與 UI 測試"
date: 2018-06-15
categories: [iOS, Swift]
tags: [Swift, iOS, Testing, XCTest, Unit Test, UI Test]
---

這週深入研究測試。在 Java 開發中，我習慣用 JUnit 和 Mockito 寫測試，測試驅動開發（TDD）也是團隊的標準實踐。轉到 iOS 後，發現 Apple 提供的 XCTest 框架功能同樣完整，但使用方式有些不同。

## 為什麼要寫測試

剛開始寫 iOS 時，我覺得「app 這麼小，手動測試就夠了」。但當專案變大、功能變多，每次修改都要手動測試所有功能變得不切實際。自動化測試能：

**提早發現問題**：在開發階段就抓到 bug，而不是等使用者回報。我遇過一個案例，一個小改動導致登入功能失效，但因為沒寫測試，直到上線後才被發現。如果有測試覆蓋，這種問題在 commit 時就會被抓出來。

**重構時更有信心**：當我想優化程式碼或調整架構，有完整的測試套件作為安全網。改完跑測試，全綠就代表功能沒被破壞。這在 Java 專案重構時也是同樣的道理。

**文件化程式行為**：測試用例本身就是最好的文件，它清楚展示「這個功能應該怎麼運作」。新加入的團隊成員看測試就能理解業務邏輯。

**加速開發**：雖然寫測試要時間，但長期來看能節省手動測試和 debug 的時間。特別是迴歸測試，自動化比人工快太多。

## XCTest 框架基礎

iOS 使用 **XCTest** 作為官方測試框架。建立測試目標時，Xcode 會自動產生測試檔案：

```swift
import XCTest
@testable import MyApp  // 讓測試能存取 app 的 internal 成員

class MyAppTests: XCTestCase {
    
    // 每個測試方法執行前呼叫
    override func setUp() {
        super.setUp()
        // 初始化測試環境
    }
    
    // 每個測試方法執行後呼叫
    override func tearDown() {
        // 清理測試環境
        super.tearDown()
    }
    
    // 測試方法必須以 test 開頭
    func testExample() {
        // 測試程式碼
        XCTAssertTrue(true)
    }
}
```

`@testable import` 是關鍵，它讓測試能存取 app 中標記為 `internal` 的類別和方法。這類似 Java 的 package-private 存取層級。

測試方法的命名慣例是 `test + 描述`，比如 `testUserLoginWithValidCredentials`。清楚的命名能讓測試失敗時快速理解問題所在。

## 斷言方法

XCTest 提供豐富的斷言方法：

```swift
func testAssertions() {
    // 基本斷言
    XCTAssertTrue(1 < 2, "1 應該小於 2")
    XCTAssertFalse(1 > 2)
    
    // 相等性斷言
    XCTAssertEqual(2 + 2, 4)
    XCTAssertNotEqual(2 + 2, 5)
    
    // Nil 檢查
    let value: String? = nil
    XCTAssertNil(value)
    XCTAssertNotNil("non-nil value")
    
    // 浮點數比較（考慮精度）
    XCTAssertEqual(0.1 + 0.2, 0.3, accuracy: 0.0001)
    
    // 拋出錯誤
    XCTAssertThrowsError(try riskyFunction())
    
    // 不拋出錯誤
    XCTAssertNoThrow(try safeFunction())
}
```

每個斷言的最後一個參數都能加上描述訊息，測試失敗時會顯示。這個習慣在 JUnit 中也很重要，清楚的錯誤訊息能大幅減少 debug 時間。

## 測試非同步程式碼

iOS 開發中大量使用非同步操作：網路請求、動畫完成、資料庫查詢。測試這些需要特殊處理：

```swift
func testAsyncNetworkRequest() {
    // 建立期望
    let expectation = self.expectation(description: "網路請求完成")
    
    let service = UserService()
    service.fetchUser(id: 123) { result in
        switch result {
        case .success(let user):
            XCTAssertEqual(user.name, "Test User")
            XCTAssertNotNil(user.email)
            
        case .failure(let error):
            XCTFail("請求失敗：\(error)")
        }
        
        // 標記期望完成
        expectation.fulfill()
    }
    
    // 等待期望完成，最多等 5 秒
    waitForExpectations(timeout: 5.0) { error in
        if let error = error {
            XCTFail("超時：\(error)")
        }
    }
}
```

`expectation` 機制告訴測試「等待某個非同步操作完成」。如果在 timeout 時間內 `fulfill()` 沒被呼叫，測試就會失敗。

這個模式在測試 Java 的 CompletableFuture 或 RxJava 時也常用到，只是語法不同。

### 測試多個非同步操作

如果有多個非同步操作需要等待：

```swift
func testMultipleAsyncOperations() {
    let exp1 = expectation(description: "第一個操作")
    let exp2 = expectation(description: "第二個操作")
    
    service.operation1 {
        XCTAssertTrue($0)
        exp1.fulfill()
    }
    
    service.operation2 {
        XCTAssertTrue($0)
        exp2.fulfill()
    }
    
    // 等待所有期望完成
    wait(for: [exp1, exp2], timeout: 5.0)
}
```

## Mock 和 Stub

測試時應該隔離外部依賴，比如網路、資料庫、第三方服務。這時需要 Mock 或 Stub。

### 使用 Protocol 實現 Mock

```swift
// 定義協定
protocol UserServiceProtocol {
    func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void)
}

// 真實實作
class RealUserService: UserServiceProtocol {
    func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void) {
        // 實際的網路請求
        URLSession.shared.dataTask(with: url) { data, response, error in
            // 處理回應...
        }.resume()
    }
}

// Mock 實作（用於測試）
class MockUserService: UserServiceProtocol {
    var shouldSucceed = true
    var mockUser: User?
    var fetchUserCallCount = 0
    
    func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void) {
        fetchUserCallCount += 1
        
        if shouldSucceed {
            let user = mockUser ?? User(id: id, name: "Mock User", email: "mock@test.com", avatarURL: nil)
            completion(.success(user))
        } else {
            completion(.failure(NSError(domain: "test", code: -1)))
        }
    }
}
```

在測試中使用 Mock：

```swift
func testViewModelWithMockService() {
    // 準備 mock service
    let mockService = MockUserService()
    mockService.mockUser = User(id: 1, name: "John", email: "john@test.com", avatarURL: nil)
    
    // 注入 mock
    let viewModel = UserProfileViewModel(userID: 1, userService: mockService)
    
    // 設定期望
    let expectation = self.expectation(description: "使用者載入完成")
    viewModel.onUserUpdated = {
        expectation.fulfill()
    }
    
    // 執行測試
    viewModel.loadUser()
    
    waitForExpectations(timeout: 1.0)
    
    // 驗證結果
    XCTAssertEqual(viewModel.user?.name, "John")
    XCTAssertEqual(mockService.fetchUserCallCount, 1, "應該只呼叫一次")
}
```

這個模式和 Java 的 Mockito 概念相同，只是 Swift 用 protocol 而不是動態代理。

### 驗證互動

除了驗證結果，有時還要驗證「某個方法被呼叫了」或「呼叫順序正確」：

```swift
class MockAnalytics {
    private(set) var loggedEvents: [(name: String, parameters: [String: Any])] = []
    
    func logEvent(_ name: String, parameters: [String: Any] = [:]) {
        loggedEvents.append((name, parameters))
    }
}

func testAnalyticsLogging() {
    let mockAnalytics = MockAnalytics()
    let viewController = MyViewController(analytics: mockAnalytics)
    
    viewController.performAction()
    
    // 驗證事件被記錄
    XCTAssertEqual(mockAnalytics.loggedEvents.count, 1)
    XCTAssertEqual(mockAnalytics.loggedEvents.first?.name, "button_clicked")
}
```

## 測試覆蓋率

Xcode 能顯示測試覆蓋率，幫助找出未測試的程式碼：

1. **Edit Scheme → Test → Options**
2. 勾選 **Code Coverage**
3. 執行測試後，在 Report Navigator 查看覆蓋率報告

覆蓋率報告會標示：
- 綠色：已測試的程式碼
- 紅色：未測試的程式碼
- 數字：該行被執行的次數

但要注意，**高覆蓋率不等於測試品質好**。100% 覆蓋率的程式碼可能只是執行過但沒真正驗證行為。重要的是測試關鍵邏輯和邊界情況。

在 Java 專案中我們用 JaCoCo 追蹤覆蓋率，目標通常是 80% 以上。但經驗告訴我，與其追求數字，不如專注於測試核心業務邏輯。

## UI 測試

單元測試驗證邏輯，UI 測試則驗證使用者介面和互動流程。XCTest 也支援 UI 測試：

```swift
import XCTest

class MyAppUITests: XCTestCase {
    var app: XCUIApplication!
    
    override func setUp() {
        super.setUp()
        
        continueAfterFailure = false  // 失敗後停止
        app = XCUIApplication()
        app.launch()
    }
    
    func testLoginFlow() {
        // 找到 UI 元素
        let emailField = app.textFields["email"]
        let passwordField = app.secureTextFields["password"]
        let loginButton = app.buttons["登入"]
        
        // 檢查元素存在
        XCTAssertTrue(emailField.exists)
        XCTAssertTrue(passwordField.exists)
        
        // 輸入文字
        emailField.tap()
        emailField.typeText("test@example.com")
        
        passwordField.tap()
        passwordField.typeText("password123")
        
        // 點擊按鈕
        loginButton.tap()
        
        // 驗證導航到新畫面
        let welcomeLabel = app.staticTexts["歡迎回來"]
        XCTAssertTrue(welcomeLabel.waitForExistence(timeout: 5))
    }
    
    func testTableViewScroll() {
        let table = app.tables.firstMatch
        
        // 滾動到底部
        let lastCell = table.cells.element(boundBy: table.cells.count - 1)
        lastCell.swipeUp()
        
        // 驗證元素可見
        XCTAssertTrue(lastCell.isHittable)
    }
}
```

### UI 測試的查詢方式

XCTest 提供多種方式找到 UI 元素：

```swift
// 透過 accessibility identifier
app.buttons["loginButton"]

// 透過文字內容
app.buttons["登入"]

// 透過類型
app.textFields.firstMatch

// 使用 predicate 查詢
let predicate = NSPredicate(format: "label BEGINSWITH '歡迎'")
app.staticTexts.matching(predicate).firstMatch
```

為了讓 UI 測試更穩定，應該為重要元素設定 **accessibility identifier**：

```swift
// 在 ViewController 中
loginButton.accessibilityIdentifier = "loginButton"
emailTextField.accessibilityIdentifier = "emailField"
```

這樣即使按鈕文字改變（比如多語言），測試仍然能找到元素。這個概念類似 Web 測試中的 `data-testid`。

## 效能測試

XCTest 還支援效能測試，確保程式碼執行速度符合預期：

```swift
func testSortingPerformance() {
    let array = (0..<10000).map { _ in Int.random(in: 0...1000) }
    
    measure {
        // 測量這段程式碼的執行時間
        _ = array.sorted()
    }
}
```

`measure` 會執行程式碼多次並計算平均時間。Xcode 會記錄基準值，後續執行如果變慢會警告你。

這在優化演算法時很有用。比如我優化了圖片壓縮邏輯後，用效能測試驗證確實快了 30%。

## 測試驅動開發（TDD）實踐

TDD 的流程是：紅燈（寫失敗的測試）→ 綠燈（實作讓測試通過）→ 重構（優化程式碼）。

實例：開發一個計算機類別

```swift
// 先寫測試
func testAddition() {
    let calculator = Calculator()
    let result = calculator.add(2, 3)
    XCTAssertEqual(result, 5)
}

// 測試會失敗，因為 Calculator 還不存在
// 接著實作最簡單能通過測試的程式碼

class Calculator {
    func add(_ a: Int, _ b: Int) -> Int {
        return a + b
    }
}

// 測試通過後，加入更多測試

func testSubtraction() {
    let calculator = Calculator()
    XCTAssertEqual(calculator.subtract(5, 3), 2)
}

func testDivisionByZero() {
    let calculator = Calculator()
    XCTAssertThrowsError(try calculator.divide(10, 0))
}
```

TDD 的好處是：
- 確保每行程式碼都有測試覆蓋
- 設計出更易測試的 API
- 測試即文件，清楚展示使用方式

但缺點是初期會比較慢，需要習慣這個節奏。在我的 Java 專案中，TDD 在寫複雜業務邏輯時特別有效。

## 持續整合與測試自動化

測試寫好後，應該整合到 CI/CD 流程：

```yaml
# .github/workflows/test.yml
name: iOS Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: macos-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Run tests
      run: |
        xcodebuild test \
          -scheme MyApp \
          -destination 'platform=iOS Simulator,name=iPhone X' \
          -enableCodeCoverage YES
    
    - name: Upload coverage
      run: bash <(curl -s https://codecov.io/bash)
```

這樣每次 commit 都會自動執行測試，確保程式碼品質。這在團隊開發中特別重要，避免有人提交破壞現有功能的程式碼。

## 測試的最佳實踐

經過這週的深入研究，我整理出幾個關鍵原則：

**測試應該快速**：單元測試應該在幾秒內完成。如果測試跑很久，開發者就不願意常跑，失去即時回饋的價值。避免在測試中做真實的網路請求或大量 I/O 操作。

**測試應該獨立**：每個測試不應該依賴其他測試的執行結果。測試順序改變不應影響結果。這樣才能並行執行測試，加快速度。

**一個測試只驗證一件事**：不要在一個測試方法中塞太多斷言。如果測試失敗，應該能立刻知道是哪個功能有問題。

**使用描述性的測試名稱**：`testUserLogin()` 不如 `testUserLoginWithValidCredentialsSucceeds()`。清楚的名稱讓測試報告更易讀。

**不要測試實作細節**：測試應該驗證行為而非實作。如果重構內部邏輯但行為不變，測試不應該失敗。

## 小結

這週研究測試，最大的體會是**測試不只是為了找 bug，更是為了設計更好的程式碼**。

當我發現某個類別很難寫測試時，通常代表它的職責太多、依賴太複雜。這時應該重構，而不是勉強寫測試。TDD 強迫我們先思考「如何使用」再思考「如何實作」，這能產生更清晰的 API 設計。

關鍵要點：
1. **單元測試驗證邏輯**：快速、可靠、易維護
2. **UI 測試驗證流程**：模擬使用者操作
3. **Mock 隔離依賴**：用 protocol 和依賴注入
4. **測試非同步程式碼**：用 expectation 機制
5. **持續整合**：每次 commit 都跑測試

下週計劃研究除錯工具和 Instruments，包括記憶體分析、效能優化、網路監控等。這些工具能幫助找出深層的效能問題。
