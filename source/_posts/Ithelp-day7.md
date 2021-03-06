---
title: Asp.Net重要物件HttpApplication(三) 取得執行的IHttpHandler (第7天)
date:  2019-09-18 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---
# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [呼叫HttpAppliaction取得HttpHandler並呼叫](#%e5%91%bc%e5%8f%abhttpappliaction%e5%8f%96%e5%be%97httphandler%e4%b8%a6%e5%91%bc%e5%8f%ab)
	- [MapHandlerExecutionStep程式碼解說](#maphandlerexecutionstep%e7%a8%8b%e5%bc%8f%e7%a2%bc%e8%a7%a3%e8%aa%aa)
	- [CallHandlerExecutionStep程式碼解說](#callhandlerexecutionstep%e7%a8%8b%e5%bc%8f%e7%a2%bc%e8%a7%a3%e8%aa%aa)
- [小結：](#%e5%b0%8f%e7%b5%90)

## 前言

前面和大家分享`StepManager`是如何建立管道和依序呼叫`IHttpModule`註冊事件

> 查看原始碼好站 [Reference Source](https://referencesource.microsoft.com/)

> 此文的程式碼比較多我會在原始碼上邊上說明相對應編號方便大家觀看

今天跟大家分享`HttpAppliaction`是如何找到要執行的`IHttpHandler`物件.

## 呼叫HttpAppliaction取得HttpHandler並呼叫

在`ApplicationStepManager`的`IExecutionStep`中重要的實現類別有兩個

1. `MapHandlerExecutionStep`:找到執行`IHttpHander`
2. `CallHandlerExecutionStep`

### MapHandlerExecutionStep程式碼解說

前面說過`IExecutionStep`最核心就是要找到一個`Execute`方法

`MapHandlerExecutionStep`的`Execute`方法是為了找到一個要執行的`HttpHander`

> 每次請求都會呼叫`HttpContext.Handler`屬性.

`MapHttpHandler`會依照下面權重來取得`HttpHander`物件.

1. `context.RemapHandlerInstance`如果有物件就優先返回(很重要因為這就是Asp.net MVC使用的`HttpHander`物件)
2. 透過`IHttpHandlerFactory`工廠來取得物件,依照我們在`Config`註冊的`HttpHander`對應資料
    * 副檔名`*.ashx`泛型處理常式透過`SimpleHandlerFactory`
    * 副檔名`*.aspx`泛型處理常式透過`PageHandlerFactory`

> 想知道更多可以查看`applicationhost.config`註冊表

```csharp
internal class MapHandlerExecutionStep : IExecutionStep {
    void IExecutionStep.Execute() {
        HttpContext context = _application.Context;
        HttpRequest request = context.Request;

        context.Handler = _application.MapHttpHandler(
            context,
            request.RequestType,
            request.FilePathObject,
            request.PhysicalPathInternal,
            false /*useAppConfig*/);
    }
}

internal IHttpHandler MapHttpHandler(HttpContext context, String requestType, VirtualPath path, String pathTranslated, bool useAppConfig) {

		IHttpHandler handler = (context.ServerExecuteDepth == 0) ? context.RemapHandlerInstance : null;

		using (new ApplicationImpersonationContext()) {
			// Use remap handler if possible
			if (handler != null){
				return handler;
			}

			// Map new handler
			HttpHandlerAction mapping = GetHandlerMapping(context, requestType, path, useAppConfig);

			if (mapping == null) {
				PerfCounters.IncrementCounter(AppPerfCounter.REQUESTS_NOT_FOUND);
				PerfCounters.IncrementCounter(AppPerfCounter.REQUESTS_FAILED);
				throw new HttpException(SR.GetString(SR.Http_handler_not_found_for_request_type, requestType));
			}

			// Get factory from the mapping
			IHttpHandlerFactory factory = GetFactory(mapping);

			// Get factory from the mapping
			try {
				IHttpHandlerFactory2 factory2 = factory as IHttpHandlerFactory2;

				if (factory2 != null) {
					handler = factory2.GetHandler(context, requestType, path, pathTranslated);
				}
				else {
					handler = factory.GetHandler(context, requestType, path.VirtualPathString, pathTranslated);
				}
			}
			catch (FileNotFoundException e) {
				//...丟Exception
			}

			// Remember for recycling
			if (_handlerRecycleList == null)
				_handlerRecycleList = new ArrayList();
			_handlerRecycleList.Add(new HandlerWithFactory(handler, factory));
		}

		return handler;
}
```

`MapHandlerExecutionStep`是為了找到我們要執行的`HttpHandler`物件.

```csharp
if (handler != null){
	return handler;
}
```

在一開始先判斷`handler`是否已經有值如果有就直接返回(這個很重要因為這是為什麼`MVC`,`WebAPI`可以運作且不用在`Config`設定配對`IHttpHandlerFactory`原因).

> 只需要在`MapHandlerExecutionStep`執行前將`context.RemapHandlerInstance`給一個`HttpHandler`物件即可.

### CallHandlerExecutionStep程式碼解說

`CallHandlerExecutionStep`物件透過`context.Handler`可以找到要執行的`HttpHandler`,這邊也是優先判斷是否可執行異步請求.

* 異步呼叫`beginProcessRequestDelegate`方法(此方法將實現`IHttpAsyncHandler`物件封裝成一個`Func<T, AsyncCallback, object, IAsyncResult>`委派方法),之後再調用返回一個`IAsyncResult`物件(處理後結果最後呼叫`EndProcessRequest`方法).
* 同步呼叫`ProcessRequest`:判斷`context.Handler`不是`IHttpAsyncHandler`型別就值型同步動作

```csharp
// execution step -- call HTTP handler (used to be a separate module)
internal class CallHandlerExecutionStep : IExecutionStep {
	private HttpApplication   _application;
	private AsyncCallback     _completionCallback;
	private IHttpAsyncHandler _handler;       // per call
	private AsyncStepCompletionInfo _asyncStepCompletionInfo; // per call
	private bool              _sync;          // per call

	internal CallHandlerExecutionStep(HttpApplication app) {
		_application = app;
		_completionCallback = new AsyncCallback(this.OnAsyncHandlerCompletion);
	}

    //...其他方法

	void IExecutionStep.Execute() {
		HttpContext context = _application.Context;
		IHttpHandler handler = context.Handler;

		if (handler != null && HttpRuntime.UseIntegratedPipeline) {
			IIS7WorkerRequest wr = context.WorkerRequest as IIS7WorkerRequest;
			if (wr != null && wr.IsHandlerExecutionDenied()) {
				_sync = true;
				HttpException error = new HttpException(403, SR.GetString(SR.Handler_access_denied));
				error.SetFormatter(new PageForbiddenErrorFormatter(context.Request.Path, SR.GetString(SR.Handler_access_denied)));
				throw error;
			}
		}

		if (handler == null) {
			_sync = true;
		}
		else if (handler is IHttpAsyncHandler) {
			// asynchronous handler
			IHttpAsyncHandler asyncHandler = (IHttpAsyncHandler)handler;

			_sync = false;
			_handler = asyncHandler;

			var beginProcessRequestDelegate = AppVerifier.WrapBeginMethod<HttpContext>(_application, asyncHandler.BeginProcessRequest);

			_asyncStepCompletionInfo.Reset();
			context.SyncContext.AllowVoidAsyncOperations();
			IAsyncResult ar;
			try {
				ar = beginProcessRequestDelegate(context, _completionCallback, null);
			}
			catch {
				// The asynchronous step has completed, so we should disallow further
				// async operations until the next step.
				context.SyncContext.ProhibitVoidAsyncOperations();
				throw;
			}

			bool operationCompleted;
			bool mustCallEndHandler;
			_asyncStepCompletionInfo.RegisterBeginUnwound(ar, out operationCompleted, out mustCallEndHandler);

			if (operationCompleted) {
				_sync = true;
				_handler = null; // not to remember

				context.SyncContext.ProhibitVoidAsyncOperations();

				try {
					if (mustCallEndHandler) {
						asyncHandler.EndProcessRequest(ar);
					}

					_asyncStepCompletionInfo.ReportError();
				}
				finally {
					SuppressPostEndRequestIfNecessary(context);

					context.Response.GenerateResponseHeadersForHandler();
				}
			}
		}
		else {
			// synchronous handler
			_sync = true;

			context.SyncContext.SetSyncCaller();

			try {
				handler.ProcessRequest(context);
			}
			finally {
				context.SyncContext.ResetSyncCaller();

				SuppressPostEndRequestIfNecessary(context);

				context.Response.GenerateResponseHeadersForHandler();
			}
		}
	}
}
```

## 小結：

希望可以讓大家對於為什麼`Asp.net`為何可以針對`IHttpModule`擴充且為何最後都會請求一個`IHttpHandler`有更深入的了解.

微軟透過一系列的管道設計模式提供有高度擴展的系統對外提供一個`IHttpHandler`讓我們可以客製化擴充要執行的請求.

對於此次請求又有`IHttpModule`可對於`HttpApplication`事件做擴充(透過AOP編成方式).

今天之後我們會開始講解**Asp.net MVC**相關的原始程式碼.
