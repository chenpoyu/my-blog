---
layout: post
title: "iOS 音訊與影片播放"
date: 2018-06-01
categories: [iOS, Swift]
tags: [Swift, iOS, AVFoundation, Audio, Video, Media Player]
---

這週研究多媒體播放功能。在 Web 開發中，音訊和影片播放主要靠 HTML5 的 `<audio>` 和 `<video>` 標籤，瀏覽器處理大部分細節。但在 iOS 原生開發中，我們能更深入控制播放行為、整合系統介面、甚至在背景持續播放。

## 音訊播放的三種層次

iOS 提供三種音訊播放 API，複雜度和功能各不相同：

**System Sound Services**
- 最簡單，只能播放短音效（< 30 秒）
- 不能控制音量，不支援循環
- 適合按鈕音效、提示音

**AVAudioPlayer**
- 中等複雜，適合播放本地音訊檔案
- 支援暫停、循環、音量控制
- 適合音樂播放器、遊戲背景音樂

**AVPlayer**
- 最複雜，支援串流、影片、進階控制
- 可以播放網路資源
- 適合影音 app、串流平台

這個分層設計很合理，類似 Java 的 File API：`Files.readString()` 很簡單，`BufferedReader` 中等，`FileChannel` 最複雜但功能最強。

## System Sound：播放簡短音效

最簡單的方式是用系統音效服務：

```swift
import AudioToolbox

class SoundEffectPlayer {
    var soundID: SystemSoundID = 0
    
    func loadSound(filename: String, ext: String) {
        if let soundURL = Bundle.main.url(forResource: filename, withExtension: ext) {
            AudioServicesCreateSystemSoundID(soundURL as CFURL, &soundID)
        }
    }
    
    func playSound() {
        AudioServicesPlaySystemSound(soundID)
    }
    
    func playWithVibrate() {
        // 播放音效同時震動
        AudioServicesPlayAlertSound(soundID)
    }
    
    deinit {
        // 釋放資源
        AudioServicesDisposeOfSystemSoundID(soundID)
    }
}

// 使用範例
let player = SoundEffectPlayer()
player.loadSound(filename: "button_click", ext: "wav")
player.playSound()
```

這個 API 的限制很多：
- **音訊不能超過 30 秒**
- **不能控制音量**，使用系統音量
- **不支援循環播放**
- **只支援特定格式**：CAF, WAV, AIF

但優點是**極度省電**，適合 UI 互動音效。很多遊戲的按鈕音、通知提示音都用這個。

## AVAudioPlayer：播放本地音訊檔案

大部分音樂播放需求用 `AVAudioPlayer` 就夠了：

```swift
import AVFoundation

class MusicPlayer: NSObject, AVAudioPlayerDelegate {
    var audioPlayer: AVAudioPlayer?
    
    func loadMusic(filename: String) {
        guard let url = Bundle.main.url(forResource: filename, withExtension: "mp3") else {
            print("找不到檔案")
            return
        }
        
        do {
            audioPlayer = try AVAudioPlayer(contentsOf: url)
            audioPlayer?.delegate = self
            audioPlayer?.prepareToPlay()  // 預載，減少播放延遲
            
            print("音訊長度：\(audioPlayer?.duration ?? 0) 秒")
            
        } catch {
            print("載入失敗：\(error)")
        }
    }
    
    func play() {
        audioPlayer?.play()
    }
    
    func pause() {
        audioPlayer?.pause()
    }
    
    func stop() {
        audioPlayer?.stop()
        audioPlayer?.currentTime = 0  // 重設到開頭
    }
    
    func setVolume(_ volume: Float) {
        // 0.0 到 1.0
        audioPlayer?.volume = volume
    }
    
    func seek(to time: TimeInterval) {
        audioPlayer?.currentTime = time
    }
    
    func enableLoop(_ enable: Bool) {
        audioPlayer?.numberOfLoops = enable ? -1 : 0  // -1 表示無限循環
    }
    
    // 播放完成時呼叫
    func audioPlayerDidFinishPlaying(_ player: AVAudioPlayer, successfully flag: Bool) {
        print("播放完成")
        // 可以在這裡自動播放下一首
    }
    
    // 播放中斷時呼叫（如來電）
    func audioPlayerBeginInterruption(_ player: AVAudioPlayer) {
        print("播放被中斷")
    }
    
    func audioPlayerEndInterruption(_ player: AVAudioPlayer, withOptions flags: Int) {
        print("中斷結束，可以恢復播放")
        audioPlayer?.play()
    }
}
```

