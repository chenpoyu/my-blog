---
layout: post
title: "Core Data 進階：複雜資料模型與效能優化"
date: 2018-08-10
categories: [iOS, Core Data]
tags: [Core Data, Database, Persistence, Performance]
---

這週深入研究 Core Data 的進階主題，從複雜的資料關係到效能優化技巧。Core Data 不只是 ORM，更是完整的物件圖管理框架，理解其運作機制對大型 app 很重要。

## Core Data 不是資料庫

很多人（包括我）一開始把 Core Data 當成類似 Hibernate 或 JPA 的 ORM 框架，但 Apple 強調：**Core Data 不是資料庫，是物件圖管理框架**。

**Core Data 的核心概念**：
- **NSManagedObject**：管理的物件
- **NSManagedObjectContext**：物件的生命週期容器，類似「工作區」
- **NSPersistentStoreCoordinator**：協調不同的儲存後端
- **NSPersistentStore**：實際的儲存（可以是 SQLite、XML、記憶體等）

```
Application Code
       ↓
NSManagedObjectContext (工作區)
       ↓
NSPersistentStoreCoordinator (協調器)
       ↓
NSPersistentStore (SQLite / XML / In-Memory)
```

## 複雜的資料關係

### 一對多關係（One-to-Many）

```swift
// Author 有多本 Book
class Author: NSManagedObject {
    @NSManaged var name: String
    @NSManaged var books: NSSet?  // 關係
}

class Book: NSManagedObject {
    @NSManaged var title: String
    @NSManaged var author: Author?  // 反向關係
}

// 建立關係
let author = Author(context: context)
author.name = "J.K. Rowling"

let book1 = Book(context: context)
book1.title = "Harry Potter 1"
book1.author = author  // 自動維護反向關係

let book2 = Book(context: context)
book2.title = "Harry Potter 2"
book2.author = author

print(author.books?.count)  // 2
```

**Delete Rule**：刪除 Author 時 Book 怎麼辦？

- **Nullify**：Book.author 變成 nil（預設）
- **Cascade**：Book 也被刪除
- **Deny**：如果有 Book，不能刪除 Author
- **No Action**：不做任何處理（可能造成資料不一致）

### 多對多關係（Many-to-Many）

```swift
// Student 修多門 Course，Course 有多個 Student
class Student: NSManagedObject {
    @NSManaged var name: String
    @NSManaged var courses: NSSet?
}

class Course: NSManagedObject {
    @NSManaged var title: String
    @NSManaged var students: NSSet?
}

// 建立關係
let student = Student(context: context)
student.name = "Alice"

let course1 = Course(context: context)
course1.title = "iOS Development"

let course2 = Course(context: context)
course2.title = "Swift Programming"

student.addToCourses(course1)
student.addToCourses(course2)

print(student.courses?.count)  // 2
print(course1.students?.count)  // 1
```

Core Data 自動建立中間表（join table）來儲存多對多關係。

### 繼承層級

```swift
// Vehicle 是抽象類別
class Vehicle: NSManagedObject {
    @NSManaged var brand: String
}

class Car: Vehicle {
    @NSManaged var numberOfDoors: Int
}

class Motorcycle: Vehicle {
    @NSManaged var engineCC: Int
}

// 查詢所有 Vehicle（包含子類別）
let request: NSFetchRequest<Vehicle> = Vehicle.fetchRequest()
let vehicles = try context.fetch(request)  // 包含 Car 和 Motorcycle
```

Core Data 支援三種繼承策略，類似 JPA：
- **Single Table**：所有類別存在同一個表（預設）
- **Table per Class**：每個類別一個表
- **Entities**：分開儲存

## NSFetchRequest：強大的查詢 API

### 基本查詢

```swift
let request: NSFetchRequest<Book> = Book.fetchRequest()

// 條件查詢（Predicate）
request.predicate = NSPredicate(format: "title CONTAINS[cd] %@", "Harry")

// 排序
request.sortDescriptors = [NSSortDescriptor(key: "title", ascending: true)]

// 限制數量
request.fetchLimit = 10

// 執行
let books = try context.fetch(request)
```

### 進階 Predicate

