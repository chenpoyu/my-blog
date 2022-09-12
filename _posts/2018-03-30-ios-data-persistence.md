---
layout: post
title: "iOS 資料持久化"
date: 2018-03-30
categories: [Swift, iOS]
tags: [iOS, Data Persistence, UserDefaults, Core Data]
---

這週研究 iOS 的資料儲存方式。在 Java 開發中，我們通常用資料庫（MySQL、PostgreSQL）來儲存資料，或者用檔案系統。行動應用的資料儲存則有不同的考量。

iOS 提供了多種資料持久化方式，適用於不同的場景：

- **UserDefaults**：儲存簡單的設定值（類似 Java 的 Properties 檔）
- **檔案系統**：儲存圖片、文件等檔案
- **Core Data**：物件關係對應框架（類似 Java 的 Hibernate）
- **SQLite**：直接操作 SQL 資料庫
- **Keychain**：儲存敏感資訊（密碼、Token）

這週先學 UserDefaults 和 Core Data，這兩個是最常用的。UserDefaults 簡單但功能有限，Core Data 強大但學習曲線陡。

## UserDefaults

UserDefaults 用於儲存簡單的鍵值對資料，適合設定、偏好等輕量資料。

### 基本使用

```swift
class SettingsManager {
    static let shared = SettingsManager()
    private let defaults = UserDefaults.standard
    
    // 儲存資料
    func saveUsername(_ username: String) {
        defaults.set(username, forKey: "username")
    }
    
    func saveAge(_ age: Int) {
        defaults.set(age, forKey: "age")
    }
    
    func saveIsLoggedIn(_ isLoggedIn: Bool) {
        defaults.set(isLoggedIn, forKey: "isLoggedIn")
    }
    
    // 讀取資料
    func getUsername() -> String? {
        return defaults.string(forKey: "username")
    }
    
    func getAge() -> Int {
        return defaults.integer(forKey: "age")
    }
    
    func getIsLoggedIn() -> Bool {
        return defaults.bool(forKey: "isLoggedIn")
    }
    
    // 刪除資料
    func removeUsername() {
        defaults.removeObject(forKey: "username")
    }
    
    // 清空所有資料
    func resetAll() {
        if let bundleID = Bundle.main.bundleIdentifier {
            defaults.removePersistentDomain(forName: bundleID)
        }
    }
}
```

### 支援的資料型別

```swift
// 基本型別
defaults.set(123, forKey: "number")
defaults.set(3.14, forKey: "pi")
defaults.set(true, forKey: "flag")
defaults.set("text", forKey: "string")

// 集合型別
defaults.set([1, 2, 3], forKey: "numbers")
defaults.set(["key": "value"], forKey: "dict")

// Date
defaults.set(Date(), forKey: "lastLogin")

// Data
let data = "Hello".data(using: .utf8)
defaults.set(data, forKey: "data")

// 讀取
let number = defaults.integer(forKey: "number")
let pi = defaults.double(forKey: "pi")
let flag = defaults.bool(forKey: "flag")
let text = defaults.string(forKey: "string")
let numbers = defaults.array(forKey: "numbers") as? [Int]
let dict = defaults.dictionary(forKey: "dict")
let date = defaults.object(forKey: "lastLogin") as? Date
```

### 自訂型別儲存

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

extension UserDefaults {
    func set<T: Codable>(_ object: T, forKey key: String) {
        let encoder = JSONEncoder()
        if let encoded = try? encoder.encode(object) {
            set(encoded, forKey: key)
        }
    }
    
    func get<T: Codable>(_ type: T.Type, forKey key: String) -> T? {
        guard let data = data(forKey: key) else { return nil }
        let decoder = JSONDecoder()
        return try? decoder.decode(T.self, from: data)
    }
}

// 使用
let user = User(id: 1, name: "John", email: "john@example.com")
UserDefaults.standard.set(user, forKey: "currentUser")