`AVAudioPlayer` 的 API 設計很直觀，和 Java 的 `javax.sound.sampled.Clip` 類似，但更簡潔。

### 音訊 Session 配置

在播放音訊前，需要設定 **Audio Session**，告訴系統你的 app 如何使用音訊：

```swift
func configureAudioSession() {
    do {
        let audioSession = AVAudioSession.sharedInstance()
        
        // 設定類別
        try audioSession.setCategory(.playback, mode: .default)
        
        // 啟用 session
        try audioSession.setActive(true)
        
    } catch {
        print("設定 Audio Session 失敗：\(error)")
    }
}
```

常用的 Audio Session 類別：

- **ambient**：與其他 app 的音訊混音，會被靜音開關關閉（遊戲背景音樂）
- **soloAmbient**：獨佔音訊，其他 app 停止（遊戲主音效）
- **playback**：背景播放，不受靜音開關影響（音樂 app）
- **record**：錄音
- **playAndRecord**：同時播放和錄音（視訊通話）

這個概念在 Java 開發中比較少見。但在行動裝置上很重要，因為**多個 app 可能同時使用音訊硬體**，系統要協調優先權。

我設定錯類別時，發現音樂會被靜音開關關閉，或者來電時不會自動暫停。正確設定很關鍵。

## 背景播放音訊

音樂 app 的核心功能是**在背景持續播放**。這需要額外配置：

### 1. 啟用背景模式

在 Xcode 專案設定中：
- **Target → Signing & Capabilities**
- 加入 **Background Modes** capability
- 勾選 **Audio, AirPlay, and Picture in Picture**

### 2. 正確設定 Audio Session

```swift
func configureBackgroundPlayback() {
    do {
        let audioSession = AVAudioSession.sharedInstance()
        try audioSession.setCategory(.playback, mode: .default, options: [])
        try audioSession.setActive(true)
        
    } catch {
        print("設定背景播放失敗：\(error)")
    }
}
```

### 3. 整合控制中心和鎖定畫面

使用者在鎖定畫面或控制中心應該能看到正在播放的音樂資訊，並且控制播放：

```swift
import MediaPlayer

func setupNowPlaying(title: String, artist: String, artwork: UIImage?, duration: TimeInterval) {
    var nowPlayingInfo: [String: Any] = [
        MPMediaItemPropertyTitle: title,
        MPMediaItemPropertyArtist: artist,
        MPMediaItemPropertyPlaybackDuration: duration,
        MPNowPlayingInfoPropertyElapsedPlaybackTime: audioPlayer?.currentTime ?? 0,
        MPNowPlayingInfoPropertyPlaybackRate: audioPlayer?.isPlaying == true ? 1.0 : 0.0
    ]
    
    // 設定專輯封面
    if let artwork = artwork {
        let artworkImage = MPMediaItemArtwork(boundsSize: artwork.size) { _ in artwork }
        nowPlayingInfo[MPMediaItemPropertyArtwork] = artworkImage
    }
    
    MPNowPlayingInfoCenter.default().nowPlayingInfo = nowPlayingInfo
}

func setupRemoteControls() {
    let commandCenter = MPRemoteCommandCenter.shared()
    
    // 播放按鈕
    commandCenter.playCommand.isEnabled = true
    commandCenter.playCommand.addTarget { [weak self] _ in
        self?.audioPlayer?.play()
        self?.updateNowPlayingPlaybackRate(1.0)
        return .success
    }
    
    // 暫停按鈕
    commandCenter.pauseCommand.isEnabled = true
    commandCenter.pauseCommand.addTarget { [weak self] _ in
        self?.audioPlayer?.pause()
        self?.updateNowPlayingPlaybackRate(0.0)
        return .success
    }
    
    // 下一首
    commandCenter.nextTrackCommand.isEnabled = true
    commandCenter.nextTrackCommand.addTarget { [weak self] _ in
        self?.playNextTrack()
        return .success
    }
    
    // 上一首
    commandCenter.previousTrackCommand.isEnabled = true
    commandCenter.previousTrackCommand.addTarget { [weak self] _ in
        self?.playPreviousTrack()
        return .success
    }
    
    // 調整播放位置
    commandCenter.changePlaybackPositionCommand.isEnabled = true
    commandCenter.changePlaybackPositionCommand.addTarget { [weak self] event in
        if let event = event as? MPChangePlaybackPositionCommandEvent {
            self?.audioPlayer?.currentTime = event.positionTime
            return .success
        }
        return .commandFailed
    }
}

func updateNowPlayingPlaybackRate(_ rate: Double) {
    var nowPlayingInfo = MPNowPlayingInfoCenter.default().nowPlayingInfo ?? [:]
    nowPlayingInfo[MPNowPlayingInfoPropertyPlaybackRate] = rate
    nowPlayingInfo[MPNowPlayingInfoPropertyElapsedPlaybackTime] = audioPlayer?.currentTime
    MPNowPlayingInfoCenter.default().nowPlayingInfo = nowPlayingInfo
}
```

