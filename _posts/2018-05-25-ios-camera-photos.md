---
layout: post
title: "iOS 相機與照片整合"
date: 2018-05-25
categories: [iOS, Swift]
tags: [Swift, iOS, Camera, Photos, UIImagePickerController, AVFoundation]
---

這週深入研究相機和照片功能。在 Web 開發中，圖片上傳通常是透過 `<input type="file">`，功能很陽春。但在 iOS 上，我們可以直接控制相機硬體、即時處理影像、整合照片庫，打造更豐富的使用者體驗。

## 相機與照片的權限管理

和定位、推送一樣，存取相機和照片需要**明確的使用者授權**。iOS 把這兩個權限分開管理：

**相機權限**：拍攝新照片或影片  
**照片庫權限**：存取已有的照片和影片

在 `Info.plist` 中加入說明：

```xml
<key>NSCameraUsageDescription</key>
<string>我們需要使用相機來拍攝您的個人照片</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>我們需要存取照片庫來選擇您要分享的照片</string>

<key>NSPhotoLibraryAddUsageDescription</key>
<string>我們需要儲存照片到您的照片庫</string>
```

注意第三個權限是 iOS 11 新增的，專門用於「儲存照片到照片庫」。這個細緻的權限設計展現了 Apple 對隱私保護的重視程度。

在 Java 桌面應用中，存取檔案系統通常沒這麼多限制。但行動裝置上的照片包含大量個人資訊（位置、時間、人臉），必須更謹慎處理。

## UIImagePickerController：快速整合

最簡單的方式是使用 **UIImagePickerController**，這是系統內建的照片選擇和拍照介面：

```swift
import UIKit

class PhotoViewController: UIViewController, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
    
    @IBOutlet weak var imageView: UIImageView!
    
    @IBAction func takePhoto() {
        // 檢查是否有相機
        guard UIImagePickerController.isSourceTypeAvailable(.camera) else {
            print("此裝置沒有相機")
            return
        }
        
        let picker = UIImagePickerController()
        picker.delegate = self
        picker.sourceType = .camera
        picker.cameraCaptureMode = .photo  // 拍照模式（或 .video）
        picker.allowsEditing = true  // 允許拍照後裁切
        
        present(picker, animated: true)
    }
    
    @IBAction func selectFromLibrary() {
        let picker = UIImagePickerController()
        picker.delegate = self
        picker.sourceType = .photoLibrary  // 或 .savedPhotosAlbum
        picker.allowsEditing = true
        
        present(picker, animated: true)
    }
    
    // 使用者選擇或拍攝照片後
    func imagePickerController(
        _ picker: UIImagePickerController,
        didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]
    ) {
        // 取得照片
        var selectedImage: UIImage?
        
        if let editedImage = info[.editedImage] as? UIImage {
            // 如果有裁切，使用裁切後的圖片
            selectedImage = editedImage
        } else if let originalImage = info[.originalImage] as? UIImage {
            // 否則使用原始圖片
            selectedImage = originalImage
        }
        
        imageView.image = selectedImage
        
        // 儲存到照片庫（需要權限）
        if let image = selectedImage {
            UIImageWriteToSavedPhotosAlbum(image, nil, nil, nil)
        }
        
        picker.dismiss(animated: true)
    }
    
    // 使用者取消
    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        picker.dismiss(animated: true)
    }
}
```

`UIImagePickerController` 的好處是**完全不用自己設計 UI**，系統提供標準介面，使用者很熟悉。但缺點是**客製化能力有限**，如果想要特殊的相機介面，就要用 AVFoundation。

### sourceType 的差異

- **camera**：啟動相機拍攝
- **photoLibrary**：顯示所有相簿，可以選擇任何照片
- **savedPhotosAlbum**：只顯示「相機膠卷」，通常是使用者最近拍的照片

實務上我發現大部分 app 用 `photoLibrary`，因為它給使用者最大的選擇彈性。

## 處理照片方向

iPhone 拍照時會記錄裝置方向，但直接使用 `UIImage` 有時會發現方向不對。這是因為**圖片資料的方向和顯示方向可能不一致**。

