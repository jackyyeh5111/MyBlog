---
title: 動手DIY改造 Asp.net MVC- 擴充在擴充,強化WebViewPage製作多國貨幣機制 (第29天)
date: 2019-10-10 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---

# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [多國路由設定](#%e5%a4%9a%e5%9c%8b%e8%b7%af%e7%94%b1%e8%a8%ad%e5%ae%9a)
- [取得貨幣符號](#%e5%8f%96%e5%be%97%e8%b2%a8%e5%b9%a3%e7%ac%a6%e8%99%9f)
- [擴充 WebViewPage<TModel>](#%e6%93%b4%e5%85%85-webviewpagetmodel)
- [RazorView使用擴充後WebViewPage<TModel>](#razorview%e4%bd%bf%e7%94%a8%e6%93%b4%e5%85%85%e5%be%8cwebviewpagetmodel)
- [小結:](#%e5%b0%8f%e7%b5%90)

## 前言

`View`頁面(`razor`,`aspx`...)都是繼承`WebViewPage<TModel>`頁面,

今天會帶大家寫一個範例完成透過`Route`判斷多國錢幣符號.

## 多國路由設定

在`Route`設定上多一個`{culture}`區塊.如果使用者沒有輸入預設使用英文(`en`).

```csharp
routes.MapRoute(
    name: "Default",
    url: "{culture}/{controller}/{action}",
    defaults: new { controller = "Home", action = "Index", culture = "en" });
```

## 取得貨幣符號

建立一個介面`ICurrency`裡面有個方法可以取得傳入國家貨幣符號.

在`CurrencyProvider`類別透過`Routes.Values["culture"]`取得使用者傳遞語系國家.

透過此參數可以知道使用者想要使用哪個國家貨幣.

```csharp
public interface ICurrency
{
    string GetCurrencySymbol();
}

public class CurrencyProvider : ICurrency
{
    public string GetCurrencySymbol()
    {
        HttpContextBase contextWrapper = new HttpContextWrapper(HttpContext.Current);

        string culture = RouteTable.Routes.GetRouteData(contextWrapper)?.Values["culture"] as string;

        return GetSymbol(culture);
    }

    private string GetSymbol(string culture)
    {
        switch (culture)
        {
            case "en":
                return "$";
            case "eu":
                return "£";
            default:
                return "$";
        }
    }
}
```

## 擴充 WebViewPage<TModel> 

在`Autofac`多註冊一個

```csharp
builder.RegisterType<CurrencyProvider>().As<ICurrency>();
DependencyResolver.SetResolver(new CustomerDependencyResolver(builder.Build()));
```

最後在建立一個`CountryViewPage<TModel>`抽象類別繼承於`WebViewPage<TModel>`.

在此類別中建立一個`ICurrency`屬性,並在建構子中透過`DependencyResolver.Current.GetService`給值

> 因為這間已經替換成`Autofac`解析器,所以會吃`Autofac`註冊的類別.

```csharp
public abstract class CountryViewPage<TModel> : WebViewPage<TModel>
{
    public CountryViewPage()
    {
        Currency = DependencyResolver.Current.GetService<ICurrency>();
    }
    public ICurrency Currency { get;  }
}
```

## RazorView使用擴充後WebViewPage<TModel>

在`View`上使用新`WebViewPage<TModel>`只需要在最上面加`@inherits CountryViewPage<object>`.
我們就可以透過`@`呼叫`Currency`物件.

```csharp
@inherits CountryViewPage<object>

@{
    ViewBag.Title = "About";
}

<h2>@ViewBag.Title.</h2>
<h3>@ViewBag.Message @Currency.GetCurrencySymbol()</h3>

<p>Use this area to provide additional information.</p>
```

如果每個頁面都需要使用新的`WebViewPage<TModel>`可以透過`web.config`新增加一個`<pages pageBaseType="CountryViewPage">`將`Razor`產生的`C#`程式碼繼承於此類別

```xml
<system.web.webPages.razor>
    <pages pageBaseType="CountryViewPage">
        <namespaces>
          <add namespace="System.Web.Mvc" />
          <add namespace="System.Web.Mvc.Ajax" />
          <add namespace="System.Web.Mvc.Html" />
          <add namespace="System.Web.Routing" />
        </namespaces>
    </pages>
</system.web.webPages.razor>
```

## 小結:

其實我們也可以繼承`WebViewPage<TModel>`來擴充`View`多變性

這邊有一個題目提供讀者來完成透過上面概念完成多國語系,這裡提供一條方法完成

> 寫一個`string transfer(string key)`透過`Resource`檔案來完成;

[Github範例程式原始碼](https://github.com/isdaniel/ItHelp_MVC_10th/tree/CustomerWebViewPage) `CustomerWebViewPage`分支上

