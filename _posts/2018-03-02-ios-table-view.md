---
layout: post
title: "iOS Table View 基礎"
date: 2018-03-02
categories: [Swift, iOS]
tags: [iOS, UITableView, DataSource]
---

這週學習 iOS 最常用的資料展示元件：**UITableView**。

在 Java Swing 中，我們用 JTable 顯示表格資料。iOS 的 Table View 比較接近 Android 的 RecyclerView，是用來顯示「列表」而非「表格」。

初次接觸會覺得概念很多：DataSource、Delegate、Cell Reuse、IndexPath。但其實逻輯很清晰：
- **DataSource**：告訴 Table View 要顯示什麼資料（類似 Swing 的 TableModel）
- **Delegate**：處理使用者互動（類似 Swing 的 Listener）
- **Cell Reuse**：重用儲存格以提升效能（Swing 沒有這個機制）

理解這些概念後，發現 iOS 的設計很合理，特別是 Cell Reuse 機制能大幅提升長列表的效能。

## UITableView 概述

UITableView 是 iOS 最常用的資料展示元件，用於顯示可滾動的列表。類似 Android 的 RecyclerView。

### 基本架構

Table View 需要兩個核心協定：
- **UITableViewDataSource**：提供資料
- **UITableViewDelegate**：處理互動

## 基本實作

### 程式碼建立 Table View

```swift
class ViewController: UIViewController {
    let tableView = UITableView()
    let data = ["Apple", "Banana", "Orange", "Grape", "Mango"]
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        setupTableView()
    }
    
    func setupTableView() {
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        view.addSubview(tableView)
    }
}

extension ViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return data.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        cell.textLabel?.text = data[indexPath.row]
        return cell
    }
}

extension ViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        print("Selected: \(data[indexPath.row])")
        tableView.deselectRow(at: indexPath, animated: true)
    }
}
```

### 必要的 DataSource 方法

```swift
// 回傳行數
func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int

// 回傳 cell
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell

// 選用：回傳 section 數量，預設為 1
func numberOfSections(in tableView: UITableView) -> Int
```

## Cell 重用機制

### Reuse Identifier

```swift
// 註冊 cell
tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")

// 重用 cell
let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
```

重用機制避免重複建立 cell，提升效能。類似 Android RecyclerView 的 ViewHolder 模式。

### Cell 樣式

UITableViewCell 提供四種內建樣式：

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    
    cell.textLabel?.text = "Title"
    cell.detailTextLabel?.text = "Detail"
    cell.imageView?.image = UIImage(named: "icon")
    
    return cell
}
```

- **Default**：左圖 + 標題
- **Subtitle**：左圖 + 標題 + 副標題（下方）
- **Value1**：標題（左）+ 值（右）
- **Value2**：標題（左，藍色）+ 值（右）

## 自訂 Cell

### 建立自訂 Cell 類別

```swift
class CustomTableViewCell: UITableViewCell {
    let titleLabel = UILabel()
    let subtitleLabel = UILabel()
    let customImageView = UIImageView()
    
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupViews()
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    func setupViews() {
        customImageView.frame = CGRect(x: 16, y: 8, width: 60, height: 60)
        contentView.addSubview(customImageView)
        
        titleLabel.frame = CGRect(x: 90, y: 16, width: 200, height: 20)
        titleLabel.font = UIFont.boldSystemFont(ofSize: 16)
        contentView.addSubview(titleLabel)
        
        subtitleLabel.frame = CGRect(x: 90, y: 40, width: 200, height: 16)
        subtitleLabel.font = UIFont.systemFont(ofSize: 14)
        subtitleLabel.textColor = .gray
        contentView.addSubview(subtitleLabel)
    }
    
    func configure(title: String, subtitle: String, image: UIImage?) {
        titleLabel.text = title
        subtitleLabel.text = subtitle
        customImageView.image = image
    }
}
```

### 使用自訂 Cell

```swift
class ViewController: UIViewController {
    let tableView = UITableView()
    let items = [
        ("Apple", "Fresh fruit", UIImage(named: "apple")),
        ("Banana", "Yellow fruit", UIImage(named: "banana")),
        ("Orange", "Citrus fruit", UIImage(named: "orange"))
    ]
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.register(CustomTableViewCell.self, forCellReuseIdentifier: "customCell")
        tableView.rowHeight = 76
        view.addSubview(tableView)
    }
}

