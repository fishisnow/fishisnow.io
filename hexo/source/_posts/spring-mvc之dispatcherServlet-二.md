---
title: spring-mvc之dispatcherServlet(二)
date: 2018-04-15 20:38:43
tags:
- spring
- java
---
dispatcherServlet在spring-mvc中负责请求处理，一个请求到web服务器，它本身就包含很多信息，比如这个请求是Get请求还是Post请求，
是同步的还是异步的，请求携带了什么参数，以及这些参数如何携带给控制器，如何根据请求来找到映射的处理器。如果对请求进行拦截如么处理。
这些顶层的设计都在dispatcherServlet如何处理request的逻辑里面。<!--more-->

### request请求处理的设计
servlet中有service()方法来处理request，在HttpServlet中则进一部封装，根据request的方式来执行不同的方法。
```java
  \\HttpServlet处理请求伪代码
    service(HttpServletRequest request, HttpServletResponse response) {
      String method = request.getRequestMethod();
      if(method.equals("GET")) {
        doGet(request,response);
      } else if(method.equals("POST")){
        doPost(request,response);
      }
    }
```
而FrameworkServlet中会将所有的请求都用processRequest方法来处理（PATCH请求除外）
```java
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
  processRequest(request, response);
}
```
FrameworkServlet中主要是获取LocaleContext和RequestAttributes并初始化到LocaleContextHolder和RequestContextHolder中
，然后将处理逻辑交由DispatcherServlet的doService方法处理，处理完成后再恢复之前的LocaleContext和RequestAttributes。
```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		long startTime = System.currentTimeMillis();
		Throwable failureCause = null;

		LocaleContext和 previousLocaleContext = LocaleContextHolder和.getLocaleContext();
		LocaleContext localeContext = buildLocaleContext(request);

		RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

		initContextHolders(request, localeContext, requestAttributes);

		try {
			doService(request, response);
		}
		catch (ServletException | IOException ex) {
			failureCause = ex;
			throw ex;
		}
		catch (Throwable ex) {
			failureCause = ex;
			throw new NestedServletException("Request processing failed", ex);
		}

		finally {
			resetContextHolders(request, previousLocaleContext, previousAttributes);
			if (requestAttributes != null) {
				requestAttributes.requestCompleted();
			}
      //... log代码
			publishRequestHandledEvent(request, response, startTime, failureCause);
		}
	}
```
这里面LocaleContextHolder和RequestContextHolder是一个抽象类，在类中声明了ThreadLocal类型的LocaleContext
和RequestAttributes，并提供了get的入口 , LocaleContext是提供获取本地信息Local的接口，ServletRequestAttributes则是提供Session,response,Attribute的获取接口。ThreadLocal我们知道，是每个线程独自拥有的。这里相当于很方便地提供了我们获取LocaleContext,RequestAttributes的入口。
最后处理请求完成后，发布一个ServletRequestHandledEvent事件。这个异步事件携带了请求的很多信息，也就是说我们可以在代码中监听这个事件来获取这些信息，比如processingTime处理时间，请求的url等。
```java
private void publishRequestHandledEvent(HttpServletRequest request, HttpServletResponse response,
			long startTime, @Nullable Throwable failureCause) {

		if (this.publishEvents && this.webApplicationContext != null) {
			// Whether or not we succeeded, publish an event.
			long processingTime = System.currentTimeMillis() - startTime;
			this.webApplicationContext.publishEvent(
					new ServletRequestHandledEvent(this,
							request.getRequestURI(), request.getRemoteAddr(),
							request.getMethod(), getServletConfig().getServletName(),
							WebUtils.getSessionId(request), getUsernameForRequest(request),
							processingTime, failureCause, response.getStatus()));
		}
	}
```

### DispatcherServlet的doService方法
request请求一上来，DispatcherServlet就把这个请求的属性保存到一个快照容器中了，并且设置了webApplicationContext,localeResolver等属性。
处理完成后又将这些属性还原。主要逻辑交给doDispatch(request, response)处理。
```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}

		try {
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
		}
	}
```
doDispatch中首先判断这个请求是否是multipart的Request,如果是上传请求的话将multipartRequestParsed置为true.请求处理完成后
根据这个标记来执行删除资源操作。接着为这个请求找到处理的handler.再根据这个handler找到对应的HandlerAdapter,然后把请求交给HandlerAdapter处理，
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());然后根据返回的ModelAndViewz找到
对应的view视图资源。在这前后有preHandler和postHandle来执行这个请求对应的Interceptor拦截器的处理方法。这里需要注意的是
如果是异步处理的话，不在这里处理请求的结果，而是finally后单独处理。
```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  HttpServletRequest processedRequest = request;
  HandlerExecutionChain mappedHandler = null;
  boolean multipartRequestParsed = false;

  WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

  try {
    ModelAndView mv = null;
    Exception dispatchException = null;

    try {
      processedRequest = checkMultipart(request);
      multipartRequestParsed = (processedRequest != request);

      // Determine handler for the current request.
      mappedHandler = getHandler(processedRequest);
      if (mappedHandler == null) {
        noHandlerFound(processedRequest, response);
        return;
      }

      // Determine handler adapter for the current request.
      HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

      // Process last-modified header, if supported by the handler.
      String method = request.getMethod();
      boolean isGet = "GET".equals(method);
      if (isGet || "HEAD".equals(method)) {
        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
        if (logger.isDebugEnabled()) {
          logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
        }
        if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
          return;
        }
      }

      if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
      }

      // Actually invoke the handler.
      mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

      if (asyncManager.isConcurrentHandlingStarted()) {
        return;
      }

      applyDefaultViewName(processedRequest, mv);
      mappedHandler.applyPostHandle(processedRequest, response, mv);
    }
    catch (Exception ex) {
      dispatchException = ex;
    }
    catch (Throwable err) {
      // As of 4.3, we're processing Errors thrown from handler methods as well,
      // making them available for @ExceptionHandler methods and other scenarios.
      dispatchException = new NestedServletException("Handler dispatch failed", err);
    }
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
  }
  catch (Exception ex) {
    triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
  }
  catch (Throwable err) {
    triggerAfterCompletion(processedRequest, response, mappedHandler,
        new NestedServletException("Handler processing failed", err));
  }
  finally {
    if (asyncManager.isConcurrentHandlingStarted()) {
      // Instead of postHandle and afterCompletion
      if (mappedHandler != null) {
        mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
      }
    }
    else {
      // Clean up any resources used by a multipart request.
      if (multipartRequestParsed) {
        cleanupMultipart(processedRequest);
      }
    }
  }
}
```
这里有个小知识点，每个get请求都会把服务器返回的lastModified带给服务器，每次服务器请求时会把请求带来的lastModified和服务器保存的进行比对，
如果返现服务器保存的时间戳比请求带来的早，则直接返回304状态码，告诉浏览器直接使用本地缓存的数据。代码如下
```java
boolean isHttpGetOrHead = SAFE_METHODS.contains(getRequest().getMethod());
if (this.notModified) {
	response.setStatus(isHttpGetOrHead ?
			HttpStatus.NOT_MODIFIED.value() : HttpStatus.PRECONDITION_FAILED.value());
}
```

### 总结
spring mvc对请求的处理流程大概是
FrameworkServlet.processedRequest->DispatcherServlet.doService->doDispatch
FrameworkServlet负责将所有的请求合并到processedRequest中，并发布处理完成事件，相当与一个外层的装饰。
doService负责设置请求的属性并在请求结束后重新还原属性。doDispatch则是对请求进行映射，找到对应的处理器处理，
并执行拦截器的方法，根据处理器返回的结果封装视图。
