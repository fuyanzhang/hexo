---
title: Springboot内嵌tomcat代码走读（三）
date: 2020-10-12 18:25:10
tags:
  - Java
  - 源码阅读
  - Spring
categories:
  - 源码阅读
  - Spring
  - 技术
---

上篇文章走读了 springboot 中消息处理进入到 servlet 里了，这次，具体走读了请求消息在 servlet 里是怎么处理的。这里主要补充 servlet 和 springmvc 相关的知识。

<!--more-->

### servlet 相关知识

#### servlet 生命周期

servlet 生命周期在 servlet 的代码里能很清楚的体现出来。代码为：

```

public interface Servlet {

    //servlet 初始化，这里摘录了部分注释
    /**
     * The servlet container calls the <code>init</code> method exactly once
     * after instantiating the servlet. The <code>init</code> method must
     * complete successfully before the servlet can receive any requests.
    */
    public void init(ServletConfig config) throws ServletException;

    /**
     *
     * Returns a {@link ServletConfig} object, which contains initialization and
     * startup parameters for this servlet. The <code>ServletConfig</code>
     * object returned is the one passed to the <code>init</code> method.
     *
     * <p>
     * Implementations of this interface are responsible for storing the
     * <code>ServletConfig</code> object so that this method can return it. The
     * {@link GenericServlet} class, which implements this interface, already
     * does this.
     *
     * @return the <code>ServletConfig</code> object that initializes this
     *         servlet
     *
     * @see #init
     */
    public ServletConfig getServletConfig();

    /**
     * Called by the servlet container to allow the servlet to respond to a
     * request.
     *
     * <p>
     * This method is only called after the servlet's <code>init()</code> method
     * has completed successfully.
     *
     * <p>
     * The status code of the response always should be set for a servlet that
     * throws or sends an error.
     */
     //真正的处理请求
    public void service(ServletRequest req, ServletResponse res)
            throws ServletException, IOException;

    /**
     * Returns information about the servlet, such as author, version, and
     * copyright.
     *
     * <p>
     * The string that this method returns should be plain text and not markup
     * of any kind (such as HTML, XML, etc.).
     *
     * @return a <code>String</code> containing servlet information
     */
    public String getServletInfo();

    /**
     * Called by the servlet container to indicate to a servlet that the servlet
     * is being taken out of service. This method is only called once all
     * threads within the servlet's <code>service</code> method have exited or
     * after a timeout period has passed. After the servlet container calls this
     * method, it will not call the <code>service</code> method again on this
     * servlet.
     *
     * <p>
     * This method gives the servlet an opportunity to clean up any resources
     * that are being held (for example, memory, file handles, threads) and make
     * sure that any persistent state is synchronized with the servlet's current
     * state in memory.
     */
    public void destroy();
}
```

从代码中可以看出，servlet 生命周期是 init-->service-->destory;

#### 具体过程

> 1、加载与实例化（new）

     servlet在启动时或第一次接收到请求时，会到内存中去查询一下是否有servlet实例，有则取出来，没有则new一个出来。

> 2、初始化（init）

    在创建servlet之后，会调用init方法，进行初始化，该步主要是为接收处理请求做一些必要的准备，只会执行一次。

> 3、提供服务（service）

    servlet实例接收客户端的ServletRequest信息，通过request的信息，调用相应的doXXX()方法，并返回response。

> 4、销毁（destroy）

    servlet容器在关闭时，销毁servlet实例。只会执行一次

后面以 SpringMVC 为例来具体说明 servlet 的生命周期。

### SpringMVC 相关知识

这里只简单的罗列一下 SpringMVC 中 M，V，C 的关系。如图：
![SpringMVC](/images/springMVC框架图.png)

> 1、客户端发送 request 请求进入到分发器中，DispatcherServlet 中。
> 2、分发器通过 uri 到控制器映射中去查询相应的处理器（HandlerMapping）。
> 3、分发器拿到对应的处理器之后，调用处理器响应的接口，返回数据及对应的视图。
> 4、分发器拿到处理器返回的 modelAndView 后，查找相应的视同解析器进行渲染。
> 5、视图将渲染好的结果返回给终端用户进行展示。

下面通过源码的方式看 springboot 中关于 springMVC 和 servlet 相关的实现。

### 请求消息在 SpringMVC 中的流向

#### servlet 的创建过程（new）