extension ViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return items.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "customCell", for: indexPath) as! CustomTableViewCell
        let item = items[indexPath.row]
        cell.configure(title: item.0, subtitle: item.1, image: item.2)
        return cell
    }
}
```

## Section

### 多 Section 實作

```swift
class ContactsViewController: UIViewController {
    let tableView = UITableView()
    let sections = ["A", "B", "C"]
    let contacts = [
        ["Alice", "Anna", "Andrew"],
        ["Bob", "Betty", "Brian"],
        ["Charlie", "Chris", "Catherine"]
    ]
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupTableView()
    }
    
    func setupTableView() {
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        view.addSubview(tableView)
    }
}

extension ContactsViewController: UITableViewDataSource {
    func numberOfSections(in tableView: UITableView) -> Int {
        return sections.count
    }
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return contacts[section].count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        cell.textLabel?.text = contacts[indexPath.section][indexPath.row]
        return cell
    }
    
    func tableView(_ tableView: UITableView, titleForHeaderInSection section: Int) -> String? {
        return sections[section]
    }
}

extension ContactsViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let name = contacts[indexPath.section][indexPath.row]
        print("Selected: \(name)")
        tableView.deselectRow(at: indexPath, animated: true)
    }
}
```

### 自訂 Section Header

```swift
func tableView(_ tableView: UITableView, viewForHeaderInSection section: Int) -> UIView? {
    let headerView = UIView()
    headerView.backgroundColor = .lightGray
    
    let label = UILabel()
    label.text = sections[section]
    label.font = UIFont.boldSystemFont(ofSize: 18)
    label.frame = CGRect(x: 16, y: 0, width: tableView.bounds.width - 32, height: 40)
    headerView.addSubview(label)
    
    return headerView
}

func tableView(_ tableView: UITableView, heightForHeaderInSection section: Int) -> CGFloat {
    return 40
}
```

## Delegate 方法

### 行高

```swift
// 固定行高
tableView.rowHeight = 80

// 動態行高
func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
    return indexPath.row % 2 == 0 ? 60 : 80
}

// 自動行高
tableView.estimatedRowHeight = 80
tableView.rowHeight = UITableView.automaticDimension
```

### 選取事件

```swift
func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    print("Selected row \(indexPath.row) in section \(indexPath.section)")
    tableView.deselectRow(at: indexPath, animated: true)
}

func tableView(_ tableView: UITableView, didDeselectRowAt indexPath: IndexPath) {
    print("Deselected row \(indexPath.row)")
}
```

### 附件按鈕

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    cell.textLabel?.text = data[indexPath.row]
    cell.accessoryType = .detailButton
    return cell
}

func tableView(_ tableView: UITableView, accessoryButtonTappedForRowWith indexPath: IndexPath) {
    print("Accessory button tapped for \(data[indexPath.row])")
}
```

## 編輯模式

### 刪除

```swift
extension ViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle, forRowAt indexPath: IndexPath) {
        if editingStyle == .delete {
            data.remove(at: indexPath.row)
            tableView.deleteRows(at: [indexPath], with: .fade)
        }
    }
}
```

### 插入

```swift
func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle, forRowAt indexPath: IndexPath) {
    if editingStyle == .insert {
        data.insert("New Item", at: indexPath.row)
        tableView.insertRows(at: [indexPath], with: .automatic)
    }
}

func tableView(_ tableView: UITableView, editingStyleForRowAt indexPath: IndexPath) -> UITableViewCell.EditingStyle {
    return .insert
}
```

### 重新排序

```swift
func tableView(_ tableView: UITableView, canMoveRowAt indexPath: IndexPath) -> Bool {
    return true
}

func tableView(_ tableView: UITableView, moveRowAt sourceIndexPath: IndexPath, to destinationIndexPath: IndexPath) {
    let item = data.remove(at: sourceIndexPath.row)
    data.insert(item, at: destinationIndexPath.row)
}

// 啟動編輯模式
tableView.setEditing(true, animated: true)
```

### 自訂編輯動作