```swift
extension UIImage {
    func fixOrientation() -> UIImage {
        // 如果方向已經正確，直接回傳
        if imageOrientation == .up {
            return self
        }
        
        // 建立新的 context 並重新繪製
        UIGraphicsBeginImageContextWithOptions(size, false, scale)
        draw(in: CGRect(origin: .zero, size: size))
        let normalizedImage = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
        
        return normalizedImage ?? self
    }
}
```

這個問題在我上傳照片到 Java 後端時也遇過。如果不處理，使用者上傳的照片在網頁上可能是歪的。iOS 端先修正方向可以避免這個問題。

## 壓縮照片

iPhone 拍的照片動輒 3-5 MB，直接上傳會浪費流量和時間。通常要先壓縮：

```swift
func compressImage(_ image: UIImage, maxSizeKB: Int) -> Data? {
    var compression: CGFloat = 0.9
    var imageData = image.jpegData(compressionQuality: compression)
    
    // 逐步降低品質直到檔案小於限制
    while let data = imageData, data.count / 1024 > maxSizeKB && compression > 0.1 {
        compression -= 0.1
        imageData = image.jpegData(compressionQuality: compression)
    }
    
    return imageData
}

// 使用
if let compressedData = compressImage(originalImage, maxSizeKB: 500) {
    // 上傳或儲存
    uploadToServer(compressedData)
}
```

`jpegData(compressionQuality:)` 的參數範圍是 0.0 到 1.0：
- **1.0**：最高品質，檔案最大
- **0.8-0.9**：視覺上幾乎無損，檔案小很多
- **< 0.5**：明顯失真，不建議

我的經驗是 **0.8 是甜蜜點**，在品質和大小間取得平衡。

## 調整照片尺寸

除了壓縮品質，還可以縮小尺寸。現代 iPhone 拍的照片有 4000x3000 像素，但顯示在手機螢幕上根本不需要這麼大：

```swift
func resizeImage(_ image: UIImage, targetWidth: CGFloat) -> UIImage? {
    let scale = targetWidth / image.size.width
    let newHeight = image.size.height * scale
    let newSize = CGSize(width: targetWidth, height: newHeight)
    
    UIGraphicsBeginImageContextWithOptions(newSize, false, 0.0)
    image.draw(in: CGRect(origin: .zero, size: newSize))
    let resizedImage = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext()
    
    return resizedImage
}

// 將照片縮小到寬度 1024
if let resized = resizeImage(originalImage, targetWidth: 1024),
   let data = resized.jpegData(compressionQuality: 0.8) {
    print("原始大小：\(originalImage.pngData()?.count ?? 0) bytes")
    print("壓縮後：\(data.count) bytes")
}
```

先縮小尺寸再壓縮品質，通常能把 5 MB 的照片降到 200-300 KB，而且視覺上差異不大。

這個技巧在 Java 後端也常用，用 ImageIO 或 Thumbnailator 處理。但在 iOS 客戶端先處理，能節省網路頻寬。

## Photos Framework：深度整合照片庫

`UIImagePickerController` 功能有限，如果要更精細的控制，可以用 **Photos Framework**：

```swift
import Photos

class PhotoLibraryManager {
    
    // 檢查授權狀態
    func checkAuthorization(completion: @escaping (Bool) -> Void) {
        let status = PHPhotoLibrary.authorizationStatus()
        
        switch status {
        case .authorized:
            completion(true)
            
        case .notDetermined:
            // 第一次請求權限
            PHPhotoLibrary.requestAuthorization { newStatus in
                completion(newStatus == .authorized)
            }
            
        case .denied, .restricted:
            completion(false)
            
        @unknown default:
            completion(false)
        }
    }
    
    // 取得所有相簿
    func fetchAlbums() -> [PHAssetCollection] {
        var albums: [PHAssetCollection] = []
        
        // 取得使用者建立的相簿
        let userAlbums = PHAssetCollection.fetchAssetCollections(
            with: .album,
            subtype: .albumRegular,
            options: nil
        )
        
        userAlbums.enumerateObjects { collection, _, _ in
            albums.append(collection)
        }
        
        // 取得系統相簿（如「最近項目」）
        let smartAlbums = PHAssetCollection.fetchAssetCollections(
            with: .smartAlbum,
            subtype: .any,
            options: nil
        )
        
        smartAlbums.enumerateObjects { collection, _, _ in
            // 只取常用的
            if collection.assetCollectionSubtype == .smartAlbumUserLibrary ||
               collection.assetCollectionSubtype == .smartAlbumFavorites {
                albums.append(collection)
            }
        }
        
        return albums
    }
    
    // 從相簿取得照片
    func fetchPhotos(from album: PHAssetCollection, limit: Int = 20) -> [PHAsset] {
        let options = PHFetchOptions()
        options.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]
        options.fetchLimit = limit
        
        let assets = PHAsset.fetchAssets(in: album, options: options)
        
        var photos: [PHAsset] = []
        assets.enumerateObjects { asset, _, _ in
            if asset.mediaType == .image {
                photos.append(asset)
            }
        }
        
        return photos
    }
    
    // 載入照片
    func loadImage(
        from asset: PHAsset,
        targetSize: CGSize,
        completion: @escaping (UIImage?) -> Void
    ) {
        let options = PHImageRequestOptions()
        options.deliveryMode = .highQualityFormat
        options.isNetworkAccessAllowed = true  // 允許從 iCloud 下載
        
        PHImageManager.default().requestImage(
            for: asset,
            targetSize: targetSize,
            contentMode: .aspectFill,
            options: options
        ) { image, _ in
            completion(image)
        }
    }
}
```