```swift
// 字串比較
NSPredicate(format: "title BEGINSWITH[cd] %@", "Harry")  // 開頭為
NSPredicate(format: "title ENDSWITH[cd] %@", "Potter")  // 結尾為
NSPredicate(format: "title CONTAINS[cd] %@", "and")  // 包含
// [cd] 代表 case-insensitive 和 diacritic-insensitive

// 數字比較
NSPredicate(format: "price > %f", 10.0)
NSPredicate(format: "price BETWEEN {10.0, 50.0}")

// 日期比較
NSPredicate(format: "publishDate > %@", yesterday as NSDate)

// 關係查詢
NSPredicate(format: "author.name == %@", "J.K. Rowling")
NSPredicate(format: "SUBQUERY(books, $book, $book.price > 20).@count > 0")

// 組合條件
let predicate1 = NSPredicate(format: "title CONTAINS[cd] %@", "Harry")
let predicate2 = NSPredicate(format: "price < %f", 20.0)
let compound = NSCompoundPredicate(andPredicateWithSubpredicates: [predicate1, predicate2])
```

### 聚合查詢

```swift
// 計數
request.resultType = .countResultType
let count = try context.count(for: request)

// 聚合函數
request.resultType = .dictionaryResultType
let avgExpression = NSExpression(forFunction: "average:", arguments: [NSExpression(forKeyPath: "price")])
let avgDescription = NSExpressionDescription()
avgDescription.name = "avgPrice"
avgDescription.expression = avgExpression
avgDescription.expressionResultType = .doubleAttributeType

request.propertiesToFetch = [avgDescription]
let results = try context.fetch(request) as! [[String: Any]]
let avgPrice = results.first?["avgPrice"] as? Double
```

類似 SQL 的 `AVG(price)`。

## 效能優化技巧

### 1. Faulting：延遲載入

Core Data 預設不會立即載入所有資料，而是建立「fault」（一個佔位符）：

```swift
let author = try context.fetch(Author.fetchRequest()).first
// 此時 author.books 還沒實際載入

print(author.books?.count)  // 現在才從資料庫載入 books
```

這類似 Hibernate 的 Lazy Loading。

**控制 faulting**：

```swift
// 預先載入關係（避免 N+1 查詢）
request.relationshipKeyPathsForPrefetching = ["author", "publisher"]

let books = try context.fetch(request)
// books[0].author 已經載入，不會再查詢資料庫
```

### 2. Batch Fetching：批次查詢

```swift
request.fetchBatchSize = 20

let books = try context.fetch(request)
// 一次只載入 20 筆，當存取第 21 筆時才載入下一批
```

這對顯示大量資料的 TableView 很有用。

### 3. NSFetchedResultsController：自動更新 TableView

```swift
class BookListViewController: UITableViewController, NSFetchedResultsControllerDelegate {
    var fetchedResultsController: NSFetchedResultsController<Book>!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let request: NSFetchRequest<Book> = Book.fetchRequest()
        request.sortDescriptors = [NSSortDescriptor(key: "title", ascending: true)]
        
        fetchedResultsController = NSFetchedResultsController(
            fetchRequest: request,
            managedObjectContext: context,
            sectionNameKeyPath: nil,
            cacheName: "BookCache"
        )
        
        fetchedResultsController.delegate = self
        try? fetchedResultsController.performFetch()
    }
    
    // TableView DataSource
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return fetchedResultsController.sections?[section].numberOfObjects ?? 0
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        let book = fetchedResultsController.object(at: indexPath)
        cell.textLabel?.text = book.title
        return cell
    }
    
    // 自動更新 delegate
    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.beginUpdates()
    }
    
    func controller(_ controller: NSFetchedResultsController<NSFetchRequestResult>,
                    didChange anObject: Any,
                    at indexPath: IndexPath?,
                    for type: NSFetchedResultsChangeType,
                    newIndexPath: IndexPath?) {
        switch type {
        case .insert:
            tableView.insertRows(at: [newIndexPath!], with: .automatic)
        case .delete:
            tableView.deleteRows(at: [indexPath!], with: .automatic)
        case .update:
            tableView.reloadRows(at: [indexPath!], with: .automatic)
        case .move:
            tableView.moveRow(at: indexPath!, to: newIndexPath!)
        @unknown default:
            break
        }
    }
    
    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.endUpdates()
    }
}
```

