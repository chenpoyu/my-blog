---
layout: post
title: "iOS 網路請求與 JSON 解析"
date: 2018-03-23
categories: [Swift, iOS]
tags: [iOS, Networking, JSON, URLSession]
---

這週學習 iOS 的網路程式設計。作為 Java 後端工程師，我習慣的是「伺服器端」的思維：用 Spring MVC 接受 HTTP 請求、處理業務邏輯、回傳 JSON 響應。

現在要寫「客戶端」，角色完全反過來：發送 HTTP 請求、等待響應、解析 JSON、更新 UI。這個視角轉換很有趣，讓我更理解客戶端開發者的痛點。

iOS 的網路 API 已經進化到第三代：
- **NSURLConnection** （舊）：複雜且難用
- **NSURLSession** （現在）：iOS 7 引入，更現代化的 API
- **第三方庫** （Alamofire 等）：進一步封裝

我先從 URLSession 開始學，這是 Apple 官方推薦的方式，也是理解 iOS 網路機制的基礎。

## URLSession 基礎

URLSession 是 iOS 進行網路請求的標準 API，取代舊的 NSURLConnection。

### GET 請求

```swift
func fetchData() {
    guard let url = URL(string: "https://api.example.com/users") else {
        print("Invalid URL")
        return
    }
    
    let task = URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            print("Error: \(error.localizedDescription)")
            return
        }
        
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            print("Invalid response")
            return
        }
        
        guard let data = data else {
            print("No data")
            return
        }
        
        // 處理資料
        if let jsonString = String(data: data, encoding: .utf8) {
            print(jsonString)
        }
    }
    
    task.resume()
}
```

### POST 請求

```swift
func postData() {
    guard let url = URL(string: "https://api.example.com/users") else { return }
    
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    
    let parameters: [String: Any] = [
        "name": "John Doe",
        "email": "john@example.com",
        "age": 30
    ]
    
    do {
        request.httpBody = try JSONSerialization.data(withJSONObject: parameters)
    } catch {
        print("Failed to serialize JSON")
        return
    }
    
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            print("Error: \(error)")
            return
        }
        
        if let data = data, let responseString = String(data: data, encoding: .utf8) {
            print("Response: \(responseString)")
        }
    }
    
    task.resume()
}
```

### 自訂 URLRequest

```swift
func makeRequest() {
    guard let url = URL(string: "https://api.example.com/data") else { return }
    
    var request = URLRequest(url: url)
    request.httpMethod = "GET"
    request.timeoutInterval = 30
    request.cachePolicy = .reloadIgnoringLocalCacheData
    
    // 設定 Headers
    request.setValue("Bearer token123", forHTTPHeaderField: "Authorization")
    request.setValue("application/json", forHTTPHeaderField: "Accept")
    request.setValue("MyApp/1.0", forHTTPHeaderField: "User-Agent")
    
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        // 處理回應
    }
    
    task.resume()
}
```

## JSON 解析

### JSONSerialization

```swift
func parseJSON(_ data: Data) {
    do {
        if let json = try JSONSerialization.jsonObject(with: data) as? [String: Any] {
            // 解析物件
            if let name = json["name"] as? String {
                print("Name: \(name)")
            }
            
            if let age = json["age"] as? Int {
                print("Age: \(age)")
            }
            
            if let tags = json["tags"] as? [String] {
                print("Tags: \(tags)")
            }
        } else if let jsonArray = try JSONSerialization.jsonObject(with: data) as? [[String: Any]] {
            // 解析陣列
            for item in jsonArray {
                if let name = item["name"] as? String {
                    print("Name: \(name)")
                }
            }
        }
    } catch {
        print("JSON parsing error: \(error)")
    }
}
```

### Codable 協定

Swift 4 引入 Codable，簡化 JSON 編解碼：

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String
    let age: Int
}

func decodeUser(from data: Data) {
    let decoder = JSONDecoder()
    
    do {
        let user = try decoder.decode(User.self, from: data)
        print("User: \(user.name), Email: \(user.email)")
    } catch {
        print("Decoding error: \(error)")
    }
}

func decodeUsers(from data: Data) {
    let decoder = JSONDecoder()
    
    do {
        let users = try decoder.decode([User].self, from: data)
        for user in users {
            print("\(user.name) - \(user.email)")
        }
    } catch {
        print("Decoding error: \(error)")
    }
}
```

### CodingKeys

當 JSON 欄位名稱與屬性名稱不同時：

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String
    let isActive: Bool
    
    enum CodingKeys: String, CodingKey {
        case id
        case name
        case email
        case isActive = "is_active"  // JSON 中是 snake_case
    }
}
```

### 巢狀結構

```swift
struct Response: Codable {
    let status: String
    let data: DataContent
    
    struct DataContent: Codable {
        let users: [User]
        let totalCount: Int
        
        enum CodingKeys: String, CodingKey {
            case users
            case totalCount = "total_count"
        }
    }
}

func parseResponse(from data: Data) {
    let decoder = JSONDecoder()
    
    do {
        let response = try decoder.decode(Response.self, from: data)
        print("Status: \(response.status)")
        print("Total users: \(response.data.totalCount)")
        
        for user in response.data.users {
            print(user.name)
        }
    } catch {
        print("Error: \(error)")
    }
}
```