源码阅读接上篇文章，现在到了 servlet 的处理了。在这之前，看 servlet 在 springboot 中是如何完成实例化及初始化的。
首先看 servlet 是如何 new 出来的。springboot 中，使用的是 dispatcherServlet，DispatcherServlet 的类的继承关系图如下
![DispatcherServlet](/images/DispatcherServlet.png)
该类使用自动装配的方式初始化一个 beanName 为`dispatcherServlet`的 bean。类名为`DispatcherServletAutoConfiguration`,具体代码如下:

```
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
			DispatcherServlet dispatcherServlet = new DispatcherServlet();
			dispatcherServlet.setDispatchOptionsRequest(webMvcProperties.isDispatchOptionsRequest());
			dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
			dispatcherServlet.setThrowExceptionIfNoHandlerFound(webMvcProperties.isThrowExceptionIfNoHandlerFound());
			dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
			dispatcherServlet.setEnableLoggingRequestDetails(webMvcProperties.isLogRequestDetails());
			return dispatcherServlet;
		}
```

这里就是直接 new 出来一个 servlet，并初始化相关的配置。
这个 servlet 对象是如何注册到 tomcat 容器里呢，这里就有下面一个名称为`dispatcherServletRegistration`的 bean 来做的事情了，具体代码为：

```
@Configuration(proxyBeanMethods = false)
	@Conditional(DispatcherServletRegistrationCondition.class)
	@ConditionalOnClass(ServletRegistration.class)
	@EnableConfigurationProperties(WebMvcProperties.class)
	@Import(DispatcherServletConfiguration.class)
	protected static class DispatcherServletRegistrationConfiguration {

		@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
				WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
			DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
					webMvcProperties.getServlet().getPath());
			registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
			registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
			multipartConfig.ifAvailable(registration::setMultipartConfig);
			return registration;
		}

	}
```

