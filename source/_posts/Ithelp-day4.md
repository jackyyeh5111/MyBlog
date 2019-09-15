---
title: 掌控HttpApplication物件建立 - HttpApplicationFactory (第4天)
date: 2019-09-15 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---
# Agenda<!-- omit in toc -->
- [前言：](#%e5%89%8d%e8%a8%80)
- [HttpApplication物件](#httpapplication%e7%89%a9%e4%bb%b6)
- [取得使用 HttpApplication物件 (GetApplicationInstance)](#%e5%8f%96%e5%be%97%e4%bd%bf%e7%94%a8-httpapplication%e7%89%a9%e4%bb%b6-getapplicationinstance)
- [GetNormalApplicationInstance](#getnormalapplicationinstance)
- [小結](#%e5%b0%8f%e7%b5%90)

## 前言：

附上`Asp.net`執行請求流程圖.

![瀏覽器請求IIS流程](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/IIS_Asp.net_Process.png)

在前一篇我們說到`HttpRunTime`會透過`GetApplicationInstance`來取得一個`IHttpHandler`對象.

今天跟著原始碼來了解到底回傳一個什麼`IHttpHandler`物件給`HttpRunTime`使用.

>　查看原始碼好站 [Reference Source](https://referencesource.microsoft.com/) 

## HttpApplication物件

HttpApplication是整個`ASP.NET`基礎的核心。一個HttpApplication物件在某個時刻只能處理一個請求，只有完成對某個請求處理後，該HttpApplication才能用於後續的請求的處理。

所以`ASP.NET`利用物件程序池機制來建立或者取得HttpApplication物件。具體來講,當第一個`Http`請求抵達的時候，`ASP.NET`會一次建立多個HttpApplication物件，並將其置於池中，選擇其中一個物件來處理該請求。

而如果程序池中沒有`HttpApplication`物件,`Asp.net`會建立新的`HttpApplication`物件處理請求

`HttpApplication`物件處理`Http`請求整個生命週期是一個相對複雜的過程,在該過程的不同階段會觸發相應的事件。我們可以註冊相應的事件(如同[上一篇](https://ithelp.ithome.com.tw/articles/10214999)介紹事件表)

下圖就是模擬`HttpApplication`的`ObjectPool`樣子

![HttpApplication](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/itHelp/4/objectPool.png)

## 取得使用 HttpApplication物件 (GetApplicationInstance)

```Csharp
internal static IHttpHandler GetApplicationInstance(HttpContext context) {
    if (_customApplication != null)
        return _customApplication;

    // Check to see if it's a debug auto-attach request
    if (context.Request.IsDebuggingRequest)
        return new HttpDebugHandler();

    _theApplicationFactory.EnsureInited();

    _theApplicationFactory.EnsureAppStartCalled(context);

    return _theApplicationFactory.GetNormalApplicationInstance(context);
}
```

所以最終我們是返回一個`HttpApplication`物件來使用.

## GetNormalApplicationInstance

方法中主要做.

1. 判斷`_freeList`集合中是否有可用`HttpApplication`物件(物件程序池中),如果沒有就利用`HttpRuntime.CreateNonPublicInstance(_theApplicationType)`透過反射建立一個新的`HttpApplication`返回(呼叫完`IHttpHandler.ProcessRequst`方法後會將這個物件存入`_freeList`中)，最後將

```Csharp
private HttpApplication GetNormalApplicationInstance(HttpContext context) {
    HttpApplication app = null;

    if (!_freeList.TryTake(out app)) {
        // If ran out of instances, create a new one
        app = (HttpApplication)HttpRuntime.CreateNonPublicInstance(_theApplicationType);

        using (new ApplicationImpersonationContext()) {
            app.InitInternal(context, _state, _eventHandlerMethods);
        }
    }

    if (AppSettings.UseTaskFriendlySynchronizationContext) {
        // When this HttpApplication instance is no longer in use, recycle it.
        app.ApplicationInstanceConsumersCounter = new CountdownTask(1); // representing required call to HttpApplication.ReleaseAppInstance
        app.ApplicationInstanceConsumersCounter.Task.ContinueWith((_, o) => RecycleApplicationInstance((HttpApplication)o), app, TaskContinuationOptions.ExecuteSynchronously);
    }
    return app;
}
```

## 小結

今天我們學到

1. `IHttpHandler GetApplicationInstance(HttpContext context)`其實是返回一個`HttpApplication`物件.
2. 這個工廠會有一個 `_freeList` 集合來存取之前用過的`HttpApplication`物件，如果集合中沒有適合的`HttpApplication`物件就會使用反射返回一個新的`HttpApplication`並將他初始化．
3. 所以`HttpRuntime`呼叫的是`HttpApplication`物件的`ProcessRequest`方法

下篇會跟大家介紹`HttpApplication`類別成員詳細資訊