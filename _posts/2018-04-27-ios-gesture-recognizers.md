---
layout: post
title: "iOS 手勢辨識"
date: 2018-04-27
categories: [iOS, Swift]
tags: [Swift, iOS, Gesture, UIGestureRecognizer, Touch]
---

這週深入研究 iOS 的手勢辨識系統。在 Java Swing 中處理滑鼠事件相對簡單，因為桌面應用主要是點擊和拖曳。但觸控螢幕的互動方式豐富太多：點擊、長按、滑動、捏合、旋轉，每種都有不同的意義。

## 觸控與手勢的差異

一開始我以為手勢辨識就是「處理觸控事件」，但實際上這是兩個層次的概念。

**觸控事件（Touch Events）** 是最底層的：`touchesBegan`、`touchesMoved`、`touchesEnded`。這些方法告訴你「手指在螢幕上什麼位置」，但不告訴你「使用者想做什麼」。

**手勢辨識器（Gesture Recognizer）** 是高階抽象：它監聽觸控事件，分析手指的移動模式，然後告訴你「這是一個滑動手勢」或「這是一個捏合手勢」。

類比到 Java，這就像 KeyListener 和 Action 的關係。前者告訴你「按下了 Ctrl+C」，後者告訴你「這是複製動作」。

iOS 鼓勵我們用手勢辨識器，因為：
1. **Apple 已經幫你實作複雜的手勢邏輯**，像是區分點擊和長按
2. **程式碼更簡潔**，不需要自己追蹤觸控狀態
3. **使用者體驗一致**，所有 app 的手勢行為都相同

## 基本手勢類型

UIKit 提供了七種內建的手勢辨識器：

1. **UITapGestureRecognizer**：點擊（單擊或多次點擊）
2. **UILongPressGestureRecognizer**：長按
3. **UIPanGestureRecognizer**：拖移（手指不離開螢幕移動）
4. **UISwipeGestureRecognizer**：快速滑動
5. **UIPinchGestureRecognizer**：捏合（兩指縮放）
6. **UIRotationGestureRecognizer**：旋轉（兩指旋轉）
7. **UIScreenEdgePanGestureRecognizer**：從螢幕邊緣滑入

每種手勢適用的場景不同，選對手勢能大幅提升使用者體驗。

## 實作點擊手勢

最簡單的例子是雙擊放大圖片：

```swift
class ImageViewController: UIViewController {
    @IBOutlet weak var imageView: UIImageView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 建立雙擊手勢
        let doubleTap = UITapGestureRecognizer(
            target: self,
            action: #selector(handleDoubleTap(_:))
        )
        doubleTap.numberOfTapsRequired = 2  // 要求雙擊
        
        // 啟用 imageView 的使用者互動
        imageView.isUserInteractionEnabled = true
        
        // 加入手勢
        imageView.addGestureRecognizer(doubleTap)
    }
    
    @objc func handleDoubleTap(_ gesture: UITapGestureRecognizer) {
        let currentScale = imageView.transform.scale
        let newScale: CGFloat = currentScale == 1.0 ? 2.0 : 1.0
        
        UIView.animate(withDuration: 0.3) {
            self.imageView.transform = CGAffineTransform(scaleX: newScale, y: newScale)
        }
    }
}
```

這個設計模式和 UIControl 的 target-action 一樣，非常一致。注意 **UIImageView 預設不接受使用者互動**，要手動設定 `isUserInteractionEnabled = true`。

我一開始花了半小時 debug 為什麼手勢沒反應，後來才發現是這個設定。這和 Java Swing 中所有元件預設都能接受事件不同。

## 長按手勢：上下文選單

長按常用來顯示更多選項，類似桌面應用的右鍵選單：

```swift
let longPress = UILongPressGestureRecognizer(
    target: self,
    action: #selector(handleLongPress(_:))
)
longPress.minimumPressDuration = 0.5  // 長按至少 0.5 秒
view.addGestureRecognizer(longPress)
```

```swift
@objc func handleLongPress(_ gesture: UILongPressGestureRecognizer) {
    // 長按手勢有多個階段，只在開始時觸發
    guard gesture.state == .began else { return }
    
    let point = gesture.location(in: tableView)
    
    if let indexPath = tableView.indexPathForRow(at: point) {
        // 顯示操作選單
        let alert = UIAlertController(
            title: "選項",
            message: "要對這個項目做什麼？",
            preferredStyle: .actionSheet
        )
        
        alert.addAction(UIAlertAction(title: "刪除", style: .destructive) { _ in
            self.deleteItem(at: indexPath)
        })
        
        alert.addAction(UIAlertAction(title: "分享", style: .default) { _ in
            self.shareItem(at: indexPath)
        })
        
        alert.addAction(UIAlertAction(title: "取消", style: .cancel))
        
        present(alert, animated: true)
    }
}
```