这里需要看一下`DispatcherServletRegistrationBean`类的继承关系图了。
![DispatcherServletRegistrationBean](/images/DispatcherServletRegistrationBean.png)
从图中看出，`DispatcherServletRegistrationBean`实现了接口`ServletContextInitializer`，[SpringBoot 内嵌 tomcat 代码走读（一）](/2020/08/04/SpringBoot内嵌tomcat代码走读（一）/#more)中，有对这个接口的调用代码。`this.webServer = factory.getWebServer(getSelfInitializer());`在获取`webServer`时，传入想改接口的调用。
接下来我们看 DispatcherServletRegistrationBean 的`OnStartup`的实现,该方法实现在其父类`RegistrationBean`中。

```
	@Override
	public final void onStartup(ServletContext servletContext) throws ServletException {
		String description = getDescription();
		if (!isEnabled()) {
			logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
			return;
		}
		register(description, servletContext);
	}

```

主要方法`register`的实现在其子类`DynamicRegistrationBean`中。

```
	@Override
	protected final void register(String description, ServletContext servletContext) {
		D registration = addRegistration(description, servletContext);
		if (registration == null) {
			logger.info(StringUtils.capitalize(description) + " was not registered (possibly already registered?)");
			return;
		}
		configure(registration);
	}
```

其主要方法`addRegistration`在其子类`ServletRegistrationBean`中实现。

```
	@Override
	protected ServletRegistration.Dynamic addRegistration(String description, ServletContext servletContext) {
		String name = getServletName();
		return servletContext.addServlet(name, this.servlet);
	}
```

`servletContext`为`ApplicationContextFacade`，`addServlet`代码为:

```
    @Override
    public ServletRegistration.Dynamic addServlet(String servletName,
            Servlet servlet) {
        if (SecurityUtil.isPackageProtectionEnabled()) {
            return (ServletRegistration.Dynamic) doPrivileged("addServlet",
                    new Class[]{String.class, Servlet.class},
                    new Object[]{servletName, servlet});
        } else {
            return context.addServlet(servletName, servlet);
        }
    }
```

Tomcat 的 ApplicationContext 里加入 servlet。代码为:

```
 private ServletRegistration.Dynamic addServlet(String servletName, String servletClass,
            Servlet servlet, Map<String,String> initParams) throws IllegalStateException {

        if (servletName == null || servletName.equals("")) {
            throw new IllegalArgumentException(sm.getString(
                    "applicationContext.invalidServletName", servletName));
        }

        if (!context.getState().equals(LifecycleState.STARTING_PREP)) {
            //TODO Spec breaking enhancement to ignore this restriction
            throw new IllegalStateException(
                    sm.getString("applicationContext.addServlet.ise",
                            getContextPath()));
        }

        Wrapper wrapper = (Wrapper) context.findChild(servletName);

        // Assume a 'complete' ServletRegistration is one that has a class and
        // a name
        if (wrapper == null) {
            wrapper = context.createWrapper();
            wrapper.setName(servletName);
            context.addChild(wrapper);
        } else {
            if (wrapper.getName() != null &&
                    wrapper.getServletClass() != null) {
                if (wrapper.isOverridable()) {
                    wrapper.setOverridable(false);
                } else {
                    return null;
                }
            }
        }

        ServletSecurity annotation = null;
        if (servlet == null) {
            wrapper.setServletClass(servletClass);
            Class<?> clazz = Introspection.loadClass(context, servletClass);
            if (clazz != null) {
                annotation = clazz.getAnnotation(ServletSecurity.class);
            }
        } else {
            wrapper.setServletClass(servlet.getClass().getName());
            wrapper.setServlet(servlet);
            if (context.wasCreatedDynamicServlet(servlet)) {
                annotation = servlet.getClass().getAnnotation(ServletSecurity.class);
            }
        }

        if (initParams != null) {
            for (Map.Entry<String, String> initParam: initParams.entrySet()) {
                wrapper.addInitParameter(initParam.getKey(), initParam.getValue());
            }
        }

        ServletRegistration.Dynamic registration =
                new ApplicationServletRegistration(wrapper, context);
        if (annotation != null) {
            registration.setServletSecurity(new ServletSecurityElement(annotation));
        }
        return registration;
    }
```

这里主要逻辑是向 tomcat 里添加 servlet。前面我们知道 tomcat 的体系结构，为了提高扩展性，tomcat 里分层定义了很多组件，在最内层组件为 Wrapper，这里就是想 warapper 中添加 servlet。这里用默认的 wrapper，`StandardWrapper`，具体代码为：

```
    @Override
    public void setServlet(Servlet servlet) {
        instance = servlet;
    }
```

至此，tomcat 的基本功能就有了，servlet 也有了。但是，当前的 servlet 还不能用，因为从上面的 servlet 生命周期看，这只完成了第一步，还有 init 这一步没有完成。那么，init 在什么时候呢。继续看代码。
从代码中可以知道，当第一个请求过来时，servlet 才进行 init 操作。接上篇文章的请求流程图，这里看请求在 servlet 中的流向。这里流程图从 StandardWrapperValue 的 invoke 方法开始进行。
调用流程图如下：
![servlet流程图](/images/servlet流程图.png)
按照流程图，我们走读一下代码，具体看看相关的处理逻辑。
首先看`invoke`方法里的代码逻辑。

```
 public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Initialize local variables we may need
        boolean unavailable = false;
        Throwable throwable = null;
        // This should be a Request attribute...
        long t1=System.currentTimeMillis();
        requestCount.incrementAndGet();
        StandardWrapper wrapper = (StandardWrapper) getContainer();
        Servlet servlet = null;
        Context context = (Context) wrapper.getParent();

        //去掉一些与主流程关系不太大的代码
        .......

        // Allocate a servlet instance to process this request
        try {
            if (!unavailable) {
                servlet = wrapper.allocate();
            }
        } catch (UnavailableException e) {
          。。。。。。

        MessageBytes requestPathMB = request.getRequestPathMB();
        DispatcherType dispatcherType = DispatcherType.REQUEST;
        if (request.getDispatcherType()==DispatcherType.ASYNC) dispatcherType = DispatcherType.ASYNC;
        request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,dispatcherType);
        request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
                requestPathMB);
        // Create the filter chain for this request
        //获取一个ApplicationFilterChain
        ApplicationFilterChain filterChain =
                ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

        // Call the filter chain for this request
        // NOTE: This also calls the servlet's service() method
        Container container = this.container;
        try {
            if ((servlet != null) && (filterChain != null)) {

               。。。。。。

                } else {
                    // 开始处理请求
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else {
                        // 开始处理请求
                        filterChain.doFilter
                            (request.getRequest(), response.getResponse());
                    }
                }

            }
        } catch (ClientAbortException | CloseNowException e) {
            。。。。。。

        }
    }
```

上面代码比较长，做了一下删减。主要的功能是初始化 servlet（init）和具体调用 servlet 的`service`方法。
`servlet`的`init`方法在`allocate()`中调用。`service`方法在`doFilter`中调用。

#### servlet 初始化过程（init）

具体看下`allocate()`的实现。

```
 public Servlet allocate() throws ServletException {

        // If we are currently unloading this servlet, throw an exception
        if (unloading) {
            throw new ServletException(sm.getString("standardWrapper.unloading", getName()));
        }

        boolean newInstance = false;

        // If not SingleThreadedModel, return the same instance every time
        //该处并不知道singleThreadModel的值，使用默认值，在loadServlet中会有确定的赋值，但SingleThreadedModel已经被废弃，这里也走不到该分支中，所以，springboot中的servlet是一个实例。
        //当为SingleThreadedModel类型时，容器中会有多个servlet实例，默认最多20个。
        if (!singleThreadModel) {
            // Load and initialize our instance if necessary
            //前文提到了instance在bean初始化时已经将instance赋值了，这里instance！=null,但instanceInitialized=false
            if (instance == null || !instanceInitialized) {
                synchronized (this) {
                    //该分支在该处不会走到。
                    if (instance == null) {
                       。。。。。。
                    }
                    if (!instanceInitialized) {
                        //init servlet
                        initServlet(instance);
                    }
                }
            }

            if (singleThreadModel) {
                if (newInstance) {
                    // Have to do this outside of the sync above to prevent a
                    // possible deadlock
                    synchronized (instancePool) {
                        instancePool.push(instance);
                        nInstances++;
                    }
                }
            } else {
                if (log.isTraceEnabled()) {
                    log.trace("  Returning non-STM instance");
                }
                // For new instances, count will have been incremented at the
                // time of creation
                if (!newInstance) {
                    countAllocated.incrementAndGet();
                }
                return instance;
            }
        }

        synchronized (instancePool) {
            while (countAllocated.get() >= nInstances) {
                // Allocate a new instance if possible, or else wait
                if (nInstances < maxInstances) {
                    try {
                        instancePool.push(loadServlet());
                        nInstances++;
                    } catch (ServletException e) {
                        throw e;
                    } catch (Throwable e) {
                        ExceptionUtils.handleThrowable(e);
                        throw new ServletException(sm.getString("standardWrapper.allocate"), e);
                    }
                } else {
                    try {
                        instancePool.wait();
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                }
            }
            if (log.isTraceEnabled()) {
                log.trace("  Returning allocated STM instance");
            }
            countAllocated.incrementAndGet();
            return instancePool.pop();
        }
    }
```

> allocate 方法中，instance ！=null，singleThreadModel 一直都为 false，因为不会走到 loadServlet 中，singleThreadModel 的值一直为 false。容器中 servlet 实例只有一个。提一句：如果 singleThreadModel 类型为 SingleThreadModel 时，容器中可以有多个 servlet 实例，放在一个大小为 20 的栈中。SingleThreadModel 已经被废弃掉了`@deprecated As of Java Servlet API 2.4, with no direct replacement.`。
> 下面看`initServlet()`方法的处理逻辑。

```
    private synchronized void initServlet(Servlet servlet)
            throws ServletException {

        if (instanceInitialized && !singleThreadModel) return;

        // Call the initialization method of this servlet
        try {
            if( Globals.IS_SECURITY_ENABLED) {
                boolean success = false;
                try {
                    Object[] args = new Object[] { facade };
                    SecurityUtil.doAsPrivilege("init",
                                               servlet,
                                               classType,
                                               args);
                    success = true;
                } finally {
                    if (!success) {
                        // destroy() will not be called, thus clear the reference now
                        SecurityUtil.remove(servlet);
                    }
                }
            } else {
                servlet.init(facade);
            }

            instanceInitialized = true;
        } catch (UnavailableException f) {
            unavailable(f);
            throw f;
        } catch (ServletException f) {
            // If the servlet wanted to be unavailable it would have
            // said so, so do not call unavailable(null).
            throw f;
        } catch (Throwable f) {
            ExceptionUtils.handleThrowable(f);
            getServletContext().log(sm.getString("standardWrapper.initException", getName()), f);
            // If the servlet wanted to be unavailable it would have
            // said so, so do not call unavailable(null).
            throw new ServletException
                (sm.getString("standardWrapper.initException", getName()), f);
        }
    }
```

这段代码很简单，就是调用 servlet 的 init 方法。
具体的 init 方法实现在其父类`GenericServlet`中。

```
    @Override
    public void init(ServletConfig config) throws ServletException {
        this.config = config;
        this.init();
    }
```

这里的 init 方法在子类`HttpServletBean`中。

```
		// Let subclasses do whatever initialization they like.
		initServletBean();
	}
```

`initServletBean()`方法在子类`FrameworkServlet`中。

```
@Override
	protected final void initServletBean() throws ServletException {

		long startTime = System.currentTimeMillis();

		try {
			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
		}
		catch (ServletException | RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			throw ex;
		}
	}

```

`initFrameworkServlet()`是留给子类实现的一个扩展接口，默认为空实现，主要的逻辑在`initWebApplicationContext()`中，该方法主要是初始化一些策略，如文件解析器，国际化解析器，主题解析器，处理器映射器，处理器适配器，异常处理器，视图解析器等。具体的实现在子类`DispatcherServlet`中。
代码如下：

```
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
        //文件解析器，没有，返回为空。
		initMultipartResolver(context);
        //国际化解析，没有，返回默认值，AcceptHeaderLocaleResolver
		initLocaleResolver(context);
        //主题解析，没有返回默认值FixedThemeResolver
		initThemeResolver(context);
        //处理器映射，若没有，返回默认值BeanNameUrlHandlerMapping，RequestMappingHandlerMapping，RouterFunctionMapping
		initHandlerMappings(context);
        //处理器适配器，若没有，返回默认值HttpRequestHandlerAdapter，SimpleControllerHandlerAdapter，RequestMappingHandlerAdapter，HandlerFunctionAdapter
		initHandlerAdapters(context);
        //异常解析，若没有，返回默认值ExceptionHandlerExceptionResolver，ResponseStatusExceptionResolver，DefaultHandlerExceptionResolver
		initHandlerExceptionResolvers(context);
        //异常后，没有返回视图时的默认处理DefaultRequestToViewNameTranslator
		initRequestToViewNameTranslator(context);
        //视图解析，没有返回InternalResourceViewResolver
		initViewResolvers(context);
        //默认值SessionFlashMapManager
		initFlashMapManager(context);
	}
```

> 后面通过看具体的请求处理来看上面的处理器如何工作的。

到此，servlet 的 init 方法正式处理完成，就可以提供服务了。

#### servlet 提供服务过程（service）

下面，看`service`如何处理。
从前文中的代码中可以知道，在调用`service`之前，需要获取一个`ApplicationFilterChain`，先来看看`ApplicationFilterFactory`中`createFilterChain`里做了哪些事情。从类名中可以猜到，这里主要是添加一些`Filter`相关的信息。具体代码为：

```
public static ApplicationFilterChain createFilterChain(ServletRequest request,
            Wrapper wrapper, Servlet servlet) {

        // Create and initialize a filter chain object
        ApplicationFilterChain filterChain = null;
        if (request instanceof Request) {
            Request req = (Request) request;
            if (Globals.IS_SECURITY_ENABLED) {
                // Security: Do not recycle
                filterChain = new ApplicationFilterChain();
            } else {
                filterChain = (ApplicationFilterChain) req.getFilterChain();
                if (filterChain == null) {
                    filterChain = new ApplicationFilterChain();
                    req.setFilterChain(filterChain);
                }
            }
        } else {
            // Request dispatcher in use
            filterChain = new ApplicationFilterChain();
        }

        filterChain.setServlet(servlet);
        filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());

        // Acquire the filter mappings for this Context
        StandardContext context = (StandardContext) wrapper.getParent();
        //获取Filter列表
        FilterMap filterMaps[] = context.findFilterMaps();

        // If there are no filter mappings, we are done
        if ((filterMaps == null) || (filterMaps.length == 0))
            return filterChain;

        // Acquire the information we will need to match filter mappings
        DispatcherType dispatcher =
                (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);

        String requestPath = null;
        Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
        if (attribute != null){
            requestPath = attribute.toString();
        }

        String servletName = wrapper.getName();

        // Add the relevant path-mapped filters to this filter chain
        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersURL(filterMap, requestPath))
                continue;
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                    context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }

        // Add filters that match on servlet name second
        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersServlet(filterMap, servletName))
                continue;
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                    context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }

        // Return the completed filter chain
        return filterChain;
    }
```

其中比较重要的一行代码`FilterMap filterMaps[] = context.findFilterMaps();`，获取过滤器列表。

```
    @Override
    public FilterMap[] findFilterMaps() {
        return filterMaps.asArray();
    }
```

这里`filterMaps`是在什么时候进行赋值的呢？此处又会回到`ServletContextInitializer`这个接口的`onStartup`方法里了。
前文里提到过，在`createWebServer`时，有个方法引用`getSelfInitializer`。这里同样走到这段代码里。

##### Filter 相关的准备工作

我们在通常在 springboot 里定义过滤器时，可以使用如下代码：

```
    @Bean
    public FilterRegistrationBean xxxFilter() {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        XxxFilter xxxFilter = new XxxFilter();
        registrationBean.addUrlPatterns("/*");
        registrationBean.setFilter(xxxFilter);
        return registrationBean;

    }
```

看下`FilterRegistrationBean`的类图![FilterRegistrationBean](/images/FilterRegistrationBean.png)

FilterRegistrationBean 同样实现了 ServletContextInitializer 接口。
同时，我们还可以进行如下方式定义过滤器。

```
@Service
public class TestFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

这种方式没有显式的定义成一个`FilterRegistrationBean`，在代码中事实上也封装成一个`FilterRegistrationBean`。接下来看具体的代码实现。
首先看`ServletContextInitializer`的`onStartup`的调用处代码。

```
	private void selfInitialize(ServletContext servletContext) throws ServletException {
		prepareWebApplicationContext(servletContext);
		registerApplicationScope(servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
		for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
			beans.onStartup(servletContext);
		}
	}
```

`getServletContextInitializerBeans()`里从 bean 工厂里获取`ServletContextInitializer`类型的 bean，调用`onStartup`方法。

```
	protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
		return new ServletContextInitializerBeans(getBeanFactory());
	}
