---
layout: post
title: "iOS Collection View 與自訂佈局"
date: 2018-03-09
categories: [Swift, iOS]
tags: [iOS, UICollectionView, Layout]
---

學完 Table View 後，這週進階到更靈活的 **UICollectionView**。

Table View 只能顯示單列的列表，Collection View 則能實現網格、瀑布流、圓形佈局等各種複雜佈局。從 Java 的角度來看，這就像 Swing 的 GridLayout 和 FlowLayout，但功能強大很多。

初次接觸會覺得 Collection View 比 Table View 複雜，但其實核心概念是一樣的：DataSource 提供資料、Delegate 處理互動、Cell Reuse 提升效能。最大的差異是多了 **Layout** 這個概念，用來控制 Cell 的排列方式。

## UICollectionView 概述

UICollectionView 是比 Table View 更靈活的資料展示元件，支援網格、瀑布流等各種佈局。

### 基本架構

Collection View 需要三個核心元件：
- **UICollectionViewDataSource**：提供資料
- **UICollectionViewDelegate**：處理互動
- **UICollectionViewLayout**：控制佈局

## 基本實作

### 程式碼建立 Collection View

```swift
class PhotoGridViewController: UIViewController {
    var collectionView: UICollectionView!
    let photos = Array(1...30)
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupCollectionView()
    }
    
    func setupCollectionView() {
        let layout = UICollectionViewFlowLayout()
        layout.itemSize = CGSize(width: 100, height: 100)
        layout.minimumInteritemSpacing = 10
        layout.minimumLineSpacing = 10
        layout.sectionInset = UIEdgeInsets(top: 10, left: 10, bottom: 10, right: 10)
        
        collectionView = UICollectionView(frame: view.bounds, collectionViewLayout: layout)
        collectionView.backgroundColor = .white
        collectionView.dataSource = self
        collectionView.delegate = self
        collectionView.register(PhotoCell.self, forCellWithReuseIdentifier: "photoCell")
        
        view.addSubview(collectionView)
    }
}

extension PhotoGridViewController: UICollectionViewDataSource {
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return photos.count
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "photoCell", for: indexPath) as! PhotoCell
        cell.configure(number: photos[indexPath.item])
        return cell
    }
}

extension PhotoGridViewController: UICollectionViewDelegate {
    func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        print("Selected photo \(photos[indexPath.item])")
    }
}
```

### 自訂 Cell

```swift
class PhotoCell: UICollectionViewCell {
    let imageView = UIImageView()
    let numberLabel = UILabel()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupViews()
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    func setupViews() {
        imageView.frame = contentView.bounds
        imageView.contentMode = .scaleAspectFill
        imageView.clipsToBounds = true
        imageView.backgroundColor = .lightGray
        contentView.addSubview(imageView)
        
        numberLabel.frame = CGRect(x: 0, y: 0, width: contentView.bounds.width, height: contentView.bounds.height)
        numberLabel.textAlignment = .center
        numberLabel.font = UIFont.boldSystemFont(ofSize: 24)
        numberLabel.textColor = .white
        contentView.addSubview(numberLabel)
    }
    
    func configure(number: Int) {
        numberLabel.text = "\(number)"
        imageView.backgroundColor = UIColor(hue: CGFloat(number) / 30.0, saturation: 0.6, brightness: 0.8, alpha: 1.0)
    }
}
```

## UICollectionViewFlowLayout

Flow Layout 是最常用的佈局方式。

### 設定 Item 大小

```swift
let layout = UICollectionViewFlowLayout()

// 固定大小
layout.itemSize = CGSize(width: 100, height: 120)

// 動態大小
extension PhotoGridViewController: UICollectionViewDelegateFlowLayout {
    func collectionView(_ collectionView: UICollectionView,
                       layout collectionViewLayout: UICollectionViewLayout,
                       sizeForItemAt indexPath: IndexPath) -> CGSize {
        let width = (collectionView.bounds.width - 30) / 3
        return CGSize(width: width, height: width)
    }
}
```

### 間距設定

