---
layout: post
title: "Tomcat 架構筆記"
date: 2017-09-08 14:55:00 +0800
categories: [設計模式, Java]
tags: [Tomcat, Servlet, Web 容器, 伺服器]
---

前面我們學習了 Spring JDBC（請參考 [Spring 資料存取](/posts/2017/09/01/spring-jdbc-template/)），現在讓我們深入了解 Web 應用的執行環境：Tomcat。作為 Java Web 開發的基石,理解 Tomcat 的運作原理對優化應用效能至關重要。

## 什麼是 Tomcat？

Tomcat 是一個 **Servlet 容器**,也是 **Web 伺服器**:
- 實作了 Servlet 和 JSP 規範
- 處理 HTTP 請求和回應
- 管理 Web 應用的生命週期

## Tomcat 架構層次

```
Tomcat Server
├── Service
    ├── Connector (處理網路連線)
    │   ├── HTTP Connector (port 8080)
    │   └── AJP Connector (port 8009)
    └── Engine (處理請求)
        └── Host (虛擬主機)
            └── Context (Web 應用)
                └── Wrapper (Servlet)
```

## 核心元件說明

### 1. Server
Tomcat 的最外層容器,代表整個伺服器實例。

### 2. Service  
連接 Connector 和 Engine,一個 Server 可以有多個 Service。

### 3. Connector
負責接收請求和發送回應:
```xml
<!-- HTTP/1.1 Connector -->
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" 
           maxThreads="200"
           minSpareThreads="10"/>
```

### 4. Engine
處理所有請求的容器,將請求路由到對應的 Host。

### 5. Host
虛擬主機,可以配置多個域名:
```xml
<Host name="www.myshop.com" appBase="webapps"
      unpackWARs="true" autoDeploy="true">
</Host>
```

### 6. Context
代表一個 Web 應用:
```xml
<Context path="/shop" docBase="myshop.war" reloadable="true"/>
```

## 請求處理流程

當使用者訪問 `http://localhost:8080/shop/products/123` 時:

```
1. Connector 接收 HTTP 請求
   ↓
2. Engine 根據域名選擇 Host
   ↓  
3. Host 根據路徑 /shop 找到 Context
   ↓
4. Context 根據 URL 模式找到對應的 Servlet
   ↓
5. Servlet 處理請求並生成回應
   ↓
6. Connector 將回應發送回客戶端
```

## 實戰案例:Tomcat 配置優化

### 場景一:調整執行緒池

電商網站流量大,需要優化 Connector 配置:

```xml
<!-- server.xml -->
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           maxThreads="500"           <!-- 最大執行緒數 -->
           minSpareThreads="25"       <!-- 最小空閒執行緒 -->
           maxConnections="10000"     <!-- 最大連線數 -->
           acceptCount="100"          <!-- 佇列大小 -->
           enableLookups="false"      <!-- 停用 DNS 查詢 -->
           compression="on"           <!-- 啟用壓縮 -->
           compressionMinSize="2048"  <!-- 壓縮最小大小 -->
           compressableMimeType="text/html,text/xml,text/plain,application/json"/>
```

### 場景二:配置 HTTPS

```xml
<Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
           maxThreads="200" scheme="https" secure="true"
           keystoreFile="conf/keystore.jks"
           keystorePass="changeit"
           clientAuth="false" sslProtocol="TLS"/>
```

### 場景三:配置多個虛擬主機

```xml
<Engine name="Catalina" defaultHost="localhost">
    
    <!-- 主站 -->
    <Host name="www.myshop.com" appBase="webapps/main"
          unpackWARs="true" autoDeploy="true">
        <Context path="" docBase="shop.war"/>
    </Host>
    
    <!-- 管理後台 -->
    <Host name="admin.myshop.com" appBase="webapps/admin"
          unpackWARs="true" autoDeploy="true">
        <Context path="" docBase="admin.war"/>
    </Host>
    
    <!-- API 服務 -->
    <Host name="api.myshop.com" appBase="webapps/api"
          unpackWARs="true" autoDeploy="true">
        <Context path="" docBase="api.war"/>
    </Host>
    
</Engine>
```

## Tomcat 執行緒模型

### BIO (Blocking I/O) - 已棄用
每個連線一個執行緒,連線多時效能差。

### NIO (Non-blocking I/O) - 推薦
```xml
<Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="200"/>
```
使用 Selector 多路復用,一個執行緒可處理多個連線。