```

```
	public ServletContextInitializerBeans(ListableBeanFactory beanFactory,
			Class<? extends ServletContextInitializer>... initializerTypes) {
		this.initializers = new LinkedMultiValueMap<>();
		this.initializerTypes = (initializerTypes.length != 0) ? Arrays.asList(initializerTypes)
				: Collections.singletonList(ServletContextInitializer.class);
        //显式定义为ServletContextInitializer的bean
		addServletContextInitializerBeans(beanFactory);
        //未显式定义为ServletContextInitializer的bean
		addAdaptableBeans(beanFactory);
		List<ServletContextInitializer> sortedInitializers = this.initializers.values().stream()
				.flatMap((value) -> value.stream().sorted(AnnotationAwareOrderComparator.INSTANCE))
				.collect(Collectors.toList());
		this.sortedList = Collections.unmodifiableList(sortedInitializers);
		logMappings(this.initializers);
	}
```

显式定义为 ServletContextInitializer 的 bean 的处理方法：

```
	private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
		for (Class<? extends ServletContextInitializer> initializerType : this.initializerTypes) {
			for (Entry<String, ? extends ServletContextInitializer> initializerBean : getOrderedBeansOfType(beanFactory,
					initializerType)) {
				addServletContextInitializerBean(initializerBean.getKey(), initializerBean.getValue(), beanFactory);
			}
		}
	}