### 日期處理

```swift
struct Post: Codable {
    let id: Int
    let title: String
    let createdAt: Date
    
    enum CodingKeys: String, CodingKey {
        case id
        case title
        case createdAt = "created_at"
    }
}

func decodePost(from data: Data) {
    let decoder = JSONDecoder()
    
    // ISO8601 格式
    decoder.dateDecodingStrategy = .iso8601
    
    // 自訂格式
    let dateFormatter = DateFormatter()
    dateFormatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
    decoder.dateDecodingStrategy = .formatted(dateFormatter)
    
    // Unix timestamp
    decoder.dateDecodingStrategy = .secondsSince1970
    
    do {
        let post = try decoder.decode(Post.self, from: data)
        print("Post: \(post.title), Created: \(post.createdAt)")
    } catch {
        print("Error: \(error)")
    }
}
```

### Encoding

```swift
struct CreateUserRequest: Codable {
    let name: String
    let email: String
    let age: Int
}

func createUser() {
    let user = CreateUserRequest(name: "John", email: "john@example.com", age: 30)
    
    let encoder = JSONEncoder()
    encoder.outputFormatting = .prettyPrinted
    
    do {
        let jsonData = try encoder.encode(user)
        
        if let jsonString = String(data: jsonData, encoding: .utf8) {
            print(jsonString)
        }
        
        // 發送請求
        sendRequest(with: jsonData)
    } catch {
        print("Encoding error: \(error)")
    }
}

func sendRequest(with data: Data) {
    guard let url = URL(string: "https://api.example.com/users") else { return }
    
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = data
    
    URLSession.shared.dataTask(with: request) { data, response, error in
        // 處理回應
    }.resume()
}
```

## 網路請求封裝

### 建立網路管理類

```swift
class NetworkManager {
    static let shared = NetworkManager()
    private init() {}
    
    func request<T: Codable>(url: String,
                            method: HTTPMethod = .get,
                            parameters: [String: Any]? = nil,
                            completion: @escaping (Result<T, Error>) -> Void) {
        guard let url = URL(string: url) else {
            completion(.failure(NetworkError.invalidURL))
            return
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        if let parameters = parameters, method != .get {
            do {
                request.httpBody = try JSONSerialization.data(withJSONObject: parameters)
            } catch {
                completion(.failure(error))
                return
            }
        }
        
        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NetworkError.noData))
                return
            }
            
            do {
                let decoder = JSONDecoder()
                let result = try decoder.decode(T.self, from: data)
                completion(.success(result))
            } catch {
                completion(.failure(error))
            }
        }
        
        task.resume()
    }
}

enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
}

enum NetworkError: Error {
    case invalidURL
    case noData
    case decodingError
}
```

### 使用網路管理類

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

class UserService {
    func fetchUsers(completion: @escaping (Result<[User], Error>) -> Void) {
        NetworkManager.shared.request(url: "https://api.example.com/users") { (result: Result<[User], Error>) in
            completion(result)
        }
    }
    
    func createUser(name: String, email: String, completion: @escaping (Result<User, Error>) -> Void) {
        let parameters: [String: Any] = [
            "name": name,
            "email": email
        ]
        
        NetworkManager.shared.request(
            url: "https://api.example.com/users",
            method: .post,
            parameters: parameters
        ) { (result: Result<User, Error>) in
            completion(result)
        }
    }
}

class ViewController: UIViewController {
    let userService = UserService()
    
    func loadUsers() {
        userService.fetchUsers { result in
            DispatchQueue.main.async {
                switch result {
                case .success(let users):
                    print("Loaded \(users.count) users")
                    self.updateUI(with: users)
                case .failure(let error):
                    print("Error: \(error)")
                    self.showError(error)
                }
            }
        }
    }
    
    func updateUI(with users: [User]) {
        // 更新 UI
    }
    
    func showError(_ error: Error) {
        // 顯示錯誤訊息
    }
}
```

## 圖片下載

### 基本下載

```swift
func downloadImage(from urlString: String, completion: @escaping (UIImage?) -> Void) {
    guard let url = URL(string: urlString) else {
        completion(nil)
        return
    }
    
    URLSession.shared.dataTask(with: url) { data, response, error in
        guard let data = data, error == nil else {
            completion(nil)
            return
        }
        
        let image = UIImage(data: data)
        DispatchQueue.main.async {
            completion(image)
        }
    }.resume()
}
```

### 圖片快取

```swift
class ImageCache {
    static let shared = ImageCache()
    private var cache = NSCache<NSString, UIImage>()
    
    private init() {
        cache.countLimit = 100
        cache.totalCostLimit = 50 * 1024 * 1024  // 50 MB
    }
    
    func image(for url: String) -> UIImage? {
        return cache.object(forKey: url as NSString)
    }
    
    func setImage(_ image: UIImage, for url: String) {
        cache.setObject(image, forKey: url as NSString)
    }
}

class ImageLoader {
    static let shared = ImageLoader()
    
