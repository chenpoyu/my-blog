---
layout: post
title: "iOS 安全性：Keychain、證書釘扎與資料保護"
date: 2018-08-17
categories: [iOS, Security]
tags: [Security, Keychain, HTTPS, Certificate Pinning]
---

這週研究 iOS app 的安全性議題，從 Keychain 儲存敏感資料，到 HTTPS 證書釘扎，再到資料保護等級。行動 app 的安全性比後端服務更重要，因為 app 跑在使用者裝置上，更容易被逆向工程。

## Keychain：安全儲存敏感資料

**為什麼不能用 UserDefaults？**

UserDefaults 的資料以明文儲存在 plist 檔案中，任何人都能讀取：

```swift
// 不安全！
UserDefaults.standard.set("myPassword123", forKey: "password")
// 儲存在 Library/Preferences/com.example.app.plist
```

**Keychain 的特性**：
- 資料加密儲存
- 受系統保護，其他 app 無法存取
- 可以設定存取控制和生物辨識
- app 刪除後還能保留（可選）

**基本使用**：

```swift
import Security

func savePassword(_ password: String, for account: String) -> Bool {
    let data = password.data(using: .utf8)!
    
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecValueData as String: data
    ]
    
    // 先刪除舊的
    SecItemDelete(query as CFDictionary)
    
    // 新增
    let status = SecItemAdd(query as CFDictionary, nil)
    return status == errSecSuccess
}

func loadPassword(for account: String) -> String? {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecReturnData as String: true
    ]
    
    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    
    guard status == errSecSuccess,
          let data = result as? Data,
          let password = String(data: data, encoding: .utf8) else {
        return nil
    }
    
    return password
}

// 使用
savePassword("mySecretPassword", for: "user@example.com")
let password = loadPassword(for: "user@example.com")
```

**Keychain Wrapper**：

Security API 很低階，通常會封裝成更易用的介面：

```swift
class KeychainHelper {
    enum KeychainError: Error {
        case duplicateItem
        case unknown(OSStatus)
    }
    
    static func save(key: String, data: Data) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]
        
        let status = SecItemAdd(query as CFDictionary, nil)
        
        guard status != errSecDuplicateItem else {
            throw KeychainError.duplicateItem
        }
        
        guard status == errSecSuccess else {
            throw KeychainError.unknown(status)
        }
    }
    
    static func load(key: String) throws -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true
        ]
        
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        guard status != errSecItemNotFound else {
            return nil
        }
        
        guard status == errSecSuccess else {
            throw KeychainError.unknown(status)
        }
        
        return result as? Data
    }
    
    static func delete(key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        
        let status = SecItemDelete(query as CFDictionary)
        
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unknown(status)
        }
    }
}

// 更簡潔的使用
let token = "abc123xyz"
try? KeychainHelper.save(key: "authToken", data: token.data(using: .utf8)!)
if let data = try? KeychainHelper.load(key: "authToken"),
   let token = String(data: data, encoding: .utf8) {
    print("Token: \(token)")
}
```

## 生物辨識保護

可以要求 Touch ID / Face ID 才能存取 Keychain 項目：

```swift
func savePasswordWithBiometric(_ password: String, for account: String) -> Bool {
    let data = password.data(using: .utf8)!
    
    // 建立存取控制
    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
        .userPresence,  // 需要生物辨識或密碼
        nil
    )!
    
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecValueData as String: data,
        kSecAttrAccessControl as String: access
    ]
    
    SecItemDelete(query as CFDictionary)
    let status = SecItemAdd(query as CFDictionary, nil)
    return status == errSecSuccess
}

// 讀取時會要求生物辨識
func loadPasswordWithBiometric(for account: String, completion: @escaping (String?) -> Void) {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecReturnData as String: true,
        kSecUseOperationPrompt as String: "驗證以存取密碼"
    ]
    
    DispatchQueue.global().async {
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        
        DispatchQueue.main.async {
            guard status == errSecSuccess,
                  let data = result as? Data,
                  let password = String(data: data, encoding: .utf8) else {
                completion(nil)
                return
            }
            completion(password)
        }
    }
}
```

## HTTPS 與 App Transport Security (ATS)

iOS 9 後，預設強制使用 HTTPS（ATS）：

```xml
<!-- Info.plist -->
<key>NSAppTransportSecurity</key>
<dict>
    <!-- 允許所有 HTTP（不建議！只用於開發） -->
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

**生產環境應該針對特定網域設定**：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy-api.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>
    </dict>
</dict>
```

## Certificate Pinning：防止中間人攻擊

即使使用 HTTPS，攻擊者還是可能用假證書進行中間人攻擊（MITM）。Certificate Pinning 確保只信任特定的證書。

**原理**：把伺服器的證書或公鑰嵌入 app，連線時驗證伺服器證書是否匹配。

**實作方式 1：用 URLSession Delegate**