```

```
private void addServletContextInitializerBean(String beanName, ServletContextInitializer initializer,
			ListableBeanFactory beanFactory) {
		if (initializer instanceof ServletRegistrationBean) {
			Servlet source = ((ServletRegistrationBean<?>) initializer).getServlet();
			addServletContextInitializerBean(Servlet.class, beanName, initializer, beanFactory, source);
		}
		else if (initializer instanceof FilterRegistrationBean) {
			Filter source = ((FilterRegistrationBean<?>) initializer).getFilter();
			addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
		}
		else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
			String source = ((DelegatingFilterProxyRegistrationBean) initializer).getTargetBeanName();
			addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
		}
		else if (initializer instanceof ServletListenerRegistrationBean) {
			EventListener source = ((ServletListenerRegistrationBean<?>) initializer).getListener();
			addServletContextInitializerBean(EventListener.class, beanName, initializer, beanFactory, source);
		}
		else {
			addServletContextInitializerBean(ServletContextInitializer.class, beanName, initializer, beanFactory,
					initializer);
		}
	}
```

```
	private void addServletContextInitializerBean(Class<?> type, String beanName, ServletContextInitializer initializer,
			ListableBeanFactory beanFactory, Object source) {
		this.initializers.add(type, initializer);
		if (source != null) {
			// Mark the underlying source as seen in case it wraps an existing bean
			this.seen.add(source);
		}
		if (logger.isTraceEnabled()) {
			String resourceDescription = getResourceDescription(beanName, beanFactory);
			int order = getOrder(initializer);
			logger.trace("Added existing " + type.getSimpleName() + " initializer bean '" + beanName + "'; order="
					+ order + ", resource=" + resourceDescription);
		}
	}