```swift
let layout = UICollectionViewFlowLayout()

// Item 之間的水平間距
layout.minimumInteritemSpacing = 10

// 行之間的垂直間距
layout.minimumLineSpacing = 10

// Section 邊距
layout.sectionInset = UIEdgeInsets(top: 10, left: 10, bottom: 10, right: 10)

// 動態設定
func collectionView(_ collectionView: UICollectionView,
                   layout collectionViewLayout: UICollectionViewLayout,
                   minimumInteritemSpacingForSectionAt section: Int) -> CGFloat {
    return 5
}

func collectionView(_ collectionView: UICollectionView,
                   layout collectionViewLayout: UICollectionViewLayout,
                   minimumLineSpacingForSectionAt section: Int) -> CGFloat {
    return 10
}

func collectionView(_ collectionView: UICollectionView,
                   layout collectionViewLayout: UICollectionViewLayout,
                   insetForSectionAt section: Int) -> UIEdgeInsets {
    return UIEdgeInsets(top: 20, left: 10, bottom: 20, right: 10)
}
```

### 滾動方向

```swift
let layout = UICollectionViewFlowLayout()
layout.scrollDirection = .horizontal  // 或 .vertical
```

## Section Header 和 Footer

### 註冊 Supplementary View

```swift
collectionView.register(HeaderView.self,
                       forSupplementaryViewOfKind: UICollectionView.elementKindSectionHeader,
                       withReuseIdentifier: "header")

collectionView.register(FooterView.self,
                       forSupplementaryViewOfKind: UICollectionView.elementKindSectionFooter,
                       withReuseIdentifier: "footer")
```

### 提供 Supplementary View

```swift
func collectionView(_ collectionView: UICollectionView,
                   viewForSupplementaryElementOfKind kind: String,
                   at indexPath: IndexPath) -> UICollectionReusableView {
    if kind == UICollectionView.elementKindSectionHeader {
        let header = collectionView.dequeueReusableSupplementaryView(
            ofKind: kind,
            withReuseIdentifier: "header",
            for: indexPath
        ) as! HeaderView
        header.titleLabel.text = "Section \(indexPath.section)"
        return header
    } else {
        let footer = collectionView.dequeueReusableSupplementaryView(
            ofKind: kind,
            withReuseIdentifier: "footer",
            for: indexPath
        ) as! FooterView
        return footer
    }
}

func collectionView(_ collectionView: UICollectionView,
                   layout collectionViewLayout: UICollectionViewLayout,
                   referenceSizeForHeaderInSection section: Int) -> CGSize {
    return CGSize(width: collectionView.bounds.width, height: 50)
}
```

### 自訂 Header View

```swift
class HeaderView: UICollectionReusableView {
    let titleLabel = UILabel()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        backgroundColor = .lightGray
        
        titleLabel.frame = CGRect(x: 16, y: 0, width: frame.width - 32, height: frame.height)
        titleLabel.font = UIFont.boldSystemFont(ofSize: 18)
        addSubview(titleLabel)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
```

## 多 Section

```swift
class CategoryViewController: UIViewController {
    var collectionView: UICollectionView!
    let categories = ["Fruits", "Vegetables", "Drinks"]
    let items = [
        ["Apple", "Banana", "Orange", "Grape"],
        ["Carrot", "Broccoli", "Spinach"],
        ["Water", "Juice", "Coffee", "Tea", "Soda"]
    ]
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupCollectionView()
    }
    
    func setupCollectionView() {
        let layout = UICollectionViewFlowLayout()
        layout.itemSize = CGSize(width: 80, height: 80)
        layout.minimumInteritemSpacing = 10
        layout.minimumLineSpacing = 10
        layout.sectionInset = UIEdgeInsets(top: 10, left: 10, bottom: 10, right: 10)
        
        collectionView = UICollectionView(frame: view.bounds, collectionViewLayout: layout)
        collectionView.backgroundColor = .white
        collectionView.dataSource = self
        collectionView.register(ItemCell.self, forCellWithReuseIdentifier: "cell")
        collectionView.register(HeaderView.self,
                               forSupplementaryViewOfKind: UICollectionView.elementKindSectionHeader,
                               withReuseIdentifier: "header")
        view.addSubview(collectionView)
    }
}

extension CategoryViewController: UICollectionViewDataSource {
    func numberOfSections(in collectionView: UICollectionView) -> Int {
        return categories.count
    }
    
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return items[section].count
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "cell", for: indexPath) as! ItemCell
        cell.configure(text: items[indexPath.section][indexPath.item])
        return cell
    }
    
    func collectionView(_ collectionView: UICollectionView,
                       viewForSupplementaryElementOfKind kind: String,
                       at indexPath: IndexPath) -> UICollectionReusableView {
        let header = collectionView.dequeueReusableSupplementaryView(
            ofKind: kind,
            withReuseIdentifier: "header",
            for: indexPath
        ) as! HeaderView
        header.titleLabel.text = categories[indexPath.section]
        return header
    }
}
```