if let savedUser = UserDefaults.standard.get(User.self, forKey: "currentUser") {
    print("Saved user: \(savedUser.name)")
}
```

### 屬性包裝器

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    
    var wrappedValue: T {
        get {
            return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
}

class Settings {
    @UserDefault(key: "username", defaultValue: "")
    static var username: String
    
    @UserDefault(key: "isDarkMode", defaultValue: false)
    static var isDarkMode: Bool
    
    @UserDefault(key: "fontSize", defaultValue: 16)
    static var fontSize: Int
}

// 使用
Settings.username = "John"
print(Settings.username)

Settings.isDarkMode = true
print(Settings.isDarkMode)
```

## 檔案系統

### 取得目錄路徑

```swift
class FileManager {
    // Documents 目錄 - 使用者產生的內容
    static func documentsDirectory() -> URL {
        return FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
    }
    
    // Caches 目錄 - 可被清除的快取
    static func cachesDirectory() -> URL {
        return FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
    }
    
    // Temp 目錄 - 臨時檔案
    static func tempDirectory() -> URL {
        return FileManager.default.temporaryDirectory
    }
}
```

### 寫入檔案

```swift
class FileStorage {
    static func saveText(_ text: String, filename: String) -> Bool {
        let fileURL = FileManager.documentsDirectory().appendingPathComponent(filename)
        
        do {
            try text.write(to: fileURL, atomically: true, encoding: .utf8)
            return true
        } catch {
            print("Error writing file: \(error)")
            return false
        }
    }
    
    static func saveData(_ data: Data, filename: String) -> Bool {
        let fileURL = FileManager.documentsDirectory().appendingPathComponent(filename)
        
        do {
            try data.write(to: fileURL)
            return true
        } catch {
            print("Error writing data: \(error)")
            return false
        }
    }
    
    static func saveImage(_ image: UIImage, filename: String) -> Bool {
        guard let data = image.jpegData(compressionQuality: 0.8) else {
            return false
        }
        return saveData(data, filename: filename)
    }
}
```

### 讀取檔案

```swift
extension FileStorage {
    static func loadText(filename: String) -> String? {
        let fileURL = FileManager.documentsDirectory().appendingPathComponent(filename)
        
        do {
            return try String(contentsOf: fileURL, encoding: .utf8)
        } catch {
            print("Error reading file: \(error)")
            return nil
        }
    }
    
    static func loadData(filename: String) -> Data? {
        let fileURL = FileManager.documentsDirectory().appendingPathComponent(filename)
        
        do {
            return try Data(contentsOf: fileURL)
        } catch {
            print("Error reading data: \(error)")
            return nil
        }
    }
    
    static func loadImage(filename: String) -> UIImage? {
        guard let data = loadData(filename: filename) else {
            return nil
        }
        return UIImage(data: data)
    }
}
```

### 檔案操作

```swift
extension FileStorage {
    static func fileExists(filename: String) -> Bool {
        let fileURL = FileManager.documentsDirectory().appendingPathComponent(filename)
        return FileManager.default.fileExists(atPath: fileURL.path)
    }
    
    static func deleteFile(filename: String) -> Bool {
        let fileURL = FileManager.documentsDirectory().appendingPathComponent(filename)
        
        do {
            try FileManager.default.removeItem(at: fileURL)
            return true
        } catch {
            print("Error deleting file: \(error)")
            return false
        }
    }
    
    static func listFiles() -> [String] {
        let documentsURL = FileManager.documentsDirectory()
        
        do {
            let fileURLs = try FileManager.default.contentsOfDirectory(
                at: documentsURL,
                includingPropertiesForKeys: nil
            )
            return fileURLs.map { $0.lastPathComponent }
        } catch {
            print("Error listing files: \(error)")
            return []
        }
    }
    
    static func fileSize(filename: String) -> Int64? {
        let fileURL = FileManager.documentsDirectory().appendingPathComponent(filename)
        
        do {
            let attributes = try FileManager.default.attributesOfItem(atPath: fileURL.path)
            return attributes[.size] as? Int64
        } catch {
            return nil
        }
    }
}
```