```swift
class CertificatePinningDelegate: NSObject, URLSessionDelegate {
    let pinnedCertificates: [Data]
    
    init(pinnedCertificates: [Data]) {
        self.pinnedCertificates = pinnedCertificates
    }
    
    func urlSession(_ session: URLSession,
                    didReceive challenge: URLAuthenticationChallenge,
                    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // 取得伺服器證書
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        let serverCertificateData = SecCertificateCopyData(serverCertificate) as Data
        
        // 驗證是否在 pinned 證書中
        if pinnedCertificates.contains(serverCertificateData) {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}

// 使用
// 1. 把伺服器證書檔案（.cer）加入專案
let certificateURL = Bundle.main.url(forResource: "api_example_com", withExtension: "cer")!
let certificateData = try! Data(contentsOf: certificateURL)

// 2. 建立 URLSession
let delegate = CertificatePinningDelegate(pinnedCertificates: [certificateData])
let session = URLSession(configuration: .default, delegate: delegate, delegateQueue: nil)

// 3. 發送請求
let url = URL(string: "https://api.example.com/data")!
session.dataTask(with: url) { data, response, error in
    // 處理回應
}.resume()
```

**實作方式 2：Public Key Pinning**

```swift
func pinPublicKey(challenge: URLAuthenticationChallenge, pinnedPublicKeyHash: String) -> Bool {
    guard let serverTrust = challenge.protectionSpace.serverTrust,
          let certificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
        return false
    }
    
    // 取得公鑰
    let policy = SecPolicyCreateBasicX509()
    let serverTrustWithPolicy = SecTrustCreateWithCertificates(certificate, policy, nil)
    
    guard let publicKey = SecTrustCopyPublicKey(serverTrust) else {
        return false
    }
    
    // 計算公鑰的 hash
    let publicKeyData = SecKeyCopyExternalRepresentation(publicKey, nil) as Data?
    guard let keyData = publicKeyData else {
        return false
    }
    
    let hash = keyData.sha256()  // 需要自己實作 SHA-256
    let hashString = hash.base64EncodedString()
    
    return hashString == pinnedPublicKeyHash
}
```

**Public Key Pinning 的優點**：證書更新時不用重新發布 app，只要公鑰不變。

## 資料保護等級（Data Protection）

iOS 提供檔案層級的加密保護：

```swift
// 建立檔案時設定保護等級
let fileURL = documentsDirectory.appendingPathComponent("sensitive.txt")
let data = "Sensitive data".data(using: .utf8)!

try data.write(to: fileURL, options: .completeFileProtection)
// .completeFileProtection：裝置鎖定時無法存取
// .completeFileProtectionUnlessOpen：開啟後可在背景存取
// .completeFileProtectionUntilFirstUserAuthentication：重開機前需解鎖一次
// .noFileProtection：不保護（預設）
```

**Keychain 的資料保護**：

```swift
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: account,
    kSecValueData as String: data,
    kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
]

// kSecAttrAccessibleWhenUnlockedThisDeviceOnly：鎖定時無法存取，不同步到其他裝置
// kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly：重開機後解鎖一次即可存取
// kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly：需要設定密碼才能使用
```

## Code Obfuscation：程式碼混淆

雖然 Swift 編譯成機器碼，但還是可以被逆向工程。敏感邏輯應該混淆或放在後端。

**簡單的字串混淆**：

```swift
// 不要這樣寫
let apiKey = "abc123xyz456"

// 至少做點混淆
let obfuscated: [UInt8] = [97, 98, 99, 49, 50, 51, 120, 121, 122, 52, 53, 54]
let apiKey = String(bytes: obfuscated, encoding: .utf8)!

// 更好的做法：放在後端，由伺服器驗證
```

**Jailbreak 檢測**：

```swift
func isJailbroken() -> Bool {
    // 檢查常見的 jailbreak 檔案
    let paths = [
        "/Applications/Cydia.app",
        "/Library/MobileSubstrate/MobileSubstrate.dylib",
        "/bin/bash",
        "/usr/sbin/sshd",
        "/etc/apt"
    ]
    
    for path in paths {
        if FileManager.default.fileExists(atPath: path) {
            return true
        }
    }
    
    // 檢查是否能寫入系統目錄
    let testPath = "/private/jailbreak_test.txt"
    do {
        try "test".write(toFile: testPath, atomically: true, encoding: .utf8)
        try FileManager.default.removeItem(atPath: testPath)
        return true
    } catch {
        return false
    }
}

// 在 app 啟動時檢查
if isJailbroken() {
    // 顯示警告或拒絕執行敏感功能
}
```

## 最佳實踐

1. **永遠不要在程式碼中寫死密碼或 API 金鑰**
2. **敏感資料用 Keychain，不是 UserDefaults**
3. **使用 HTTPS 和 Certificate Pinning**
4. **重要檔案設定資料保護等級**
5. **敏感操作加上生物辨識驗證**
6. **定期更新證書和檢查安全漏洞**
7. **假設 app 會被逆向工程，敏感邏輯放後端**

## 與 Java 後端的對比

**Java 的安全性**：
- 伺服器受控，相對安全
- 可以用 Spring Security、JWT 等
- 資料加密用 AES、RSA

**iOS 的安全性**：
- App 跑在使用者裝置，容易被攻擊
- Keychain 提供系統層級保護
- Certificate Pinning 防止 MITM
- 需考慮 jailbreak 和逆向工程

## 學習心得

iOS 的安全性是我之前在後端開發較少考慮的。後端的 API 金鑰、資料庫密碼都在伺服器上，相對安全。但 app 的所有程式碼和資源都在使用者手上，必須假設會被攻擊。

Keychain 是 iOS 安全性的基石，一定要用對。Certificate Pinning 雖然麻煩，但對高安全性需求的 app（金融、醫療等）是必須的。

下週繼續研究 App Extension 和 Widget，看看如何擴展 app 的功能。