## 自訂 Layout

### 建立自訂 Layout

```swift
class WaterfallLayout: UICollectionViewLayout {
    weak var delegate: WaterfallLayoutDelegate?
    
    private var numberOfColumns = 2
    private var cellPadding: CGFloat = 6
    private var cache: [UICollectionViewLayoutAttributes] = []
    private var contentHeight: CGFloat = 0
    private var contentWidth: CGFloat {
        guard let collectionView = collectionView else { return 0 }
        let insets = collectionView.contentInset
        return collectionView.bounds.width - (insets.left + insets.right)
    }
    
    override var collectionViewContentSize: CGSize {
        return CGSize(width: contentWidth, height: contentHeight)
    }
    
    override func prepare() {
        guard let collectionView = collectionView,
              cache.isEmpty else { return }
        
        let columnWidth = contentWidth / CGFloat(numberOfColumns)
        var xOffset: [CGFloat] = []
        for column in 0..<numberOfColumns {
            xOffset.append(CGFloat(column) * columnWidth)
        }
        var yOffset: [CGFloat] = Array(repeating: 0, count: numberOfColumns)
        
        var column = 0
        for item in 0..<collectionView.numberOfItems(inSection: 0) {
            let indexPath = IndexPath(item: item, section: 0)
            
            let photoHeight = delegate?.collectionView(collectionView, heightForPhotoAtIndexPath: indexPath) ?? 180
            let height = cellPadding * 2 + photoHeight
            
            let frame = CGRect(x: xOffset[column], y: yOffset[column], width: columnWidth, height: height)
            let insetFrame = frame.insetBy(dx: cellPadding, dy: cellPadding)
            
            let attributes = UICollectionViewLayoutAttributes(forCellWith: indexPath)
            attributes.frame = insetFrame
            cache.append(attributes)
            
            contentHeight = max(contentHeight, frame.maxY)
            yOffset[column] = yOffset[column] + height
            
            column = column < (numberOfColumns - 1) ? (column + 1) : 0
        }
    }
    
    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        return cache.filter { $0.frame.intersects(rect) }
    }
    
    override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        return cache[indexPath.item]
    }
}

protocol WaterfallLayoutDelegate: AnyObject {
    func collectionView(_ collectionView: UICollectionView, heightForPhotoAtIndexPath indexPath: IndexPath) -> CGFloat
}
```

### 使用自訂 Layout

```swift
class WaterfallViewController: UIViewController {
    var collectionView: UICollectionView!
    let photos = Array(1...50)
    let photoHeights = (1...50).map { _ in CGFloat.random(in: 100...300) }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupCollectionView()
    }
    
    func setupCollectionView() {
        let layout = WaterfallLayout()
        layout.delegate = self
        
        collectionView = UICollectionView(frame: view.bounds, collectionViewLayout: layout)
        collectionView.backgroundColor = .white
        collectionView.dataSource = self
        collectionView.register(PhotoCell.self, forCellWithReuseIdentifier: "cell")
        view.addSubview(collectionView)
    }
}

extension WaterfallViewController: WaterfallLayoutDelegate {
    func collectionView(_ collectionView: UICollectionView, heightForPhotoAtIndexPath indexPath: IndexPath) -> CGFloat {
        return photoHeights[indexPath.item]
    }
}

extension WaterfallViewController: UICollectionViewDataSource {
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return photos.count
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "cell", for: indexPath) as! PhotoCell
        cell.configure(number: photos[indexPath.item])
        return cell
    }
}
```

## 實作照片牆應用

