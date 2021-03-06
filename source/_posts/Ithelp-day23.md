---
title:  6個基本(ActionResult) View是如何被建立(二) (第23天)
date: 2019-10-04 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---

# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [6種基本的ActionResult](#6%e7%a8%ae%e5%9f%ba%e6%9c%ac%e7%9a%84actionresult)
- [ContentResult](#contentresult)
- [RedirectResult & RedirectToRouteResult](#redirectresult--redirecttorouteresult)
- [EmptyResult](#emptyresult)
- [FileResult](#fileresult)
  - [FileContentResult](#filecontentresult)
  - [FilePathResult](#filepathresult)
- [小結:](#%e5%b0%8f%e7%b5%90)

## 前言

上一篇介紹到`CreateActionResult`方法會產生一個`ActionResult`物件利用`MethodInfo`資訊.

最後透過`InvokeActionResult`來呼叫`ExecuteResult`方法來執行`ActionResult`的`ExecuteResult`方法,基本上**MVC**找到且執行**Action**方法後面就沒再做甚麼特別的事情了(後面做資源釋放...)

```csharp
protected virtual void InvokeActionResult(ControllerContext controllerContext, ActionResult actionResult)
{
	actionResult.ExecuteResult(controllerContext);
}
```

本篇來介紹常用的`ActionResult`其內部運作程式碼

> 我有做一個可以針對於[Asp.net MVC Debugger](https://github.com/isdaniel/Asp.net-MVC-Debuger)的專案，只要下中斷點就可輕易進入Asp.net MVC原始碼.

## 6種基本的ActionResult

下面這六個類別是直接繼承於`ActionResult`的類別(其中有標註**Base class**代表這是抽象類別另外有類別繼承它)

* `ContentResult`:回傳一組字串,利用`response.Write`方法
* `EmptyResult`:什麼都不動作(當`Action`回傳`void`使用)
* `FileResult(Base class)`:把檔案當作回傳
* `HttpStatusCodeResult`:回傳**HTTP**狀態碼
* `RedirectResult & RedirectToRouteResult`:使用`Response.Redirect`轉導到其他頁面
* `ViewResultBase(Base class)`:會找尋相對應`View`檔案(`cshtml`會編譯成一個`DLL`)來執行

> `ViewResultBase`會在另一篇介紹(因為機制比較複雜)

## ContentResult

在`ContentResult`有三個屬性

* `Content`:響應內容.
* `ContentType`:設置Http `Header`攔位`ContentType`
* `ContentEncoding`:設置`Encoding`方式

```CSHARP
public class ContentResult : ActionResult
{
    public string Content { get; set; }

    public Encoding ContentEncoding { get; set; }

    public string ContentType { get; set; }

    public override void ExecuteResult(ControllerContext context)
    {
        if (context == null)
        {
            throw new ArgumentNullException("context");
        }

        HttpResponseBase response = context.HttpContext.Response;

        if (!String.IsNullOrEmpty(ContentType))
        {
            response.ContentType = ContentType;
        }
        if (ContentEncoding != null)
        {
            response.ContentEncoding = ContentEncoding;
        }
        if (Content != null)
        {
            response.Write(Content);
        }
    }
}
```

`ContentResult`操作很簡單透過`response.Write`把內容`Print`出來

## RedirectResult & RedirectToRouteResult 

`RedirectResult`這個`ActionResult`如其名就是導轉頁面.

* `Permanent`:屬性判斷是否需要`Permanently`導轉頁面(**Http-Code**:`RedirectPermanent=301`,`Redirect=302`)
* `Url`:轉導的`URL`透過`UrlHelper.GenerateContentUrl`產生`URL`.(在`GenerateContentUrl`會判斷第一個字元是否是`~`波浪號,如果是代表站內導轉.)

最後利用`Permanent`布林判斷使用`RedirectPermanent`還是`Redirect`方法.

```csharp
public class RedirectResult : ActionResult
{
	public bool Permanent { get; private set; }
	
	public string Url { get; private set; }

	public override void ExecuteResult(ControllerContext context)
	{
		if (context == null)
		{
			throw new ArgumentNullException("context");
		}
		if (context.IsChildAction)
		{
			throw new InvalidOperationException(MvcResources.RedirectAction_CannotRedirectInChildAction);
		}

		string destinationUrl = UrlHelper.GenerateContentUrl(Url, context.HttpContext);
		context.Controller.TempData.Keep();

		if (Permanent)
		{
			context.HttpContext.Response.RedirectPermanent(destinationUrl, endResponse: false);
		}
		else
		{
			context.HttpContext.Response.Redirect(destinationUrl, endResponse: false);
		}
	}
}
```

> `RedirectToRouteResult`基本流程跟上面一樣只是透過`UrlHelper.GenerateUrl`產生要導轉URL

## EmptyResult

`EmptyResult`這個類別很有趣,只有override `ExecuteResult`方法但沒有實做,上篇小結有提到這裡使用一個設計模式[null object pattern](https://en.wikipedia.org/wiki/Null_object_pattern).

```csharp
public class EmptyResult : ActionResult
{
    private static readonly EmptyResult _singleton = new EmptyResult();

    internal static EmptyResult Instance
    {
        get { return _singleton; }
    }

    public override void ExecuteResult(ControllerContext context)
    {
    }
}
```

## FileResult

`FileResult`是一個抽象類別,提供一個抽象方法給`abstract void WriteFile(HttpResponseBase response)`子類提供覆寫.

有兩個類別繼承於`FileResult`抽象類別

* `FilePathResult`
* `FileContentResult`

`FileResult`抽象類別在`ExecuteResult`設置傳輸檔案需要的前置作業(設置`Content-Type`...),最後的資料傳輸透過各個子類別去實現.

> 其中`headerValue`實做Http回應擋頭對於`RFC`規範.

```csharp
// From RFC 2183, Sec. 2.3:
// The sender may want to suggest a filename to be used if the entity is
// detached and stored in a separate file. If the receiving MUA writes
// the entity to a file, the suggested filename should be used as a
// basis for the actual filename, where possible.
string headerValue = ContentDispositionUtil.GetHeaderValue(FileDownloadName);
```

```csharp
public abstract class FileResult : ActionResult
{
        private string _fileDownloadName;

        protected FileResult(string contentType)
        {
            if (String.IsNullOrEmpty(contentType))
            {
                throw new ArgumentException(MvcResources.Common_NullOrEmpty, "contentType");
            }

            ContentType = contentType;
        }

        public string ContentType { get; private set; }

        public string FileDownloadName
        {
            get { return _fileDownloadName ?? String.Empty; }
            set { _fileDownloadName = value; }
        }

        public override void ExecuteResult(ControllerContext context)
        {
            if (context == null)
            {
                throw new ArgumentNullException("context");
            }

            HttpResponseBase response = context.HttpContext.Response;
            response.ContentType = ContentType;

            if (!String.IsNullOrEmpty(FileDownloadName))
            {
                string headerValue = ContentDispositionUtil.GetHeaderValue(FileDownloadName);
                context.HttpContext.Response.AddHeader("Content-Disposition", headerValue);
            }

            WriteFile(response);
        }

        protected abstract void WriteFile(HttpResponseBase response);
    }
}
```

### FileContentResult

`FileContentResult`將檔案已位元組方式轉存給`Client`端.

透過`HttpResponseBase.OutputStream.Write`方法.

```csharp
public class FileContentResult : FileResult
{
    public FileContentResult(byte[] fileContents, string contentType)
        : base(contentType)
    {
        if (fileContents == null)
        {
            throw new ArgumentNullException("fileContents");
        }

        FileContents = fileContents;
    }

    public byte[] FileContents { get; private set; }

    protected override void WriteFile(HttpResponseBase response)
    {
        response.OutputStream.Write(FileContents, 0, FileContents.Length);
    }
}
```

### FilePathResult


`FilePathResult`透過檔案名稱`FileName`將檔案提供給`Client`

藉由`HttpResponseBase.TransmitFile`方法.

```csharp
public class FilePathResult : FileResult
{
    public FilePathResult(string fileName, string contentType)
        : base(contentType)
    {
        if (String.IsNullOrEmpty(fileName))
        {
            throw new ArgumentException(MvcResources.Common_NullOrEmpty, "fileName");
        }

        FileName = fileName;
    }

    public string FileName { get; private set; }

    protected override void WriteFile(HttpResponseBase response)
    {
        response.TransmitFile(FileName);
    }
}
```

## 小結:

本篇介紹了幾個實現`ActionResult`類別,跟其內部程式碼,這裡能了解到**MVC**返回結果機於`ActionResult`方法.(這個概念我運用在`Web Api`服務,建立`ResponseBase`共同簽章,因為在做服務串接每個服務都有自己的加解密,回傳格式攔位.我可以統一透過一個`ResponseBase`類別裝載資料再藉由過濾器來幫忙組成相對應的資料回傳....)

下篇會來介紹繼承`ActionResult`最複雜的`ViewResultBase`相關程式碼.