長按手勢的 `state` 屬性很重要。它會經過多個階段：`.began` → `.changed` → `.ended`。如果不檢查狀態，同一個長按會觸發多次回呼，造成重複處理。

這和 Java 的滑鼠監聽器類似，要區分 mousePressed、mouseDragged、mouseReleased。但 iOS 的狀態機設計更明確。

## 拖移手勢：移動物件

Pan（拖移）是最實用的手勢之一。實作一個可拖動的便利貼效果：

```swift
@objc func handlePan(_ gesture: UIPanGestureRecognizer) {
    let view = gesture.view!
    let translation = gesture.translation(in: view.superview)
    
    switch gesture.state {
    case .began, .changed:
        // 移動 view
        view.center = CGPoint(
            x: view.center.x + translation.x,
            y: view.center.y + translation.y
        )
        // 重設 translation，否則會累積
        gesture.setTranslation(.zero, in: view.superview)
        
    case .ended:
        // 手指離開時加入彈性效果
        let velocity = gesture.velocity(in: view.superview)
        
        UIView.animate(
            withDuration: 0.3,
            delay: 0,
            usingSpringWithDamping: 0.7,
            initialSpringVelocity: 0.5,
            options: [],
            animations: {
                // 可以在這裡加入對齊到網格的邏輯
            }
        )
        
    default:
        break
    }
}
```

`translation` 是**相對於上次回呼的位移量**，不是絕對位置。所以每次更新位置後要呼叫 `setTranslation(.zero, ...)`，重設累積值。

這個設計一開始讓我困惑，因為 Java AWT 的 MouseMotionListener 提供的是絕對座標。但實際使用後發現相對位移更方便，不需要自己記錄起始點。

`velocity` 屬性提供拖移速度，可以用來實現「甩飛」效果。使用者快速滑動後，物件會依照慣性繼續移動，這是很自然的物理效果。

## 捏合手勢：縮放

雙指捏合用來縮放內容，這是 iOS 的標誌性互動：

```swift
let pinch = UIPinchGestureRecognizer(
    target: self,
    action: #selector(handlePinch(_:))
)
imageView.addGestureRecognizer(pinch)
```

```swift
@objc func handlePinch(_ gesture: UIPinchGestureRecognizer) {
    guard let view = gesture.view else { return }
    
    switch gesture.state {
    case .began, .changed:
        // scale 是相對於手勢開始時的比例
        let scale = gesture.scale
        view.transform = view.transform.scaledBy(x: scale, y: scale)
        // 重設 scale，避免倍數累積
        gesture.scale = 1.0
        
    case .ended:
        // 限制最小和最大縮放
        let currentScale = view.frame.width / view.bounds.width
        if currentScale < 0.5 {
            UIView.animate(withDuration: 0.2) {
                view.transform = CGAffineTransform(scaleX: 0.5, y: 0.5)
            }
        } else if currentScale > 3.0 {
            UIView.animate(withDuration: 0.2) {
                view.transform = CGAffineTransform(scaleX: 3.0, y: 3.0)
            }
        }
        
    default:
        break
    }
}
```

和 pan 手勢一樣，`scale` 也是相對值。每次套用變換後要重設為 1.0。

這裡展示了一個重要概念：**在手勢結束時檢查並修正不合理的狀態**。如果使用者縮放太小或太大，平滑地動畫回合理範圍，比直接限制更友善。

## 旋轉手勢

雙指旋轉適合用在圖片編輯或地圖旋轉：

```swift
@objc func handleRotation(_ gesture: UIRotationGestureRecognizer) {
    guard let view = gesture.view else { return }
    
    view.transform = view.transform.rotated(by: gesture.rotation)
    gesture.rotation = 0  // 重設累積的角度
}
```

`rotation` 的單位是弧度，正值代表順時針。這和 Java 2D 的 AffineTransform 類似，但 iOS 的 API 更簡潔。

## 同時使用多個手勢

一個 view 可以同時有多個手勢辨識器。比如同時支援縮放和旋轉：

```swift
let pinch = UIPinchGestureRecognizer(target: self, action: #selector(handlePinch(_:)))
let rotation = UIRotationGestureRecognizer(target: self, action: #selector(handleRotation(_:)))

imageView.addGestureRecognizer(pinch)
imageView.addGestureRecognizer(rotation)

// 讓兩個手勢同時辨識
pinch.delegate = self
rotation.delegate = self
```

實作 delegate 方法：

```swift
extension ViewController: UIGestureRecognizerDelegate {
    func gestureRecognizer(
        _ gestureRecognizer: UIGestureRecognizer,
        shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer
    ) -> Bool {
        return true  // 允許同時辨識
    }
}
```

預設情況下，一個手勢被辨識後會阻止其他手勢。設定 delegate 可以改變這個行為。