```swift
struct Photo {
    let id: Int
    let color: UIColor
}

class PhotoWallViewController: UIViewController {
    var collectionView: UICollectionView!
    var photos: [Photo] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "Photo Wall"
        generatePhotos()
        setupCollectionView()
        setupNavigationBar()
    }
    
    func generatePhotos() {
        for i in 1...100 {
            let color = UIColor(
                red: CGFloat.random(in: 0...1),
                green: CGFloat.random(in: 0...1),
                blue: CGFloat.random(in: 0...1),
                alpha: 1.0
            )
            photos.append(Photo(id: i, color: color))
        }
    }
    
    func setupCollectionView() {
        let layout = UICollectionViewFlowLayout()
        let spacing: CGFloat = 2
        let itemsPerRow: CGFloat = 3
        let totalSpacing = spacing * (itemsPerRow + 1)
        let itemWidth = (view.bounds.width - totalSpacing) / itemsPerRow
        
        layout.itemSize = CGSize(width: itemWidth, height: itemWidth)
        layout.minimumInteritemSpacing = spacing
        layout.minimumLineSpacing = spacing
        layout.sectionInset = UIEdgeInsets(top: spacing, left: spacing, bottom: spacing, right: spacing)
        
        collectionView = UICollectionView(frame: view.bounds, collectionViewLayout: layout)
        collectionView.backgroundColor = .black
        collectionView.dataSource = self
        collectionView.delegate = self
        collectionView.register(PhotoWallCell.self, forCellWithReuseIdentifier: "cell")
        view.addSubview(collectionView)
    }
    
    func setupNavigationBar() {
        navigationItem.rightBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .add,
            target: self,
            action: #selector(addPhoto)
        )
    }
    
    @objc func addPhoto() {
        let color = UIColor(
            red: CGFloat.random(in: 0...1),
            green: CGFloat.random(in: 0...1),
            blue: CGFloat.random(in: 0...1),
            alpha: 1.0
        )
        let photo = Photo(id: photos.count + 1, color: color)
        photos.insert(photo, at: 0)
        
        collectionView.performBatchUpdates({
            collectionView.insertItems(at: [IndexPath(item: 0, section: 0)])
        })
    }
}

extension PhotoWallViewController: UICollectionViewDataSource {
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return photos.count
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "cell", for: indexPath) as! PhotoWallCell
        cell.configure(photo: photos[indexPath.item])
        return cell
    }
}

extension PhotoWallViewController: UICollectionViewDelegate {
    func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        let photo = photos[indexPath.item]
        print("Selected photo \(photo.id)")
        
        // 顯示全螢幕照片
        let detailVC = PhotoDetailViewController()
        detailVC.photo = photo
        navigationController?.pushViewController(detailVC, animated: true)
    }
}

class PhotoWallCell: UICollectionViewCell {
    let imageView = UIImageView()
    let idLabel = UILabel()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        imageView.frame = contentView.bounds
        imageView.contentMode = .scaleAspectFill
        imageView.clipsToBounds = true
        contentView.addSubview(imageView)
        
        idLabel.frame = CGRect(x: 4, y: 4, width: 30, height: 20)
        idLabel.font = UIFont.boldSystemFont(ofSize: 12)
        idLabel.textColor = .white
        idLabel.backgroundColor = UIColor.black.withAlphaComponent(0.5)
        idLabel.textAlignment = .center
        idLabel.layer.cornerRadius = 4
        idLabel.clipsToBounds = true
        contentView.addSubview(idLabel)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    func configure(photo: Photo) {
        imageView.backgroundColor = photo.color
        idLabel.text = "\(photo.id)"
    }
}
```

## 效能優化

### 預載

```swift
func collectionView(_ collectionView: UICollectionView,
                   willDisplay cell: UICollectionViewCell,
                   forItemAt indexPath: IndexPath) {
    // 預載即將顯示的資料
    if indexPath.item == photos.count - 5 {
        loadMorePhotos()
    }
}
```

### 快取圖片

```swift
class ImageCache {
    static let shared = ImageCache()
    private var cache = NSCache<NSString, UIImage>()
    
    func image(for key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
    
    func setImage(_ image: UIImage, for key: String) {
        cache.setObject(image, forKey: key as NSString)
    }
}
```

## 小結

UICollectionView 提供靈活的佈局能力，適合網格、瀑布流等各種排列方式。Flow Layout 滿足大部分需求，自訂 Layout 提供完全的控制。理解佈局機制和效能優化是建構流暢 UI 的關鍵。

下週將學習導航和畫面轉場，包含 Navigation Controller 和 Tab Bar Controller。