```

未显式定义为 ServletContextInitializer 的 bean

```
	protected void addAdaptableBeans(ListableBeanFactory beanFactory) {
		MultipartConfigElement multipartConfig = getMultipartConfig(beanFactory);
		addAsRegistrationBean(beanFactory, Servlet.class, new ServletRegistrationBeanAdapter(multipartConfig));
		addAsRegistrationBean(beanFactory, Filter.class, new FilterRegistrationBeanAdapter());
		for (Class<?> listenerType : ServletListenerRegistrationBean.getSupportedTypes()) {
			addAsRegistrationBean(beanFactory, EventListener.class, (Class<EventListener>) listenerType,
					new ServletListenerRegistrationBeanAdapter());
		}
	}
```

```
	private <T, B extends T> void addAsRegistrationBean(ListableBeanFactory beanFactory, Class<T> type,
			Class<B> beanType, RegistrationBeanAdapter<T> adapter) {
		List<Map.Entry<String, B>> entries = getOrderedBeansOfType(beanFactory, beanType, this.seen);
		for (Entry<String, B> entry : entries) {
			String beanName = entry.getKey();
			B bean = entry.getValue();
			if (this.seen.add(bean)) {
				// One that we haven't already seen
                //通过适配器模式，进行处理。
				RegistrationBean registration = adapter.createRegistrationBean(beanName, bean, entries.size());
				int order = getOrder(bean);
				registration.setOrder(order);
				this.initializers.add(type, registration);
				if (logger.isTraceEnabled()) {
					logger.trace("Created " + type.getSimpleName() + " initializer for bean '" + beanName + "'; order="
							+ order + ", resource=" + getResourceDescription(beanName, beanFactory));
				}
			}
		}
	}
