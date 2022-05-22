---
layout: post 
title: "代理模式：在不侵入代碼的前提下實現權限控管與效能監控" 
date: 2017-06-21 13:40:00 +0800 
categories: [設計模式, Java] 
tags: [代理模式, 動態代理, AOP]
---

上個月的組內會役，我們討論到一個問題：AOP 的優勢，舉例情境就是，如何確保只有「具有管理員權限」的請求才能呼叫 `MemberService` 的刪除方法？

最差的做法是在每個方法的第一行寫 `if (!user.isAdmin())`。這不僅讓代碼顯得混亂，且一旦規則改變，修改量將是災難性的。

## 代理模式 (Proxy Pattern) 的解法

代理模式的精神是：**「為其他物件提供一種代理以控制對這個物件的存取。」** 代理物件在客戶端與目標物件之間起到中介的作用。

### 1. 靜態代理 (Static Proxy)

這是最直覺的實作方式。代理類別與目標類別實作同一個介面。

```java
public interface MemberService {
    void deleteMember(long id);
}

// 真實的實作類別
public class MemberServiceImpl implements MemberService {
    @Override
    public void deleteMember(long id) {
        System.out.println("從資料庫刪除會員 ID: " + id);
    }
}

// 代理類別
public class MemberServiceProxy implements MemberService {
    private MemberService target;

    public MemberServiceProxy(MemberService target) {
        this.target = target;
    }

    @Override
    public void deleteMember(long id) {
        if (checkPermission()) { // 權限檢查邏輯
            long start = System.currentTimeMillis();
            
            target.deleteMember(id); // 呼叫真實物件
            
            long end = System.currentTimeMillis();
            System.out.println("方法執行耗時：" + (end - start) + "ms");
        } else {
            System.out.println("權限不足，拒絕訪問。");
        }
    }

    private boolean checkPermission() { /* 模擬權限驗證 */ return true; }
}

```

## 2. 動態代理 (Dynamic Proxy)

靜態代理雖然好用，但如果我們有 100 個 Service，就要寫 100 個 Proxy 類別。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class PerformanceHandler implements InvocationHandler {
    private Object target;

    public PerformanceHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        
        // 執行目標方法
        Object result = method.invoke(target, args);
        
        long end = System.currentTimeMillis();
        System.out.println(method.getName() + " 執行耗時: " + (end - start) + "ms");
        return result;
    }

    // 建立代理物件的靜態工具方法
    public static Object getProxy(Object target) {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new PerformanceHandler(target)
        );
    }
}

```

### 實際應用：

```java
MemberService realService = new MemberServiceImpl();
MemberService proxy = (MemberService) PerformanceHandler.getProxy(realService);

proxy.deleteMember(12345L); // 呼叫時會自動觸發 PerformanceHandler

```

## 代理模式的價值

1. **延遲載入 (Virtual Proxy)**：在電商首頁顯示商品列表時，我們不一定要立刻加載所有大圖。可以先給一個代理圖片物件，等真正要顯示時才去讀取磁碟。
2. **遠端代理 (Remote Proxy)**：這在微服務架構中非常常見，客戶端呼叫本地代理，代理物件則透過網路與遠端服務通訊。
3. **保護代理 (Protection Proxy)**：如上述範例，根據權限控制存取。