### JSON 檔案

```swift
struct Note: Codable {
    let id: String
    let title: String
    let content: String
    let createdAt: Date
}

class NoteStorage {
    static func saveNotes(_ notes: [Note]) -> Bool {
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601
        
        do {
            let data = try encoder.encode(notes)
            return FileStorage.saveData(data, filename: "notes.json")
        } catch {
            print("Encoding error: \(error)")
            return false
        }
    }
    
    static func loadNotes() -> [Note]? {
        guard let data = FileStorage.loadData(filename: "notes.json") else {
            return nil
        }
        
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        
        do {
            return try decoder.decode([Note].self, from: data)
        } catch {
            print("Decoding error: \(error)")
            return nil
        }
    }
}
```

## Keychain

Keychain 用於儲存敏感資料，如密碼、Token 等。

### 基本 Keychain 操作

```swift
import Security

class KeychainManager {
    static func save(key: String, data: Data) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]
        
        SecItemDelete(query as CFDictionary)
        
        let status = SecItemAdd(query as CFDictionary, nil)
        return status == errSecSuccess
    }
    
    static func load(key: String) -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        return status == errSecSuccess ? result as? Data : nil
    }
    
    static func delete(key: String) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        
        let status = SecItemDelete(query as CFDictionary)
        return status == errSecSuccess
    }
}

// 使用
extension KeychainManager {
    static func savePassword(_ password: String, for account: String) -> Bool {
        guard let data = password.data(using: .utf8) else { return false }
        return save(key: account, data: data)
    }
    
    static func loadPassword(for account: String) -> String? {
        guard let data = load(key: account) else { return nil }
        return String(data: data, encoding: .utf8)
    }
}

// 儲存
KeychainManager.savePassword("myPassword123", for: "user@example.com")

// 讀取
if let password = KeychainManager.loadPassword(for: "user@example.com") {
    print("Password: \(password)")
}
```

## Core Data 基礎

Core Data 是 Apple 的物件關係對應框架，適合複雜的資料結構。

### 建立 Data Model

1. New File -> Data Model
2. 建立 Entity (例如: Person)
3. 新增 Attributes (例如: name, age, email)

### Core Data Stack

```swift
import CoreData

class CoreDataManager {
    static let shared = CoreDataManager()
    
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "MyApp")
        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Unable to load persistent stores: \(error)")
            }
        }
        return container
    }()
    
    var context: NSManagedObjectContext {
        return persistentContainer.viewContext
    }
    
    func saveContext() {
        if context.hasChanges {
            do {
                try context.save()
            } catch {
                print("Error saving context: \(error)")
            }
        }
    }
}
```

### CRUD 操作

```swift
// Create
func createPerson(name: String, age: Int, email: String) {
    let context = CoreDataManager.shared.context
    let person = Person(context: context)
    person.name = name
    person.age = Int16(age)
    person.email = email
    
    CoreDataManager.shared.saveContext()
}

// Read
func fetchAllPersons() -> [Person] {
    let context = CoreDataManager.shared.context
    let fetchRequest: NSFetchRequest<Person> = Person.fetchRequest()
    
    do {
        return try context.fetch(fetchRequest)
    } catch {
        print("Error fetching persons: \(error)")
        return []
    }
}

// Read with predicate
func fetchPersons(olderThan age: Int) -> [Person] {
    let context = CoreDataManager.shared.context
    let fetchRequest: NSFetchRequest<Person> = Person.fetchRequest()
    fetchRequest.predicate = NSPredicate(format: "age > %d", age)
    fetchRequest.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
    
    do {
        return try context.fetch(fetchRequest)
    } catch {
        print("Error: \(error)")
        return []
    }
}

// Update
func updatePerson(_ person: Person, newName: String) {
    person.name = newName
    CoreDataManager.shared.saveContext()
}

// Delete
func deletePerson(_ person: Person) {
    let context = CoreDataManager.shared.context
    context.delete(person)
    CoreDataManager.shared.saveContext()
}
```

