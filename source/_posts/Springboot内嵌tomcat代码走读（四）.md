---
title: Springboot内嵌tomcat代码走读（四）
date: 2020-10-21 10:04:00
tags:
  - Java
  - 源码阅读
  - Spring
categories:
  - 源码阅读
  - Spring
  - 技术
---

上篇文章我们走读了 servlet 生命周期中的创建和初始化以及 servlet 中 Filter 在 springboot 中的使用。本文继续进行 servlet 生命周期中下面的部分，提供服务即 service 方法。

<!--more-->

Servlet 的 service 方法在 HttpServlet 中实现，代码为：

```
    @Override
    public void service(ServletRequest req, ServletResponse res)
        throws ServletException, IOException {

        HttpServletRequest  request;
        HttpServletResponse response;

        try {
            request = (HttpServletRequest) req;
            response = (HttpServletResponse) res;
        } catch (ClassCastException e) {
            throw new ServletException(lStrings.getString("http.non_http"));
        }
        service(request, response);
    }
```

```
	@Override
	protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
		if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
			processRequest(request, response);
		}
		else {
			super.service(request, response);
		}
	}
```

```
protected void service(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {

        String method = req.getMethod();

        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                // servlet doesn't support if-modified-since, no reason
                // to go through further expensive logic
                doGet(req, resp);
            } else {
                long ifModifiedSince;
                try {
                    ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                } catch (IllegalArgumentException iae) {
                    // Invalid date header - proceed as if none was set
                    ifModifiedSince = -1;
                }
                if (ifModifiedSince < (lastModified / 1000 * 1000)) {
                    // If the servlet mod time is later, call doGet()
                    // Round down to the nearest second for a proper compare
                    // A ifModifiedSince of -1 will always be less
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
                }
            }

        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);

        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);

        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);

        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);

        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req,resp);

        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req,resp);

        } else {
            //
            // Note that this means NO servlet supports whatever
            // method was requested, anywhere on this server.
            //

            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);

            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
        }
    }
```

罗列了上面三段代码后，发现，我们就进入到了`doXXX`方法了，这里，我们选`doGet`方法来走读代码。
`doGet`方法的实现在类 FrameworkServlet 中。

```
	@Override
	protected final void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		processRequest(request, response);
	}
```

```
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		......
		try {
            //真正执行操作。
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
			logResult(request, response, failureCause, asyncManager);
            //广播一个请求处理完成的事件，这个是一个可扩展的口子
			publishRequestHandledEvent(request, response, startTime, failureCause);
		}
	}

```

DispatcherServlet 中实现了`doService`.

```
@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		......
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

		RequestPath requestPath = null;
		if (this.parseRequestPath && !ServletRequestPathUtils.hasParsedRequestPath(request)) {
			requestPath = ServletRequestPathUtils.parseAndCache(request);
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
			if (requestPath != null) {
				ServletRequestPathUtils.clearParsedRequestPath(request);
			}
		}
	}
```

`doDispatch`方法实现：

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
            //定义视图与模型对象
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
                //判断是否为文件类操作
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
                //获取处理器。
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
                //获取处理器适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

                //我们定义的interceptor的preHandler方法调用
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
                //真正调用处理器，返回modelAndView
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
                //调用interceptor的postHandler方法。
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
            //处理结果，将ModelAndView解析成view，或者异常的处理
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
            //inteceptor中preHandler和postHandler一一对应
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

接下来，我们看每步操作的具体实现。

##### Handler 的获取

先看 getHandler 方法的代码：

```
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

这里从 handlerMappings 中获取 mapping，handlerMappings 值是有顺序的，排第一的一般是 RequestMappingHandlerMapping,这里 handlerMappings 在前文中说的`initHandlerMappings`里做的初始化，主要取 HandlerMapping 类型的 bean。默认情况下，springboot 里有 5 个，分别为`RequestMappingHandlerMapping`,`WelcomePageHandlerMapping`,`BeanNameUrlHandlerMapping`,`RouterFunctionMapping`和`SimpleUrlHandlerMapping`,这些 bean 定义在 WebMvcConfiguration 及其父类中。这里就不做过详细的赘述了。
`RequestMappingHandlerMapping`比较特殊一些，这里拿出来详细看一下其 bean 的初始化过程，这个跟 getHandler 有关系。
先看类图：
![RequestMappingHandlerMapping](/images/RequestMappingHandlerMapping.png)
RequestMappingHandlerMapping 实现了 InitializingBean 接口，就是说在该 bean 初始化完成后，会调用`afterPropertiesSet`进行后置处理。
我们来看该方法的实现：

```
	public void afterPropertiesSet() {

		this.config = new RequestMappingInfo.BuilderConfiguration();
		this.config.setTrailingSlashMatch(useTrailingSlashMatch());
		this.config.setContentNegotiationManager(getContentNegotiationManager());

		if (getPatternParser() != null) {
			this.config.setPatternParser(getPatternParser());
			Assert.isTrue(!this.useSuffixPatternMatch && !this.useRegisteredSuffixPatternMatch,
					"Suffix pattern matching not supported with PathPatternParser.");
		}
		else {
			this.config.setSuffixPatternMatch(useSuffixPatternMatch());
			this.config.setRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch());
			this.config.setPathMatcher(getPathMatcher());
		}

		super.afterPropertiesSet();
	}