    func loadImage(from urlString: String, completion: @escaping (UIImage?) -> Void) {
        // 檢查快取
        if let cachedImage = ImageCache.shared.image(for: urlString) {
            completion(cachedImage)
            return
        }
        
        // 下載圖片
        guard let url = URL(string: urlString) else {
            completion(nil)
            return
        }
        
        URLSession.shared.dataTask(with: url) { data, response, error in
            guard let data = data,
                  let image = UIImage(data: data) else {
                DispatchQueue.main.async {
                    completion(nil)
                }
                return
            }
            
            // 儲存到快取
            ImageCache.shared.setImage(image, for: urlString)
            
            DispatchQueue.main.async {
                completion(image)
            }
        }.resume()
    }
}
```

### UIImageView 擴展

```swift
extension UIImageView {
    func loadImage(from urlString: String, placeholder: UIImage? = nil) {
        image = placeholder
        
        ImageLoader.shared.loadImage(from: urlString) { [weak self] image in
            self?.image = image ?? placeholder
        }
    }
}

// 使用
let imageView = UIImageView()
imageView.loadImage(from: "https://example.com/photo.jpg", placeholder: UIImage(named: "placeholder"))
```

## 實作 API 客戶端

```swift
// Models
struct Post: Codable {
    let id: Int
    let title: String
    let body: String
    let userId: Int
}

struct Comment: Codable {
    let id: Int
    let postId: Int
    let name: String
    let email: String
    let body: String
}

// API Service
class APIService {
    static let shared = APIService()
    private let baseURL = "https://jsonplaceholder.typicode.com"
    
    func fetchPosts(completion: @escaping (Result<[Post], Error>) -> Void) {
        let urlString = "\(baseURL)/posts"
        
        guard let url = URL(string: urlString) else {
            completion(.failure(NetworkError.invalidURL))
            return
        }
        
        URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NetworkError.noData))
                return
            }
            
            do {
                let posts = try JSONDecoder().decode([Post].self, from: data)
                completion(.success(posts))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
    
    func fetchPost(id: Int, completion: @escaping (Result<Post, Error>) -> Void) {
        let urlString = "\(baseURL)/posts/\(id)"
        
        guard let url = URL(string: urlString) else {
            completion(.failure(NetworkError.invalidURL))
            return
        }
        
        URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NetworkError.noData))
                return
            }
            
            do {
                let post = try JSONDecoder().decode(Post.self, from: data)
                completion(.success(post))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
    
    func createPost(title: String, body: String, userId: Int, completion: @escaping (Result<Post, Error>) -> Void) {
        let urlString = "\(baseURL)/posts"
        
        guard let url = URL(string: urlString) else {
            completion(.failure(NetworkError.invalidURL))
            return
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        let parameters: [String: Any] = [
            "title": title,
            "body": body,
            "userId": userId
        ]
        
        do {
            request.httpBody = try JSONSerialization.data(withJSONObject: parameters)
        } catch {
            completion(.failure(error))
            return
        }
        
        URLSession.shared.dataTask(with: request) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            
            guard let data = data else {
                completion(.failure(NetworkError.noData))
                return
            }
            
            do {
                let post = try JSONDecoder().decode(Post.self, from: data)
                completion(.success(post))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}

// ViewController
class PostsViewController: UIViewController {
    let tableView = UITableView()
    var posts: [Post] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Posts"
        setupTableView()
        loadPosts()
    }
    
    func setupTableView() {
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        view.addSubview(tableView)
    }
    
    func loadPosts() {
        APIService.shared.fetchPosts { [weak self] result in
            DispatchQueue.main.async {
                switch result {
                case .success(let posts):
                    self?.posts = posts
                    self?.tableView.reloadData()
                case .failure(let error):
                    print("Error loading posts: \(error)")
                }
            }
        }
    }
}

extension PostsViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return posts.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        let post = posts[indexPath.row]
        cell.textLabel?.text = post.title
        return cell
    }
}

extension PostsViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let post = posts[indexPath.row]
        let detailVC = PostDetailViewController()
        detailVC.post = post
        navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

## 錯誤處理

```swift
enum APIError: Error {
    case invalidURL
    case noData
    case decodingError
    case serverError(Int)
    case unknown
    
    var localizedDescription: String {
        switch self {
        case .invalidURL:
            return "無效的 URL"
        case .noData:
            return "沒有收到資料"
        case .decodingError:
            return "資料解析失敗"
        case .serverError(let code):
            return "伺服器錯誤: \(code)"
        case .unknown:
            return "未知錯誤"
        }
    }
}

func handleError(_ error: Error) {
    if let apiError = error as? APIError {
        showAlert(message: apiError.localizedDescription)
    } else {
        showAlert(message: error.localizedDescription)
    }
}
```

## 小結

URLSession 是 iOS 網路請求的核心 API，支援各種 HTTP 方法。Codable 協定大幅簡化 JSON 編解碼工作，相較於手動解析更安全且易維護。封裝網路層可以統一處理錯誤、認證等共通邏輯。圖片快取避免重複下載，提升使用者體驗。

下週將學習本地資料儲存，包含 UserDefaults、檔案系統和 Core Data。