Photos Framework 的威力在於：
- **不用把所有照片載入記憶體**，只取需要的
- **支援 iCloud 照片庫**，自動下載雲端照片
- **效能優化**，可以指定縮圖大小避免浪費資源
- **即時監控變化**，照片庫更新時自動通知

這個設計類似 Java 的分頁查詢（Pagination），不會一次載入全部資料。

## 儲存照片到照片庫

用 Photos Framework 儲存照片更可靠：

```swift
func savePhoto(_ image: UIImage, completion: @escaping (Bool, Error?) -> Void) {
    PHPhotoLibrary.shared().performChanges({
        PHAssetChangeRequest.creationRequestForAsset(from: image)
    }) { success, error in
        DispatchQueue.main.async {
            completion(success, error)
        }
    }
}

// 儲存到特定相簿
func savePhotoToAlbum(_ image: UIImage, albumName: String) {
    // 查找或建立相簿
    var album: PHAssetCollection?
    
    let collections = PHAssetCollection.fetchAssetCollections(
        with: .album,
        subtype: .albumRegular,
        options: nil
    )
    
    collections.enumerateObjects { collection, _, stop in
        if collection.localizedTitle == albumName {
            album = collection
            stop.pointee = true
        }
    }
    
    PHPhotoLibrary.shared().performChanges({
        // 建立照片 asset
        let creationRequest = PHAssetChangeRequest.creationRequestForAsset(from: image)
        
        if let album = album,
           let placeholder = creationRequest.placeholderForCreatedAsset {
            // 加入現有相簿
            let albumChangeRequest = PHAssetCollectionChangeRequest(for: album)
            albumChangeRequest?.addAssets([placeholder] as NSArray)
        } else {
            // 建立新相簿
            let createAlbumRequest = PHAssetCollectionChangeRequest.creationRequestForAssetCollection(
                withTitle: albumName
            )
            if let placeholder = creationRequest.placeholderForCreatedAsset,
               let albumPlaceholder = createAlbumRequest.placeholderForCreatedAssetCollection {
                let albumChangeRequest = PHAssetCollectionChangeRequest(
                    for: albumPlaceholder
                )
                albumChangeRequest?.addAssets([placeholder] as NSArray)
            }
        }
    })
}
```

這個 API 設計得很安全，所有修改都包在 `performChanges` 區塊中，如果出錯會自動回滾。這類似資料庫的 Transaction。

## AVFoundation：自訂相機介面

如果系統的相機介面不符合需求，可以用 **AVFoundation** 完全客製化：