這個整合讓使用者體驗大幅提升。在 iOS 上，**不整合控制中心的音樂 app 會被認為是半成品**。

這讓我想到 Java 桌面應用的系統托盤整合，都是為了讓 app 和作業系統更緊密結合。

## AVPlayer：播放網路音訊和影片

`AVPlayer` 是最強大的播放器，支援**串流媒體**和**影片**：

```swift
import AVFoundation
import AVKit

class StreamPlayer {
    var player: AVPlayer?
    var playerItem: AVPlayerItem?
    var timeObserver: Any?
    
    func playStream(url: String) {
        guard let streamURL = URL(string: url) else { return }
        
        playerItem = AVPlayerItem(url: streamURL)
        player = AVPlayer(playerItem: playerItem)
        
        // 監控播放狀態
        playerItem?.addObserver(
            self,
            forKeyPath: "status",
            options: [.new],
            context: nil
        )
        
        // 監控緩衝進度
        playerItem?.addObserver(
            self,
            forKeyPath: "loadedTimeRanges",
            options: [.new],
            context: nil
        )
        
        // 監控播放進度（每 0.1 秒更新）
        let interval = CMTime(seconds: 0.1, preferredTimescale: CMTimeScale(NSEC_PER_SEC))
        timeObserver = player?.addPeriodicTimeObserver(
            forInterval: interval,
            queue: .main
        ) { [weak self] time in
            let currentTime = CMTimeGetSeconds(time)
            print("播放進度：\(currentTime) 秒")
            // 更新 UI
        }
        
        player?.play()
    }
    
    override func observeValue(
        forKeyPath keyPath: String?,
        of object: Any?,
        change: [NSKeyValueChangeKey : Any]?,
        context: UnsafeMutableRawPointer?
    ) {
        if keyPath == "status" {
            if playerItem?.status == .readyToPlay {
                print("準備就緒，可以播放")
            } else if playerItem?.status == .failed {
                print("載入失敗：\(playerItem?.error?.localizedDescription ?? "")")
            }
        } else if keyPath == "loadedTimeRanges" {
            // 計算緩衝百分比
            if let timeRanges = playerItem?.loadedTimeRanges,
               let duration = playerItem?.duration,
               !timeRanges.isEmpty {
                let timeRange = timeRanges[0].timeRangeValue
                let bufferedTime = CMTimeGetSeconds(timeRange.start) + CMTimeGetSeconds(timeRange.duration)
                let totalTime = CMTimeGetSeconds(duration)
                let bufferedPercent = bufferedTime / totalTime * 100
                print("已緩衝：\(bufferedPercent)%")
            }
        }
    }
    
    deinit {
        // 移除觀察者
        if let timeObserver = timeObserver {
            player?.removeTimeObserver(timeObserver)
        }
        playerItem?.removeObserver(self, forKeyPath: "status")
        playerItem?.removeObserver(self, forKeyPath: "loadedTimeRanges")
    }
}
```

AVPlayer 的設計用了 **KVO（Key-Value Observing）** 模式，這在 Swift 中不太常見，但在 AVFoundation 中大量使用。

這讓我想起 Java 的 PropertyChangeListener，都是觀察者模式的應用。

## 播放影片

播放影片最簡單的方式是用 **AVPlayerViewController**：

```swift
import AVKit

func playVideo(url: String) {
    guard let videoURL = URL(string: url) else { return }
    
    let player = AVPlayer(url: videoURL)
    let playerViewController = AVPlayerViewController()
    playerViewController.player = player
    
    present(playerViewController, animated: true) {
        player.play()
    }
}
```

系統會提供全螢幕播放介面，包含播放/暫停、進度條、音量、AirPlay 等控制。

如果需要自訂 UI，可以用 `AVPlayerLayer`：