這在實作複雜互動時很有用，比如照片編輯器可以同時縮放、旋轉、拖移。

## 手勢優先順序

有時希望某個手勢優先，只有它失敗時才嘗試其他手勢。比如**雙擊優先於單擊**：

```swift
let singleTap = UITapGestureRecognizer(target: self, action: #selector(handleSingleTap))
let doubleTap = UITapGestureRecognizer(target: self, action: #selector(handleDoubleTap))
doubleTap.numberOfTapsRequired = 2

// 單擊要等雙擊失敗後才觸發
singleTap.require(toFail: doubleTap)

view.addGestureRecognizer(singleTap)
view.addGestureRecognizer(doubleTap)
```

`require(toFail:)` 建立依賴關係。這樣使用者雙擊時不會同時觸發單擊，提升體驗。

這個機制比自己用計時器判斷優雅太多。我在 Java Swing 專案中實作過類似功能，要自己管理計時器和狀態，相當麻煩。

## 自訂手勢辨識器

內建手勢無法滿足需求時，可以繼承 `UIGestureRecognizer` 自訂。比如實作一個「畫勾」手勢：

```swift
class CheckmarkGestureRecognizer: UIGestureRecognizer {
    private var strokePoints: [CGPoint] = []
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent) {
        super.touchesBegan(touches, with: event)
        
        if touches.count != 1 {
            state = .failed
            return
        }
        
        strokePoints.removeAll()
        if let point = touches.first?.location(in: view) {
            strokePoints.append(point)
        }
    }
    
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent) {
        super.touchesMoved(touches, with: event)
        
        guard state != .failed else { return }
        
        if let point = touches.first?.location(in: view) {
            strokePoints.append(point)
        }
    }
    
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent) {
        super.touchesEnded(touches, with: event)
        
        // 分析軌跡是否像勾勾
        if isCheckmarkStroke(strokePoints) {
            state = .recognized  // 辨識成功
        } else {
            state = .failed
        }
    }
    
    private func isCheckmarkStroke(_ points: [CGPoint]) -> Bool {
        // 簡化版：檢查是否先向下再向上
        guard points.count >= 3 else { return false }
        
        let midIndex = points.count / 2
        let firstHalfDown = points[midIndex].y > points[0].y
        let secondHalfUp = points[points.count - 1].y < points[midIndex].y
        
        return firstHalfDown && secondHalfUp
    }
}
```

自訂手勢辨識器的核心是**覆寫觸控方法並設定 state 屬性**。當 state 變成 `.recognized` 時，target-action 就會被觸發。

這個架構很清楚地分離了「手勢辨識邏輯」和「手勢回應處理」，符合單一職責原則。

## 實際案例：可拖動排序的清單

結合多種手勢實作一個進階功能：長按後可以拖動項目重新排序。

```swift
class DraggableTableViewController: UITableViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let longPress = UILongPressGestureRecognizer(
            target: self,
            action: #selector(handleLongPress(_:))
        )
        tableView.addGestureRecognizer(longPress)
    }
    
    @objc func handleLongPress(_ gesture: UILongPressGestureRecognizer) {
        let location = gesture.location(in: tableView)
        
        switch gesture.state {
        case .began:
            // 找到被長按的 cell
            if let indexPath = tableView.indexPathForRow(at: location) {
                // 開始拖動模式
                tableView.beginInteractiveMovement(forRowAt: indexPath)
            }
            
        case .changed:
            // 更新拖動位置
            tableView.updateInteractiveMovement(targetPosition: location)
            
        case .ended:
            // 完成拖動
            tableView.endInteractiveMovement()
            
        default:
            // 取消拖動
            tableView.cancelInteractiveMovement()
        }
    }
    
    override func tableView(
        _ tableView: UITableView,
        moveRowAt sourceIndexPath: IndexPath,
        to destinationIndexPath: IndexPath
    ) {
        // 更新資料模型
        let item = items.remove(at: sourceIndexPath.row)
        items.insert(item, at: destinationIndexPath.row)
    }
}
```

UITableView 已經內建了互動式移動的 API，我們只需要用手勢辨識器觸發它。這個設計展示了 iOS 框架的**可組合性**：不同功能可以靈活組合。

## 小結

這週學習手勢辨識最大的收穫是理解了 **iOS 的分層設計哲學**：

- **底層**：觸控事件（touchesBegan/Moved/Ended）
- **中層**：手勢辨識器（UIGestureRecognizer）
- **高層**：業務邏輯（回應手勢的實際功能）

每一層都有明確的職責，不會互相干擾。這和 Spring 的分層架構異曲同工：Controller、Service、Repository 各司其職。

手勢辨識器讓 iOS app 的互動更豐富、更直觀。下週打算研究推送通知，這是 app 與使用者持續互動的重要管道。
