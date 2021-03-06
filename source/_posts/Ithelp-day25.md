---
title:  動態產生程式碼(WebViewPage) View是如何被建立(四) (第25天)
date: 2019-10-06 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---

# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [WebViewPage](#webviewpage)
	- [呼叫WebViewPage.ExecutePageHierarchy方法時機](#%e5%91%bc%e5%8f%abwebviewpageexecutepagehierarchy%e6%96%b9%e6%b3%95%e6%99%82%e6%a9%9f)
	- [ApplicationStartPage and WebPageRenderingBase](#applicationstartpage-and-webpagerenderingbase)
- [WebViewPage vs WebViewPage<TModel>](#webviewpage-vs-webviewpagetmodel)
- [小結：](#%e5%b0%8f%e7%b5%90)

## 前言

上一篇說到最終會透過一個實現`IView`物件(Razor是透過`RazorView`)來完成,`RenderView`方法將`BuildManagerCompiledView`方法取得物件轉換型別成`WebViewPage`.

`.cshtml`最終會編譯成一個繼承`WebViewPage`檔案.

本篇會來解析`View`編譯原理

> 我有做一個可以針對於[Asp.net MVC Debugger](https://github.com/isdaniel/Asp.net-MVC-Debuger)的專案，只要下中斷點就可輕易進入Asp.net MVC原始碼.

## WebViewPage

`WebViewPage`繼承樹最頂層有個`WebPageExecutingBase`抽象類別,他擁有一個抽象方法`Execute`,`View`轉成`c#`程式會建立一個類別就會繼承於`WebViewPage`並把使用者頁面程式碼實現在`Execute`方法.

```csharp
public abstract void Execute();
```

先來看一下`View`產生的DLL檔案會放在哪裡

透過在`View`檔案上寫`@GetType().Assembly.Location`.

在頁面上顯示`DLL`存放位置,一般會放在`Temp`資料夾區中

可以根據顯示路徑找到`View`編譯成`DLL`

```csharp
public class _Page_Views_Shared__Layout_cshtml : WebViewPage<object>
{
    protected global_asax ApplicationInstance
    {
        get
        {
            return (global_asax)this.get_Context().ApplicationInstance;
        }
    }

    public override void Execute()
    {
        //... user print logic
    }
}
```

我使用`JustDecomplie`反編譯工具,查看原始碼.

下圖對於`View`檔案產生`DLL`反編譯

![WebViewPage_decompile.PNG](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/itHelp/25/WebViewPage_decompile.PNG)

透過反編譯工具可以看到原始碼,每個頁面都會產生相對應的類別並繼承於`WebViewPage<object>`類別(會因為使用泛型,因為有一個`@Model`)

> `Page_Views_Home_About_cshtml`類別命名有個規則.
> `Page_Views_{ViewfolderName}_{ViewFileName}_{ExtensionFileName}`

我目前看到的是一個`About`的`cshtml`檔案(`About.cshtml`).

看到`override void Execute()`將我們頁面上的邏輯透過`WriteLiteral`將資料寫到`Output`上,在`ApplicationStartPage`有`WriteLiteral`實作方式.

```csharp
public override void WriteLiteral(object value)
{
    Output.Write(value);
}
```

### 呼叫WebViewPage.ExecutePageHierarchy方法時機

在`RazorView`類別中的`RenderView`方法最下面有一段程式碼.

先判斷是否取得`StartPage`在呼叫`ExecutePageHierarchy`方法進行頁面的渲染.

```csharp
WebPageRenderingBase startPage = null;
if (RunViewStartPages)
{
    startPage = StartPageLookup(webViewPage, RazorViewEngine.ViewStartFileName, ViewStartFileExtensions);
}
webViewPage.ExecutePageHierarchy(new WebPageContext(context: viewContext.HttpContext, page: null, model: null), writer, startPage);
```

### ApplicationStartPage and WebPageRenderingBase

`WebViewPage`類別關係圖如下,`WebViewPage`擁有個複雜繼承樹.

![WebViewPage_UML.png](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/itHelp/25/WebViewPage_UML.png)

主要分為兩派,在微軟官網有張圖來表示上面兩個比較

![https://docs.microsoft.com/zh-tw/aspnet/web-pages/overview/ui-layouts-and-themes/18-customizing-site-wide-behavior/_static/image1.jpg](https://docs.microsoft.com/zh-tw/aspnet/web-pages/overview/ui-layouts-and-themes/18-customizing-site-wide-behavior/_static/image1.jpg)

* `ApplicationStartPage`:當站台被啟動時會使用`WebPageHttpModule`(`IHttpModule`)初始化並呼叫`ApplicationStartPage`的`ExecuteStartPageInternal`找尋`_appstart.cshtml`檔案來執行.[ASP.NET Web Pages (Razor) 網站的自訂全網站行為](https://docs.microsoft.com/zh-tw/aspnet/web-pages/overview/ui-layouts-and-themes/18-customizing-site-wide-behavior)

> `_AppStart.cshtml`頁面上運作。 當要求傳入頁面中，和如果這是第一個要求任何頁面在網站中，`ASP.NET`會先檢查是否 `_AppStart.cshtml`頁面存在。 如果是的話，任何程式碼中 `_AppStart.cshtml`頁面上執行，並執行要求的頁面。

* `WebPageRenderingBase`:透過`ExecutePageHierarchy`呼叫`BaseLayout`頁面或執行請求`Execute`方法

`WebPageBase`類別中`ExecutePageHierarchy`重載實作,透過`ExecutePageHierarchy`呼叫開發者實現`Execute`方法(`Page_Views_Home_About_cshtml.Execute`方法).

```csharp
public override void ExecutePageHierarchy()
{
	if (WebPageHttpHandler.ShouldGenerateSourceHeader(Context))
	{
		try
		{
			string vp = VirtualPath;
			if (vp != null)
			{
				string path = Context.Request.MapPath(vp);
				if (!path.IsEmpty())
				{
					PageContext.SourceFiles.Add(path);
				}
			}
		}
		catch
		{
			// we really don't care if this ever fails, so we swallow all exceptions
		}
	}

	TemplateStack.Push(Context, this);
	try
	{
		// Execute the developer-written code of the WebPage
		Execute();
	}
	finally
	{
		TemplateStack.Pop(Context);
	}
}
```

## WebViewPage vs WebViewPage<TModel>

`c#` 有一個關鍵字`new`對於類別成員修飾詞

> `new`關鍵字做為宣告修飾詞使用時，會明確隱藏繼承自基底類別的成員。當您隱藏繼承的成員時，該成員的衍生版本就會取代基底類別版本
> 
[new 修飾詞 (C# 參考)](https://docs.microsoft.com/zh-tw/dotnet/csharp/language-reference/keywords/new-modifier)

我覺得這個關鍵字有點打壞物件導向的概念,因為他會把父類別原本的成員隱藏起來.強制替換成子類.

但我看到`WebViewPage<TModel>`實作時覺得`new`原來可以這麼好用

`WebViewPage<TModel>`很巧妙使用`new`把`View`重點成員物件轉成泛型.可以讓我們在`Razor`或`aspx`可以更方便使用.

```csharp
public abstract class WebViewPage<TModel> : WebViewPage
{
	private ViewDataDictionary<TModel> _viewData;

	public new AjaxHelper<TModel> Ajax { get; set; }

	public new HtmlHelper<TModel> Html { get; set; }

	public new TModel Model
	{
		get { return ViewData.Model; }
	}

	[SuppressMessage("Microsoft.Usage", "CA2227:CollectionPropertiesShouldBeReadOnly", Justification = "This is the mechanism by which the ViewPage gets its ViewDataDictionary object.")]
	public new ViewDataDictionary<TModel> ViewData
	{
		get
		{
			if (_viewData == null)
			{
				SetViewData(new ViewDataDictionary<TModel>());
			}
			return _viewData;
		}
		set { SetViewData(value); }
	}
}
```

所以之後如果有遇到類似情況(需要使用泛型替代父類別`object`類型成員可以考慮使用`new`)

## 小結：

`View`頁面程式會轉成一個類別繼承於`WebViewPage`抽象類別,並把我們撰寫邏輯填充在`Execute`方法中.讓**Asp.net MVC**來呼叫.

這裡設計非常巧妙透過一個抽象類別和一個動態編譯程式,讓`View`更有彈性可以透過`Razor`語法實現`View`邏輯(更人性化).

`WebViewPage<TModel>`很巧妙使用`new`把`View`重點成員物件轉成泛型.可以讓我們在`Razor`或`aspx`可以更方便使用.

最後透過呼叫`ActionResult.ExecuteResult`方法將資料塞到`Response`物件中,提供回傳給`Client`端,最後執行資源`Release`動作.

後面幾篇會利用前面所學來改寫**MVC**框架.