```

这里会调用父类的 afterPropertiesSet 方法。

```
	@Override
	public void afterPropertiesSet() {
		initHandlerMethods();
	}
	protected void initHandlerMethods() {
        //遍历工厂中所有的bean
		for (String beanName : getCandidateBeanNames()) {
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                //处理选中的bean
				processCandidateBean(beanName);
			}
		}
		handlerMethodsInitialized(getHandlerMethods());
	}
```

```
	protected void processCandidateBean(String beanName) {
		Class<?> beanType = null;
		try {
			beanType = obtainApplicationContext().getType(beanName);
		}
		catch (Throwable ex) {
			// An unresolvable bean type, probably from a lazy bean - let's ignore it.
			if (logger.isTraceEnabled()) {
				logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
			}
		}
		if (beanType != null && isHandler(beanType)) {
			detectHandlerMethods(beanName);
		}
	}
```

```
	@Override
	protected boolean isHandler(Class<?> beanType) {
		return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
				AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
	}
```

只有被 Controller 或 RequestMapping 注解的 bean，才被当成处理器。即需要业务实现的 bean，也就是我们平时写的 controller。

```
protected void detectHandlerMethods(Object handler) {
		Class<?> handlerType = (handler instanceof String ?
				obtainApplicationContext().getType((String) handler) : handler.getClass());

		if (handlerType != null) {
			Class<?> userType = ClassUtils.getUserClass(handlerType);
			Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
					(MethodIntrospector.MetadataLookup<T>) method -> {
						try {
							return getMappingForMethod(method, userType);
						}
						catch (Throwable ex) {
							throw new IllegalStateException("Invalid mapping on handler class [" +
									userType.getName() + "]: " + method, ex);
						}
					});
			if (logger.isTraceEnabled()) {
				logger.trace(formatMappings(userType, methods));
			}
			methods.forEach((method, mapping) -> {
				Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
				registerHandlerMethod(handler, invocableMethod, mapping);
			});
		}
	}
```

`registerHandlerMethod`会调用`AbstractHandlerMethodMapping`的 register 方法，将 handler 注册到 mapping 中。
此后，使用 getHandler 时，就能得到对应的 handler 了。返回头来，再继续看 getHandler 方法。

先看 getHandler 方法的代码：

```
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

这里，for 循环里第一个一般取的是`RequestMappingHandlerMapping`，看 RequestMappingHandlerMapping 的 getHandler 方法。

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		//在mappingRegistry中获取到处理器。
        Object handler = getHandlerInternal(request);
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}
        //封装成HandlerExecutionChain返回。
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		......
		if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
			CorsConfiguration config = getCorsConfiguration(handler, request);
			if (getCorsConfigurationSource() != null) {
				CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
				config = (globalConfig != null ? globalConfig.combine(config) : config);
			}
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}

		return executionChain;
	}
```

至此，就获取到一个 mappedHandler。下面，我们看如何获取处理器适配器。

##### HandlerAdapter 的获取

`HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());`在工厂中获取 HandlerAdapter 类型的 bean。bean 的定义也在 WebMvcAutoConfiguration 及其父类中。

##### 具体的业务处理

mappedHandler 和 HandlerAdapter 都获取完了，接下来就是具体的业务处理了。首先处理 interceptor 的 preHandler。接下来通过 HandlerAdapter 的 handle 方法通过反射的方式调用具体的处理器里的接口，返回 ModelAndView，然后处理 interceptor 的 postHandler。最后解析 ModelAndView 进行视图渲染。

自此，客户端请求在 springboot 中的流转全部完成。后面代码逻辑比较清晰，就没有专门贴代码来做详细的说明了。以后用到的时候再进行说明吧。