```

```
	private static class FilterRegistrationBeanAdapter implements RegistrationBeanAdapter<Filter> {

		@Override
		public RegistrationBean createRegistrationBean(String name, Filter source, int totalNumberOfSourceBeans) {
			FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<>(source);
			bean.setName(name);
			return bean;
		}

	}
```

代码比较简单，这里只做代码的罗列，不做详细说明。
接下来看`FilterRegistrationBean`里`onStartup`方法的处理逻辑，该方法的实现在其父类`RegistrationBean`中。

```
	@Override
	public final void onStartup(ServletContext servletContext) throws ServletException {
		String description = getDescription();
		if (!isEnabled()) {
			logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
			return;
		}
		register(description, servletContext);
	}
```

`register`方法在`RegistrationBean`子类`DynamicRegistrationBean`中。

```
	@Override
	protected final void register(String description, ServletContext servletContext) {
		D registration = addRegistration(description, servletContext);
		if (registration == null) {
			logger.info(StringUtils.capitalize(description) + " was not registered (possibly already registered?)");
			return;
		}
		configure(registration);
	}
```

`addRegistration`在其子类`AbstractFilterRegistrationBean`中实现。

```
	@Override
	protected Dynamic addRegistration(String description, ServletContext servletContext) {
		Filter filter = getFilter();
		return servletContext.addFilter(getOrDeduceName(filter), filter);
	}
```

`addFilter`代码走到`ApplicationContextFacade`中。

```
    @Override
    public FilterRegistration.Dynamic addFilter(String filterName,
            Filter filter) {
        if (SecurityUtil.isPackageProtectionEnabled()) {
            return (FilterRegistration.Dynamic) doPrivileged("addFilter",
                    new Class[]{String.class, Filter.class},
                    new Object[]{filterName, filter});
        } else {
            return context.addFilter(filterName, filter);
        }
    }
