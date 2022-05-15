---
layout: post 
title: "單例模式：在多執行緒環境下的陷阜與實踐" 
date: 2016-08-22 13:45:00 +0800 
categories: [設計模式, Java] 
tags: [單例模式, 多線程, 並發控制]

單例模式的初衷非常單純：確保一個類別只有一個實例，並提供一個全域的存取點。這在處理設定檔（Config）、資料庫連接池（Connection Pool）或執行緒池時非常有用。

然而，在試圖手寫一個高效能的快取管理員時，我才發現「確保唯一」這件事在併發環境下有多麼棘手、多麽困難。

## 錯誤的執行緒

最基礎的「懶漢式 (Lazy Initialization)」在單執行緒下運作良好，但在多執行緒下會產生多個實例：

```java
// 錯誤的示範：非執行緒安全
public class UnsafeSingleton {
    private static UnsafeSingleton instance;
    private UnsafeSingleton() {}

    public static UnsafeSingleton getInstance() {
        if (instance == null) { // 執行緒 A 與 B 可能同時通過這個判斷
            instance = new UnsafeSingleton();
        }
        return instance;
    }
}

```

## 雙重檢查鎖 (Double-Checked Locking)

為了效能，我不想在整個 `getInstance()` 方法上加 `synchronized`（那會變成序列化存取）。於是我嘗試了雙重檢查鎖：

```java
public class DclSingleton {
    // 關鍵字 volatile 是必須的！
    private static volatile DclSingleton instance;

    private DclSingleton() {}

    public static DclSingleton getInstance() {
        if (instance == null) {
            synchronized (DclSingleton.class) {
                if (instance == null) {
                    instance = new DclSingleton();
                }
            }
        }
        return instance;
    }
}

```

### 為什麼要加 `volatile`？

`instance = new DclSingleton()` 在 JVM 中其實分為三步：

1. 分配記憶體空間。
2. 初始化物件。
3. 將 `instance` 指向分配的空間。

由於 JVM 的**指令重排 (Instruction Reordering)**，步驟 2 和 3 可能會顛倒。如果沒有 `volatile`，執行緒 B 可能會拿到一個「已分配空間但尚未初始化完成」的半成品物件，導致系統拋出空指標異常（NPE）。

雖然 DCL 有效，但代碼顯得臃腫。在實踐中，我就比較喜歡另外兩種方法：

### 1. 靜態內部類 (Bill Pugh Singleton)

這種方式利用了類別載入機制來保證執行緒安全，同時兼具延遲加載的特性。

```java
public class SmartSingleton {
    private SmartSingleton() {}

    private static class SingletonHolder {
        private static final SmartSingleton INSTANCE = new SmartSingleton();
    }

    public static SmartSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

```

### 2. 枚舉單例 (Enum Singleton)

這是 《Effective Java》作者 Joshua Bloch 推薦的方式。它不僅能防止反射攻擊，還能自動處理序列化問題。

```java
public enum EnumSingleton {
    INSTANCE;
    
    public void doSomething() {
        // 業務邏輯
    }
}

```

## 單例模式，為什麼要少用？

雖然學會了如何寫正確的單例，但在 OOAD 的原則下，要開始反思：**單例是否被過度使用了？**

1. **隱藏依賴**：單例像是一個全域變數，使得類別之間的依賴關係變得不明確。
2. **難以測試**：單例的狀態會持續存在於整個 JVM 生命周期，這使得單元測試（Unit Test）難以保持獨立性。
3. **違背單一職責**：單例類別通常既負責自己的業務邏輯，又負責管理自己的生命週期。

## 結語

在專案中最後決定將大多數的單例管理交給 **Spring Framework (IoC Container)**。透過依賴注入，物件預設就是單例（Singleton Scope），但開發者不需要手寫繁瑣的 `getInstance` 邏輯，也更容易進行 Mock 測試。