```swift
import AVFoundation

class CustomCameraViewController: UIViewController {
    var captureSession: AVCaptureSession?
    var photoOutput: AVCapturePhotoOutput?
    var previewLayer: AVCaptureVideoPreviewLayer?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupCamera()
    }
    
    func setupCamera() {
        // 建立 capture session
        captureSession = AVCaptureSession()
        captureSession?.sessionPreset = .photo  // 照片品質
        
        // 取得相機裝置（後鏡頭）
        guard let camera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back) else {
            print("找不到相機")
            return
        }
        
        do {
            // 建立輸入
            let input = try AVCaptureDeviceInput(device: camera)
            if captureSession?.canAddInput(input) == true {
                captureSession?.addInput(input)
            }
            
            // 建立輸出
            photoOutput = AVCapturePhotoOutput()
            if captureSession?.canAddOutput(photoOutput!) == true {
                captureSession?.addOutput(photoOutput!)
            }
            
            // 建立預覽層
            previewLayer = AVCaptureVideoPreviewLayer(session: captureSession!)
            previewLayer?.videoGravity = .resizeAspectFill
            previewLayer?.frame = view.bounds
            view.layer.insertSublayer(previewLayer!, at: 0)
            
            // 開始執行
            captureSession?.startRunning()
            
        } catch {
            print("設定相機失敗：\(error)")
        }
    }
    
    @IBAction func capturePhoto() {
        let settings = AVCapturePhotoSettings()
        settings.flashMode = .auto
        
        photoOutput?.capturePhoto(with: settings, delegate: self)
    }
}

extension CustomCameraViewController: AVCapturePhotoCaptureDelegate {
    func photoOutput(
        _ output: AVCapturePhotoOutput,
        didFinishProcessingPhoto photo: AVCapturePhoto,
        error: Error?
    ) {
        guard let imageData = photo.fileDataRepresentation(),
              let image = UIImage(data: imageData) else {
            print("無法產生照片")
            return
        }
        
        print("拍攝成功：\(image.size)")
        // 處理照片
    }
}
```

用 AVFoundation 的好處：
- **完全控制 UI**：按鈕、濾鏡、網格線都能自訂
- **即時處理**：可以在拍照前套用濾鏡或特效
- **存取更多參數**：ISO、快門速度、白平衡等
- **支援影片錄製**：切換 output 類型即可

缺點是**程式碼複雜很多**。如果只是基本的拍照功能，用 `UIImagePickerController` 就夠了。

這讓我想到 Spring Boot 和原生 Spring 的關係。前者簡單但彈性有限，後者複雜但無所不能。

## 圖片濾鏡：Core Image

iOS 提供 **Core Image** 框架來處理圖片效果：

```swift
import CoreImage

func applyFilter(to image: UIImage, filterName: String) -> UIImage? {
    guard let ciImage = CIImage(image: image) else { return nil }
    
    let filter = CIFilter(name: filterName)
    filter?.setValue(ciImage, forKey: kCIInputImageKey)
    
    // 某些濾鏡有額外參數
    if filterName == "CIGaussianBlur" {
        filter?.setValue(10.0, forKey: kCIInputRadiusKey)
    }
    
    guard let outputImage = filter?.outputImage else { return nil }
    
    let context = CIContext()
    guard let cgImage = context.createCGImage(outputImage, from: outputImage.extent) else {
        return nil
    }
    
    return UIImage(cgImage: cgImage)
}

// 使用範例
if let filtered = applyFilter(to: originalImage, filterName: "CIPhotoEffectNoir") {
    imageView.image = filtered  // 黑白效果
}
```

常用濾鏡：
- **CIPhotoEffectNoir**：黑白
- **CIPhotoEffectChrome**：復古
- **CIGaussianBlur**：模糊
- **CISharpenLuminance**：銳利化
- **CIVibrance**：鮮豔度

Core Image 的效能很好，因為它利用 **GPU 加速**。處理大圖片也很快。

## 小結

這週研究相機和照片功能，體會到 iOS 在多媒體處理上的**分層設計哲學**：

- **簡單需求**：用 UIImagePickerController，幾行程式碼搞定
- **進階需求**：用 Photos Framework，精細控制照片庫
- **完全客製化**：用 AVFoundation，從頭打造相機

每一層都有明確的適用場景，不會為了彈性而犧牲易用性。

關鍵要點：
1. **尊重權限**：清楚說明為何需要存取相機或照片
2. **壓縮照片**：上傳前先調整大小和品質
3. **處理方向**：iPhone 照片可能有旋轉標記
4. **非同步載入**：Photos Framework 的操作都是非同步的
5. **記憶體管理**：不要同時載入大量高解析度照片

下週計劃研究音訊和影片播放，包括 AVAudioPlayer、AVPlayer、背景播放、控制中心整合等。這能完善多媒體功能的知識體系。