```

最后走到`ApplicationContext`中。

```
 private FilterRegistration.Dynamic addFilter(String filterName,
            String filterClass, Filter filter) throws IllegalStateException {


        FilterDef filterDef = context.findFilterDef(filterName);

        if (filterDef == null) {
            filterDef = new FilterDef();
            filterDef.setFilterName(filterName);
            context.addFilterDef(filterDef);
        } else {
            if (filterDef.getFilterName() != null &&
                    filterDef.getFilterClass() != null) {
                return null;
            }
        }

     ......

        return new ApplicationFilterRegistration(filterDef, context);
    }
```

最后将 FilterDef 加入到`StandradContext`的`filterDefs`中。
回过头来，看 DynamicRegistrationBean 中`register`的第二个主要调用的方法`configure()`
对于过滤器来说，实现方法在`AbstractFilterRegistrationBean`中。

```
@Override
	protected void configure(FilterRegistration.Dynamic registration) {
		super.configure(registration);
		EnumSet<DispatcherType> dispatcherTypes = this.dispatcherTypes;
		if (dispatcherTypes == null) {
			T filter = getFilter();
			if (ClassUtils.isPresent("org.springframework.web.filter.OncePerRequestFilter",
					filter.getClass().getClassLoader()) && filter instanceof OncePerRequestFilter) {
				dispatcherTypes = EnumSet.allOf(DispatcherType.class);
			}
			else {
				dispatcherTypes = EnumSet.of(DispatcherType.REQUEST);
			}
		}
		Set<String> servletNames = new LinkedHashSet<>();
		for (ServletRegistrationBean<?> servletRegistrationBean : this.servletRegistrationBeans) {
			servletNames.add(servletRegistrationBean.getServletName());
		}
		servletNames.addAll(this.servletNames);
		if (servletNames.isEmpty() && this.urlPatterns.isEmpty()) {
			registration.addMappingForUrlPatterns(dispatcherTypes, this.matchAfter, DEFAULT_URL_MAPPINGS);
		}
		else {
			if (!servletNames.isEmpty()) {
				registration.addMappingForServletNames(dispatcherTypes, this.matchAfter,
						StringUtils.toStringArray(servletNames));
			}
			if (!this.urlPatterns.isEmpty()) {
				registration.addMappingForUrlPatterns(dispatcherTypes, this.matchAfter,
						StringUtils.toStringArray(this.urlPatterns));
			}
		}
	}
```

`registration.addMappingForUrlPatterns(dispatcherTypes, this.matchAfter, DEFAULT_URL_MAPPINGS);`这里的方法将每个 filter 通过 StandardContext 的`addFilterMapBefore`方式，塞到 filterMap 中的 arrays 中。通过`findFilterMaps`就能得到所有的 filter 了。
到此，Filter 相关的准备工作完成。

> **_需要补充上面流程的流程图_**。

##### 回到 service

回到前文的`ApplicationFilterChain`的获取上。`ApplicationFilterFactory`里通过`context.findFilterMaps();`获取所有的过滤器，放入 filterChain 中。
`StandardWrapperValue`中代码继续往下走，调用了 ApplicationFilterChain 的`doFilter`方法。

```
 private void internalDoFilter(ServletRequest request,
                                  ServletResponse response)
        throws IOException, ServletException {

        // 遍历所有的filter，这里就是为啥我们在定义Filter时，为了让chain能顺利的走完需要调用chain.doFilter()的原因。
        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            try {
                Filter filter = filterConfig.getFilter();

                if (request.isAsyncSupported() && "false".equalsIgnoreCase(
                        filterConfig.getFilterDef().getAsyncSupported())) {
                    request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
                }
               。。。。。。
                } else {
                    filter.doFilter(request, response, this);
                }
            }
            。。。。。。
            }
            return;
        }

        // We fell off the end of the chain -- call the servlet instance
        try {
            。。。。。。
            if ((request instanceof HttpServletRequest) &&
                    (response instanceof HttpServletResponse) &&
                    Globals.IS_SECURITY_ENABLED ) {
                final ServletRequest req = request;
                final ServletResponse res = response;
                Principal principal =
                    ((HttpServletRequest) req).getUserPrincipal();
                Object[] args = new Object[]{req, res};
                SecurityUtil.doAsPrivilege("service",
                                           servlet,
                                           classTypeUsedInService,
                                           args,
                                           principal);
            } else {
                servlet.service(request, response);
            }
        } 。。。。。。
        。。。。。。
        }
    }

```

以上代码就进入到了 servlet 的另外一个生命周期了，提供 service 服务了。
下篇文章，我们具体看下 service 接口里的逻辑.
