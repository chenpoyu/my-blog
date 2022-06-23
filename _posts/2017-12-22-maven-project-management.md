---
layout: post
title: "Maven 專案管理筆記"
date: 2017-12-22 10:45:00 +0800
categories: [設計模式, Java]
tags: [Maven, 建構工具, 依賴管理]
---

開發Java專案時,我們需要管理各種jar檔、編譯、打包、測試等。Maven提供了標準化的專案管理方式,這週來整理Maven的使用經驗。

## Maven 簡介

Maven是Java專案的建構工具,主要功能:
- 依賴管理
- 專案建構
- 生命週期管理
- 多模組專案支援

### Maven 座標

每個jar檔都有唯一座標:

```xml
<dependency>
    <groupId>org.springframework</groupId>  <!-- 組織 -->
    <artifactId>spring-core</artifactId>     <!-- 專案名稱 -->
    <version>4.3.10.RELEASE</version>        <!-- 版本 -->
</dependency>
```

## pom.xml 基礎

專案的核心配置檔:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <!-- 專案座標 -->
    <groupId>com.example</groupId>
    <artifactId>ecommerce</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    
    <!-- 專案資訊 -->
    <name>電商系統</name>
    <description>電商後端服務</description>
    
    <!-- 屬性 -->
    <properties>
        <java.version>1.8</java.version>
        <spring.version>4.3.10.RELEASE</spring.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    
    <!-- 依賴 -->
    <dependencies>
        <!-- Spring Framework -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
    </dependencies>
    
    <!-- 建構配置 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## 依賴管理

### 1. 加入依賴

電商系統常用依賴:

```xml
<dependencies>
    <!-- Spring MVC -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>4.3.10.RELEASE</version>
    </dependency>
    
    <!-- MySQL Driver -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.44</version>
    </dependency>
    
    <!-- Hibernate -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>5.2.10.Final</version>
    </dependency>
    
    <!-- JSON處理 -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.8.9</version>
    </dependency>
    
    <!-- Logging -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.3</version>
    </dependency>
    
    <!-- 測試 -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 2. 依賴範圍 (scope)

```xml
<!-- compile: 預設,編譯和執行都需要 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>4.3.10.RELEASE</version>
    <scope>compile</scope>
</dependency>

<!-- test: 只在測試時需要 -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>

<!-- provided: 編譯需要,執行時由容器提供 -->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>servlet-api</artifactId>
    <version>2.5</version>
    <scope>provided</scope>
</dependency>

<!-- runtime: 編譯不需要,執行時需要 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.44</version>
    <scope>runtime</scope>
</dependency>
```

### 3. 排除依賴

當依賴有衝突時:

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>4.3.10.RELEASE</version>
    <exclusions>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 4. 依賴傳遞

```
專案 → spring-webmvc → spring-core → commons-logging
```

Maven會自動下載所有相關依賴。

查看依賴樹:
```bash
mvn dependency:tree
```

輸出範例:
```
[INFO] com.example:ecommerce:jar:1.0-SNAPSHOT
[INFO] +- org.springframework:spring-webmvc:jar:4.3.10.RELEASE:compile
[INFO] |  +- org.springframework:spring-aop:jar:4.3.10.RELEASE:compile
[INFO] |  +- org.springframework:spring-beans:jar:4.3.10.RELEASE:compile
[INFO] |  +- org.springframework:spring-context:jar:4.3.10.RELEASE:compile
[INFO] |  +- org.springframework:spring-core:jar:4.3.10.RELEASE:compile
[INFO] |  |  \- commons-logging:commons-logging:jar:1.2:compile
[INFO] |  +- org.springframework:spring-expression:jar:4.3.10.RELEASE:compile
[INFO] |  \- org.springframework:spring-web:jar:4.3.10.RELEASE:compile
```

### 5. 版本管理

用properties集中管理版本:

```xml
<properties>
    <spring.version>4.3.10.RELEASE</spring.version>
    <hibernate.version>5.2.10.Final</hibernate.version>
    <mysql.version>5.1.44</mysql.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>${spring.version}</version>
    </dependency>
    
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>${spring.version}</version>
    </dependency>
    
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>${hibernate.version}</version>
    </dependency>
</dependencies>
```

## Maven 生命週期

### 三個生命週期

1. **clean** - 清理專案
2. **default** - 建構專案
3. **site** - 產生專案文件

### default 生命週期的階段

```
validate → compile → test → package → verify → install → deploy
```

常用指令:

```bash
# 清理 (刪除 target 目錄)
mvn clean

# 編譯
mvn compile

# 執行測試
mvn test

# 打包 (產生 jar 或 war)
mvn package

# 安裝到本地倉庫
mvn install

# 部署到遠端倉庫
mvn deploy

# 組合使用
mvn clean package
mvn clean install
```

### 跳過測試

```bash
# 完全跳過測試
mvn package -DskipTests

# 跳過測試編譯和執行
mvn package -Dmaven.test.skip=true
```

## 多模組專案

電商系統拆分成多個模組:

```
ecommerce-parent/
├── pom.xml (父POM)
├── ecommerce-common/     (共用工具)
│   └── pom.xml
├── ecommerce-product/    (商品服務)
│   └── pom.xml
├── ecommerce-order/      (訂單服務)
│   └── pom.xml
└── ecommerce-user/       (用戶服務)
    └── pom.xml