```swift
func tableView(_ tableView: UITableView, editActionsForRowAt indexPath: IndexPath) -> [UITableViewRowAction]? {
    let deleteAction = UITableViewRowAction(style: .destructive, title: "Delete") { _, indexPath in
        self.data.remove(at: indexPath.row)
        tableView.deleteRows(at: [indexPath], with: .fade)
    }
    
    let shareAction = UITableViewRowAction(style: .normal, title: "Share") { _, indexPath in
        print("Share \(self.data[indexPath.row])")
    }
    shareAction.backgroundColor = .blue
    
    return [deleteAction, shareAction]
}
```

## 重新整理控制

### Pull to Refresh

```swift
class ViewController: UIViewController {
    let tableView = UITableView()
    var data = Array(1...20)
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupTableView()
        setupRefreshControl()
    }
    
    func setupRefreshControl() {
        let refreshControl = UIRefreshControl()
        refreshControl.addTarget(self, action: #selector(refreshData), for: .valueChanged)
        tableView.refreshControl = refreshControl
    }
    
    @objc func refreshData() {
        DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) {
            self.data = Array(1...Int.random(in: 10...30))
            self.tableView.reloadData()
            self.tableView.refreshControl?.endRefreshing()
        }
    }
}
```

## 效能優化

### 避免在 cellForRowAt 做耗時操作

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    
    // 好的做法
    cell.textLabel?.text = data[indexPath.row].name
    
    // 避免
    // let image = processLargeImage()  // 耗時操作
    // cell.imageView?.image = image
    
    // 改用非同步載入
    loadImageAsync(for: indexPath) { image in
        cell.imageView?.image = image
    }
    
    return cell
}
```

### 快取高度

```swift
class ViewController: UIViewController {
    var heightCache: [IndexPath: CGFloat] = [:]
    
    func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        if let height = heightCache[indexPath] {
            return height
        }
        
        let height = calculateHeight(for: indexPath)
        heightCache[indexPath] = height
        return height
    }
    
    func calculateHeight(for indexPath: IndexPath) -> CGFloat {
        // 計算邏輯
        return 80
    }
}
```

## 實作待辦事項列表

```swift
struct TodoItem {
    var title: String
    var isCompleted: Bool
}

class TodoListViewController: UIViewController {
    let tableView = UITableView()
    var todos: [TodoItem] = [
        TodoItem(title: "Buy groceries", isCompleted: false),
        TodoItem(title: "Call mom", isCompleted: true),
        TodoItem(title: "Finish report", isCompleted: false)
    ]
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Todo List"
        setupTableView()
        setupAddButton()
    }
    
    func setupTableView() {
        tableView.frame = view.bounds
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
        view.addSubview(tableView)
    }
    
    func setupAddButton() {
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .add,
            target: self,
            action: #selector(addTodo)
        )
    }
    
    @objc func addTodo() {
        let alert = UIAlertController(title: "New Todo", message: nil, preferredStyle: .alert)
        alert.addTextField { textField in
            textField.placeholder = "Enter todo"
        }
        alert.addAction(UIAlertAction(title: "Cancel", style: .cancel))
        alert.addAction(UIAlertAction(title: "Add", style: .default) { _ in
            if let text = alert.textFields?.first?.text, !text.isEmpty {
                self.todos.append(TodoItem(title: text, isCompleted: false))
                self.tableView.reloadData()
            }
        })
        present(alert, animated: true)
    }
}

extension TodoListViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return todos.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
        let todo = todos[indexPath.row]
        
        cell.textLabel?.text = todo.title
        cell.accessoryType = todo.isCompleted ? .checkmark : .none
        cell.textLabel?.textColor = todo.isCompleted ? .gray : .black
        
        return cell
    }
    
    func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle, forRowAt indexPath: IndexPath) {
        if editingStyle == .delete {
            todos.remove(at: indexPath.row)
            tableView.deleteRows(at: [indexPath], with: .fade)
        }
    }
}

extension TodoListViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        todos[indexPath.row].isCompleted.toggle()
        tableView.reloadRows(at: [indexPath], with: .automatic)
        tableView.deselectRow(at: indexPath, animated: true)
    }
}
```

## 小結

UITableView 是 iOS 開發的核心元件，掌握 DataSource 和 Delegate 協定是關鍵。Cell 重用機制提升效能，自訂 Cell 滿足各種 UI 需求。編輯模式支援刪除、插入、重新排序等操作。理解這些基礎後，才能建構複雜的列表介面。

下週將學習 Collection View，提供更靈活的網格佈局。
