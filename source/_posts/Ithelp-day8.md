---
title:  揭密Mvc使用IHttpHandler by UrlRoutingModule-4.0 (第8天)
date: 2019-09-19 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---
# Agenda<!-- omit in toc -->
- [前言：](#%e5%89%8d%e8%a8%80)
- [UrlRoutingModule-4.0](#urlroutingmodule-40)
	- [OnApplicationPostResolveRequestCache事件](#onapplicationpostresolverequestcache%e4%ba%8b%e4%bb%b6)
	- [PostResolveRequestCache方法](#postresolverequestcache%e6%96%b9%e6%b3%95)
	- [IRouteHandler取得執行HttpHandler](#iroutehandler%e5%8f%96%e5%be%97%e5%9f%b7%e8%a1%8chttphandler)
	- [RemapHandler設置HttpContext的HttpHandler](#remaphandler%e8%a8%ad%e7%bd%aehttpcontext%e7%9a%84httphandler)
- [小結](#%e5%b0%8f%e7%b5%90)

## 前言：

前面幾篇文章已經詳細分享解說`Asp.net`如何透過`HttpApplication`找到`IHttpHandler`並執行呼叫介面方法.

![瀏覽器請求IIS流程](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/IIS_Asp.net_Process.png)

今天要跟大家分享上圖的最後一塊拼圖揭密並探索`Asp.net MVC`使用的`IHttpHandler`.

## UrlRoutingModule-4.0

在標題已經透漏我們是透過`UrlRoutingModule`這個繼承`IHttpModule`的類別來取得`IHttpHandler`

有人可能會有疑問是我明明沒有註冊此`HttpModule` `Asp.net`怎麼知道的呢?

原因是這個`Module`是預設就載入

下圖是一般IIS預設載入的`HttpModule`可以看到`UrlRoutingModule`已經在裡面了.

![7-MVCModule.PNG](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/7-MVCModule.PNG)

另外我們也可以看`applicationhost.config`檔案,也可以看到`UrlRoutingModule-4.0`也已經在裡面了.

我們可以發現他是在`System.Web.Routing`這個命名空間下.

```xml
<modules>
	....
	<add name="ServiceModel-4.0" type="System.ServiceModel.Activation.ServiceHttpModule,System.ServiceModel.Activation,Version=4.0.0.0,Culture=neutral,PublicKeyToken=31bf3856ad364e35" preCondition="managedHandler,runtimeVersionv4.0" />
	<add name="UrlRoutingModule-4.0" type="System.Web.Routing.UrlRoutingModule" preCondition="managedHandler,runtimeVersionv4.0" />
</modules>
```

此連結可以看到 [UrlRoutingModule 原始碼](https://referencesource.microsoft.com/#System.Web/Routing/UrlRoutingModule.cs,9b4115ad16e4f4a1)

```csharp
protected virtual void Init(HttpApplication application) {

	// Check if this module has been already addded
	if (application.Context.Items[_contextKey] != null) {
		return; // already added to the pipeline
	}
	application.Context.Items[_contextKey] = _contextKey;

	application.PostResolveRequestCache += OnApplicationPostResolveRequestCache;
}
```

前面文章有說道`Init`方法會在`HttpApplication`呼叫`InitInternal`方法時被呼叫.

這這裡可看到`application.PostResolveRequestCache`多註冊一個`OnApplicationPostResolveRequestCache`事件.

讓我們來看看此事件做了什麼事情

### OnApplicationPostResolveRequestCache事件

`OnApplicationPostResolveRequestCache`方法中,利用 `HttpContextWrapper`轉接器模式把`app.Context`轉接成一個可接受`HttpContextBase`物件,並呼叫傳入`PostResolveRequestCache`方法中.

```csharp
private void OnApplicationPostResolveRequestCache(object sender, EventArgs e) {
	HttpApplication app = (HttpApplication)sender;
	HttpContextBase context = new HttpContextWrapper(app.Context);
	PostResolveRequestCache(context);
}
```

### PostResolveRequestCache方法

```csharp
public virtual void PostResolveRequestCache(HttpContextBase context) {
	// Match the incoming URL against the route table
	RouteData routeData = RouteCollection.GetRouteData(context);

	// Do nothing if no route found
	if (routeData == null) {
		return;
	}

	// If a route was found, get an IHttpHandler from the route's RouteHandler
	IRouteHandler routeHandler = routeData.RouteHandler;

     //... 判斷 error 程式碼

	if (routeHandler is StopRoutingHandler) {
		return;
	}

	RequestContext requestContext = new RequestContext(context, routeData);

	// Dev10 766875	Adding RouteData to HttpContext
	context.Request.RequestContext = requestContext;

	IHttpHandler httpHandler = routeHandler.GetHttpHandler(requestContext);

    //... 判斷 error 程式碼

	// Remap IIS7 to our handler
	context.RemapHandler(httpHandler);
}
```

`RouteCollection`是一個全域路由集合,註冊使用路由(`Asp.net Global.cs`中我們很常看到使用).

> 對於此集合註冊路由,是`MVC`,`WebApi`能運行的關鍵喔

在`MVC`中我們透過`MapRoute`擴展方法來註冊路由,其實在這個擴展方法中會建立一個`Route`物件並加入`RouteCollection`集合中.

> `Route`物件會提供一個`HttpHandler`來給我們呼叫使用.

```csharp
routes.MapRoute(
    name: "Default",
    url: "{controller}/{action}/{id}",
    defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional }
);
```

`RouteCollection.GetRouteData(context)`取得路由中匹配此次請求的路由資料，藉由此註冊進集合並繼承`RouteBase`抽象類別的物件

### IRouteHandler取得執行HttpHandler

在`routeData`會有一個重要的屬性`RouteHandler`是繼承於`IRouteHandler`

這個介面只有一個方法就是回傳`IHttpHandler`看到這基本上就可以知道`MVC`的`IHttpHandler`是呼叫`RouteHandler.GetHttpHandler`回傳的物件.

```csharp
public interface IRouteHandler {
    IHttpHandler GetHttpHandler(RequestContext requestContext);
}
```

> 後面會對於此介面有更詳細介紹

### RemapHandler設置HttpContext的HttpHandler

在`PostResolveRequestCache`最後面幾段程式碼,是透過`routeHandler.GetHttpHandler(requestContext)`取得`IHttpHandler`，並將其設置給`context`

```csharp
IHttpHandler httpHandler = routeHandler.GetHttpHandler(requestContext);

// Remap IIS7 to our handler
context.RemapHandler(httpHandler);
```

這邊說明一下`RemapHandler`作用,最主要是把傳入參數`handler`傳給`_remapHandler`欄位

```csharp
public void RemapHandler(IHttpHandler handler) {
    EnsureHasNotTransitionedToWebSocket();

    IIS7WorkerRequest wr = _wr as IIS7WorkerRequest;

    if (wr != null) {
        // Remap handler not allowed after ResolveRequestCache notification
        if (_notificationContext.CurrentNotification >= RequestNotification.MapRequestHandler) {
            throw new InvalidOperationException(SR.GetString(SR.Invoke_before_pipeline_event, "HttpContext.RemapHandler", "HttpApplication.MapRequestHandler"));
        }

        string handlerTypeName = null;
        string handlerName = null;

        if (handler != null) {
            Type handlerType = handler.GetType();

            handlerTypeName = handlerType.AssemblyQualifiedName;
            handlerName = handlerType.FullName;
        }

        wr.SetRemapHandler(handlerTypeName, handlerName);
    }

    _remapHandler = handler;
}
```

`_remapHandler`就是`RemapHandlerInstance`屬性回傳的值

```csharp
internal IHttpHandler RemapHandlerInstance {
    get {
        return _remapHandler;
    }
}
```

> 我們之前有分享`MapHandlerExecutionStep`,`MapHttpHandler`會優先讀取存在`context.RemapHandlerInstance`中`HttpHandler`如果有物件就給`CallHandlerExecutionStep`呼叫使用.

這邊算是比較完整圓了上一篇埋的小伏筆.

## 小結

今天談到我們了解到

1. MVC是透過`UrlRoutingModule-4.0`這個HttpModule取得`HttpHandler`
2. `MVC`是在`application.PostResolveRequestCache`這個事件決定使用的`HttpHandler`
3. 路由其實是`Asp.net MVC`呼叫的關鍵
4. 因為在`MapHandlerExecutionStep`執行前已經決定`context.RemapHandlerInstance`所以就不會呼叫到`config`設定`HttpHander`物件

基本上`Asp.net`部分已經介紹完了,接下來會進入`Asp.net MVC`的世界.