```

### 父POM

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>ecommerce-parent</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    
    <!-- 子模組 -->
    <modules>
        <module>ecommerce-common</module>
        <module>ecommerce-product</module>
        <module>ecommerce-order</module>
        <module>ecommerce-user</module>
    </modules>
    
    <!-- 統一依賴管理 -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-context</artifactId>
                <version>4.3.10.RELEASE</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

### 子模組 POM

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    
    <!-- 繼承父POM -->
    <parent>
        <groupId>com.example</groupId>
        <artifactId>ecommerce-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    
    <artifactId>ecommerce-product</artifactId>
    <packaging>jar</packaging>
    
    <dependencies>
        <!-- 依賴其他模組 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>ecommerce-common</artifactId>
            <version>${project.version}</version>
        </dependency>
        
        <!-- 不用寫版本,從父POM繼承 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
        </dependency>
    </dependencies>
</project>
```

建構時在父目錄執行:
```bash
mvn clean install
```

Maven會按順序建構所有模組。

## 常用插件

### 1. Compiler 插件

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.6.1</version>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
```

### 2. Surefire 插件 (測試)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.20</version>
    <configuration>
        <!-- 測試失敗繼續建構 -->
        <testFailureIgnore>true</testFailureIgnore>
        <!-- 指定測試類別 -->
        <includes>
            <include>**/*Test.java</include>
        </includes>
    </configuration>
</plugin>
```

### 3. Assembly 插件 (打包)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <archive>
            <manifest>
                <mainClass>com.example.Application</mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 4. Tomcat 插件

```xml
<plugin>
    <groupId>org.apache.tomcat.maven</groupId>
    <artifactId>tomcat7-maven-plugin</artifactId>
    <version>2.2</version>
    <configuration>
        <port>8080</port>
        <path>/</path>
    </configuration>
</plugin>
```

執行:
```bash
mvn tomcat7:run
```

## Profile 環境配置

不同環境用不同配置:

```xml
<profiles>
    <!-- 開發環境 -->
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <db.url>jdbc:mysql://localhost:3306/ecommerce_dev</db.url>
            <db.username>dev</db.username>
            <db.password>dev123</db.password>
        </properties>
    </profile>
    
    <!-- 測試環境 -->
    <profile>
        <id>test</id>
        <properties>
            <db.url>jdbc:mysql://test-db:3306/ecommerce_test</db.url>
            <db.username>test</db.username>
            <db.password>test123</db.password>
        </properties>
    </profile>
    
    <!-- 生產環境 -->
    <profile>
        <id>prod</id>
        <properties>
            <db.url>jdbc:mysql://prod-db:3306/ecommerce</db.url>
            <db.username>prod</db.username>
            <db.password>${env.DB_PASSWORD}</db.password>
        </properties>
    </profile>
</profiles>
```

使用:
```bash
mvn clean package -P prod
```

在配置檔中使用:
```properties
# application.properties
db.url=${db.url}
db.username=${db.username}
db.password=${db.password}
```

啟用資源過濾:
```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

## 倉庫管理

### 本地倉庫

預設位置: `~/.m2/repository`

修改位置 (in `~/.m2/settings.xml`):
```xml
<localRepository>/path/to/local/repo</localRepository>
```

### 遠端倉庫

pom.xml中配置:

```xml
<repositories>
    <repository>
        <id>central</id>
        <url>https://repo.maven.apache.org/maven2</url>
    </repository>
    
    <!-- 公司私有倉庫 -->
    <repository>
        <id>company-repo</id>
        <url>http://nexus.company.com/repository/maven-public/</url>
    </repository>
</repositories>
```

### Mirror 鏡像

settings.xml中配置(加速下載):

```xml
<mirrors>
    <mirror>
        <id>alimaven</id>
        <name>aliyun maven</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
</mirrors>
```

## 實用技巧

### 1. 查看依賴

```bash
# 依賴樹
mvn dependency:tree

# 分析依賴
mvn dependency:analyze

# 解決衝突
mvn dependency:tree -Dverbose
```

### 2. 更新依賴

```bash
# 更新SNAPSHOT版本
mvn clean install -U

# 查看可更新的依賴
mvn versions:display-dependency-updates
```

### 3. 產生專案

```bash
# 從archetype產生專案
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=my-app \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

### 4. 執行Main方法

```bash
mvn exec:java -Dexec.mainClass="com.example.Application"
```

## 常見問題

### 1. 依賴下載失敗

原因: 網路問題或倉庫錯誤

解決:
- 檢查網路
- 刪除 ~/.m2/repository 對應目錄重新下載
- 使用鏡像

### 2. 依賴衝突

原因: 不同jar依賴了同一個jar的不同版本

解決:
```bash
mvn dependency:tree
```
找出衝突,用exclusion排除。

### 3. 測試失敗

跳過測試:
```bash
mvn package -DskipTests
```

## 小結

Maven提供了:
- 標準化專案結構
- 依賴管理和版本控制
- 建構生命週期
- 多模組支援
- 插件擴展

掌握Maven讓專案管理更輕鬆,團隊協作更順暢。

下週我們將進行 **年終回顧**,總結這一年的學習!