### 實作筆記應用

```swift
// Note Entity: title (String), content (String), createdAt (Date)

class NoteManager {
    static let shared = NoteManager()
    private let context = CoreDataManager.shared.context
    
    func createNote(title: String, content: String) {
        let note = Note(context: context)
        note.title = title
        note.content = content
        note.createdAt = Date()
        
        CoreDataManager.shared.saveContext()
    }
    
    func fetchAllNotes() -> [Note] {
        let fetchRequest: NSFetchRequest<Note> = Note.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        
        do {
            return try context.fetch(fetchRequest)
        } catch {
            print("Error fetching notes: \(error)")
            return []
        }
    }
    
    func searchNotes(keyword: String) -> [Note] {
        let fetchRequest: NSFetchRequest<Note> = Note.fetchRequest()
        fetchRequest.predicate = NSPredicate(
            format: "title CONTAINS[cd] %@ OR content CONTAINS[cd] %@",
            keyword, keyword
        )
        
        do {
            return try context.fetch(fetchRequest)
        } catch {
            print("Error: \(error)")
            return []
        }
    }
    
    func updateNote(_ note: Note, title: String, content: String) {
        note.title = title
        note.content = content
        CoreDataManager.shared.saveContext()
    }
    
    func deleteNote(_ note: Note) {
        context.delete(note)
        CoreDataManager.shared.saveContext()
    }
}

class NotesViewController: UIViewController {
    let tableView = UITableView()
    var notes: [Note] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Notes"
        setupTableView()
        setupNavigationBar()
        loadNotes()
    }
    
    func setupNavigationBar() {
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .add,
            target: self,
            action: #selector(addNote)
        )
    }
    
    func loadNotes() {
        notes = NoteManager.shared.fetchAllNotes()
        tableView.reloadData()
    }
    
    @objc func addNote() {
        let alert = UIAlertController(title: "New Note", message: nil, preferredStyle: .alert)
        alert.addTextField { $0.placeholder = "Title" }
        alert.addTextField { $0.placeholder = "Content" }
        
        alert.addAction(UIAlertAction(title: "Cancel", style: .cancel))
        alert.addAction(UIAlertAction(title: "Save", style: .default) { _ in
            guard let title = alert.textFields?[0].text,
                  let content = alert.textFields?[1].text,
                  !title.isEmpty else { return }
            
            NoteManager.shared.createNote(title: title, content: content)
            self.loadNotes()
        })
        
        present(alert, animated: true)
    }
    
    func setupTableView() {
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        view.addSubview(tableView)
    }
}

extension NotesViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return notes.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        let note = notes[indexPath.row]
        cell.textLabel?.text = note.title
        cell.detailTextLabel?.text = note.content
        return cell
    }
    
    func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle, forRowAt indexPath: IndexPath) {
        if editingStyle == .delete {
            let note = notes[indexPath.row]
            NoteManager.shared.deleteNote(note)
            notes.remove(at: indexPath.row)
            tableView.deleteRows(at: [indexPath], with: .fade)
        }
    }
}

extension NotesViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let note = notes[indexPath.row]
        let detailVC = NoteDetailViewController()
        detailVC.note = note
        navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

## 選擇合適的儲存方式

```
UserDefaults:
- 簡單的設定、偏好
- 少量資料 (<1MB)
- 不敏感的資料

檔案系統:
- 大型檔案 (圖片、影片、文件)
- 結構化的 JSON 資料
- 需要完全控制檔案的場景

Keychain:
- 密碼、Token
- 敏感資料
- 需要加密的資料

Core Data:
- 複雜的資料結構
- 需要關聯查詢
- 大量結構化資料
- 需要資料同步
```

## 小結

iOS 提供多種資料持久化方案。UserDefaults 適合簡單設定，檔案系統處理大型檔案，Keychain 保護敏感資料，Core Data 管理複雜資料結構。選擇合適的方案取決於資料特性和需求。下週將學習多執行緒和 GCD，處理非同步任務。