當資料變化時，TableView 自動更新，不用手動呼叫 `reloadData()`。

### 4. Batch Update 和 Batch Delete

大量更新或刪除時，用 batch 操作避免載入所有物件到記憶體：

```swift
// Batch Delete
let fetchRequest: NSFetchRequest<NSFetchRequestResult> = Book.fetchRequest()
fetchRequest.predicate = NSPredicate(format: "publishDate < %@", oldDate as NSDate)

let batchDelete = NSBatchDeleteRequest(fetchRequest: fetchRequest)
try context.execute(batchDelete)

// Batch Update
let batchUpdate = NSBatchUpdateRequest(entityName: "Book")
batchUpdate.predicate = NSPredicate(format: "price == 0")
batchUpdate.propertiesToUpdate = ["price": 9.99]
try context.execute(batchUpdate)
```

這類似 SQL 的 `UPDATE` 和 `DELETE` 語句，直接在資料庫層操作。

### 5. Background Context：避免阻塞主執行緒

```swift
// 在背景 context 執行耗時操作
let backgroundContext = persistentContainer.newBackgroundContext()

backgroundContext.perform {
    // 大量資料插入
    for i in 0..<10000 {
        let book = Book(context: backgroundContext)
        book.title = "Book \(i)"
    }
    
    try? backgroundContext.save()
    
    // 通知主執行緒更新 UI
    DispatchQueue.main.async {
        self.tableView.reloadData()
    }
}
```

## 資料遷移（Migration）

資料模型改變時，需要遷移現有資料：

### Lightweight Migration（輕量遷移）

適合簡單的變更：加欄位、刪欄位、改名

```swift
let container = NSPersistentContainer(name: "Model")
let description = container.persistentStoreDescriptions.first
description?.shouldMigrateStoreAutomatically = true
description?.shouldInferMappingModelAutomatically = true  // 自動推斷遷移

container.loadPersistentStores { description, error in
    if let error = error {
        fatalError("Unable to load store: \(error)")
    }
}
```

### Heavyweight Migration（重量遷移）

複雜的變更需要自訂 Mapping Model：

1. 建立新版本的 Data Model（Editor → Add Model Version）
2. 建立 Mapping Model（File → New → Mapping Model）
3. 自訂遷移邏輯

```swift
class BookToBookV2MigrationPolicy: NSEntityMigrationPolicy {
    override func createDestinationInstances(
        forSource sInstance: NSManagedObject,
        in mapping: NSEntityMapping,
        manager: NSMigrationManager
    ) throws {
        let sourceBook = sInstance
        let destinationBook = NSEntityDescription.insertNewObject(
            forEntityName: "Book",
            into: manager.destinationContext
        )
        
        // 自訂資料轉換
        destinationBook.setValue(sourceBook.value(forKey: "title"), forKey: "title")
        let price = sourceBook.value(forKey: "price") as? Double ?? 0.0
        destinationBook.setValue(price * 1.1, forKey: "price")  // 價格加 10%
        
        manager.associate(sourceInstance: sInstance, withDestinationInstance: destinationBook, for: mapping)
    }
}
```

## 與 Java ORM 的對比

**Hibernate/JPA**：
- 物件關係對映（ORM）
- 直接對應資料庫表
- HQL 查詢語言
- 自動 schema 生成

**Core Data**：
- 物件圖管理框架
- 不直接暴露 SQL
- NSPredicate 查詢
- 視覺化的 Data Model Editor

Core Data 更抽象，隱藏底層資料庫細節。Hibernate 更接近 SQL，適合複雜查詢。

## 學習心得

Core Data 的學習曲線陡峭，特別是理解 Context、Coordinator、Store 的關係。但一旦掌握，能很有效率地管理複雜的資料模型。

關鍵重點：
- **理解 Faulting 機制**：避免不必要的資料載入
- **用 NSFetchedResultsController**：TableView 自動更新
- **背景 Context**：耗時操作不阻塞 UI
- **Batch 操作**：大量資料處理
- **資料遷移**：妥善規劃 schema 變更

Core Data 在小型 app 可能過度設計，但在大型 app 中是不可或缺的工具。