```swift
class CustomVideoPlayer: UIViewController {
    var player: AVPlayer?
    var playerLayer: AVPlayerLayer?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let videoURL = URL(string: "https://example.com/video.mp4")!
        player = AVPlayer(url: videoURL)
        
        playerLayer = AVPlayerLayer(player: player)
        playerLayer?.frame = view.bounds
        playerLayer?.videoGravity = .resizeAspect
        view.layer.addSublayer(playerLayer!)
    }
    
    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        playerLayer?.frame = view.bounds
    }
    
    @IBAction func playTapped() {
        player?.play()
    }
    
    @IBAction func pauseTapped() {
        player?.pause()
    }
}
```

這給了完全的控制權，但也要自己實作所有 UI 和邏輯。

## 處理中斷和音訊路由變更

音訊播放常被**外部事件中斷**，像是來電、鬧鐘、其他 app 的音訊。正確處理這些情況很重要：

```swift
func observeAudioSessionNotifications() {
    NotificationCenter.default.addObserver(
        self,
        selector: #selector(handleInterruption),
        name: AVAudioSession.interruptionNotification,
        object: AVAudioSession.sharedInstance()
    )
    
    NotificationCenter.default.addObserver(
        self,
        selector: #selector(handleRouteChange),
        name: AVAudioSession.routeChangeNotification,
        object: AVAudioSession.sharedInstance()
    )
}

@objc func handleInterruption(notification: Notification) {
    guard let userInfo = notification.userInfo,
          let typeValue = userInfo[AVAudioSessionInterruptionTypeKey] as? UInt,
          let type = AVAudioSession.InterruptionType(rawValue: typeValue) else {
        return
    }
    
    switch type {
    case .began:
        // 中斷開始（如來電）
        print("音訊被中斷，暫停播放")
        audioPlayer?.pause()
        
    case .ended:
        // 中斷結束
        if let optionsValue = userInfo[AVAudioSessionInterruptionOptionKey] as? UInt {
            let options = AVAudioSession.InterruptionOptions(rawValue: optionsValue)
            if options.contains(.shouldResume) {
                // 系統建議恢復播放
                print("恢復播放")
                audioPlayer?.play()
            }
        }
        
    @unknown default:
        break
    }
}

@objc func handleRouteChange(notification: Notification) {
    guard let userInfo = notification.userInfo,
          let reasonValue = userInfo[AVAudioSessionRouteChangeReasonKey] as? UInt,
          let reason = AVAudioSession.RouteChangeReason(rawValue: reasonValue) else {
        return
    }
    
    switch reason {
    case .oldDeviceUnavailable:
        // 音訊裝置中斷（如拔掉耳機）
        print("耳機被拔掉，暫停播放")
        audioPlayer?.pause()
        
    case .newDeviceAvailable:
        // 新裝置可用（如插入耳機）
        print("耳機插入")
        
    default:
        break
    }
}
```

這些通知處理確保了**良好的使用者體驗**。想像你在聽音樂時拔掉耳機，音樂如果繼續從喇叭播放會很尷尬。iOS app 應該自動暫停。

## 音訊格式選擇

iOS 支援多種音訊格式，選擇合適的格式能平衡品質和檔案大小：

- **AAC (m4a)**：iOS 原生支援，品質好檔案小，最推薦
- **MP3**：相容性最好，但授權有疑慮
- **WAV / AIFF**：無損，但檔案巨大
- **ALAC (Apple Lossless)**：無損但檔案較小，適合高品質音樂

對於 app 內建的音效和音樂，我建議：
- **音效**：CAF 或 WAV（未壓縮，低延遲）
- **背景音樂**：AAC（壓縮，節省空間）
- **串流**：AAC 或 HLS（HTTP Live Streaming）

這和 Web 開發選擇圖片格式類似：PNG 無損但大，JPEG 壓縮但小，WebP 更新但相容性待提升。

## 小結

這週研究音訊和影片播放，最大的感受是 **iOS 對多媒體的深度整合**。從簡單的系統音效到完整的串流播放器，框架提供了完整的工具鏈。

關鍵要點：
1. **選對 API**：簡單需求用簡單 API，不要過度設計
2. **配置 Audio Session**：告訴系統你的音訊用途
3. **背景播放三要素**：Background Mode + Audio Session + Now Playing
4. **處理中斷**：來電、耳機拔除等事件要正確回應
5. **優化效能**：預載音訊、選擇合適格式、管理記憶體

下週計劃研究架構模式，特別是 MVC、MVVM、以及如何組織大型 iOS 專案的程式碼結構。這對維護性很關鍵。
