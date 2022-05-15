---
layout: post 
title: "抽象工廠模式：實現多資料庫切換的架構設計" 
date: 2016-07-01 09:45:00 +0800 
categories: [設計模式, Java] 
tags: [抽象工廠, 系統架構]
---

當系統規模擴大到需要支援多種環境（私有雲、公有雲、不同資料庫廠牌）時，如果程式碼中充斥著 `if (dbType == "MYSQL")` 這樣的邏輯，那將會是維護者的噩夢。

## 核心問題：物件家族的依賴

在我們的系統中，處理資料庫需要一組相關聯的物件：

1. **Connection** (連線管理)
2. **Command** (指令執行)
3. **Dialect** (方言處理，處理分頁等差異)

如果我們直接在業務邏輯中實作這些類別，當要切換資料庫時，我們必須修改成百上千個地方。

## 抽象工廠模式的解決方案

抽象工廠模式提供了一個介面，用於建立「相關或相依物件的家族」，而不需要指定它們具體的類別。

### 1. 定義抽象產品介面

首先，定義資料庫組件的標準。

```java
public interface Connection {
    void connect();
}

public interface Command {
    void execute(String sql);
}

```

### 2. 定義抽象工廠介面

這是工廠模式的核心，它定義了「生產一組產品」的能力。

```java
public interface DatabaseFactory {
    Connection createConnection();
    Command createCommand();
}

```

### 3. 實作具體的產品家族

針對 MySQL 與 Oracle 分別實作。

```java
// MySQL 家族
public class MySqlConnection implements Connection {
    public void connect() { System.out.println("連接到 MySQL..."); }
}
public class MySqlCommand implements Command {
    public void execute(String sql) { System.out.println("MySQL 執行: " + sql); }
}

// Oracle 家族
public class OracleConnection implements Connection {
    public void connect() { System.out.println("連接到 Oracle..."); }
}
public class OracleCommand implements Command {
    public void execute(String sql) { System.out.println("Oracle 執行: " + sql); }
}

```

### 4. 實作具體工廠

```java
public class MySqlFactory implements DatabaseFactory {
    public Connection createConnection() { return new MySqlConnection(); }
    public Command createCommand() { return new MySqlCommand(); }
}

public class OracleFactory implements DatabaseFactory {
    public Connection createConnection() { return new OracleConnection(); }
    public Command createCommand() { return new OracleCommand(); }
}

```

## 業務端的應用 (Client)

現在，我們的業務邏輯完全不需要知道底層是哪種資料庫。

```java
public class DataService {
    private Connection connection;
    private Command command;

    // 透過抽象工廠注入
    public DataService(DatabaseFactory factory) {
        this.connection = factory.createConnection();
        this.command = factory.createCommand();
    }

    public void saveData(String data) {
        connection.connect();
        command.execute("INSERT INTO table VALUES ('" + data + "')");
    }
}

```

### 如何動態切換？

我們可以結合**設定檔**與**反射 (Reflection)** 或簡單的簡單工廠來決定載入哪個 `DatabaseFactory`：

```java
String dbType = Config.get("db.type"); // 從 application.properties 讀取
DatabaseFactory factory;

if ("MYSQL".equals(dbType)) {
    factory = new MySqlFactory();
} else {
    factory = new OracleFactory();
}

DataService service = new DataService(factory);

```

## 實戰深度分析：與簡單工廠的區別

我很常會混淆 **簡單工廠 (Simple Factory)** 與 **抽象工廠 (Abstract Factory)**：

1. **簡單工廠**：生產「一種」抽象產品的不同實作（例如：只要一個 `Connection`）。
2. **抽象工廠**：生產「一組」相關聯的產品家族（例如：`Connection` + `Command` + `Transaction`）。

當系統需要確保 `MySqlConnection` 絕對不會誤用到 `OracleCommand` 時，抽象工廠提供了型別安全的保障。

## 結語

抽象工廠模式雖然增加了類別的數量，但它為系統帶來了極高的**穩定性**與**可互換性**。在 2019 年的這次架構升級中，我透過這種方式，僅花了一週時間就完成了原本預計一個月才能完成的資料庫遷移測試。