### NIO2 (Asynchronous I/O)
```xml
<Connector port="8080" protocol="org.apache.coyote.http11.Http11Nio2Protocol"
           maxThreads="200"/>
```
非同步 I/O,效能更好。

### APR (Apache Portable Runtime)
```xml
<Connector port="8080" protocol="org.apache.coyote.http11.Http11AprProtocol"
           maxThreads="200"/>
```
使用原生程式庫,效能最佳。

## Servlet 容器職責

### 1. 管理 Servlet 生命週期
```java
public class ProductServlet extends HttpServlet {
    
    @Override
    public void init() throws ServletException {
        // Tomcat 啟動時呼叫
        System.out.println("ProductServlet 初始化");
    }
    
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
            throws ServletException, IOException {
        // 處理 GET 請求
        String productId = req.getParameter("id");
        // ...
    }
    
    @Override
    public void destroy() {
        // Tomcat 關閉時呼叫
        System.out.println("ProductServlet 銷毀");
    }
}
```

### 2. URL 映射
```xml
<!-- web.xml -->
<servlet>
    <servlet-name>ProductServlet</servlet-name>
    <servlet-class>com.example.ProductServlet</servlet-class>
</servlet>

<servlet-mapping>
    <servlet-name>ProductServlet</servlet-name>
    <url-pattern>/products/*</url-pattern>
</servlet-mapping>
```

### 3. 管理 Session
```java
HttpSession session = request.getSession();
session.setAttribute("user", user);
session.setMaxInactiveInterval(1800); // 30 分鐘
```

### 4. 安全控制
```xml
<security-constraint>
    <web-resource-collection>
        <web-resource-name>Admin</web-resource-name>
        <url-pattern>/admin/*</url-pattern>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>
```

## Tomcat 類別載入機制

```
Bootstrap ClassLoader (JDK 核心類別)
    ↓
System ClassLoader (CLASSPATH)
    ↓
Common ClassLoader (Tomcat 共用類別)
    ├── Server ClassLoader (Tomcat 內部)
    └── Shared ClassLoader
        ↓
    WebApp ClassLoader (各個 Web 應用隔離)
```

這種設計讓每個 Web 應用可以使用不同版本的程式庫而不衝突。

## 效能監控與調優

### 1. JVM 參數調整
```bash
JAVA_OPTS="-server 
           -Xms2048m 
           -Xmx2048m 
           -XX:MetaspaceSize=256m 
           -XX:MaxMetaspaceSize=512m
           -XX:+UseG1GC
           -XX:MaxGCPauseMillis=200"
```

### 2. 啟用 JMX 監控
```bash
CATALINA_OPTS="-Dcom.sun.management.jmxremote
               -Dcom.sun.management.jmxremote.port=9000
               -Dcom.sun.management.jmxremote.ssl=false
               -Dcom.sun.management.jmxremote.authenticate=false"
```

### 3. 查看執行緒狀態
```bash
# 使用 JConsole 或 VisualVM 連線
# 或使用 Tomcat Manager 應用
http://localhost:8080/manager/status
```

### 4. 啟用存取日誌
```xml
<Valve className="org.apache.catalina.valves.AccessLogValve" 
       directory="logs"
       prefix="localhost_access_log" 
       suffix=".txt"
       pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
```

## 常見問題與解決

### 問題一:記憶體洩漏
原因:未正確關閉資源、ThreadLocal 未清理

解決:
```java
// 使用 ServletContextListener 清理資源
public class AppCleanupListener implements ServletContextListener {
    
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        // 關閉資料庫連線池
        dataSource.close();
        
        // 停止背景執行緒
        scheduledExecutor.shutdown();
        
        // 清理 ThreadLocal
        threadLocalCache.remove();
    }
}
```

### 問題二:Session 過多
解決:縮短 Session 超時時間,使用 Redis 等外部儲存

### 問題三:執行緒耗盡
解決:優化業務邏輯,使用非同步處理,增加執行緒池大小

## 小結

理解 Tomcat 架構能幫助我們:
- **調優效能**:合理配置執行緒池和連線池
- **診斷問題**:理解請求處理流程
- **安全加固**:正確配置安全機制
- **監控運維**:掌握監控指標

下週我們將進入 **Spring Boot**,看看它如何內嵌 Tomcat 並簡化配置。
