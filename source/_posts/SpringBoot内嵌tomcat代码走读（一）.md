---
title: SpringBoot内嵌tomcat代码走读（一）
date: 2020-08-04 10:21:46
tags: 
- Java
- 源码阅读
- Spring
categories:
- 源码阅读
- Spring
---

花了一个礼拜的时间，大致走读了一下springboot内嵌tomcat的代码，为下一步自己实现一个web容器做知识储备。这里对走读的代码做一个大致的记录。
<!--more-->

### springboot内嵌tomcat
我们知道，springboot自动装配的相关功能封装在spring-boot-autoconfigure包中。这里通过走读spring-boot的2.4.0-M版本的代码来大致看一下tomcat的启动过程。
首先，写一个简单的demo，调用springboot的run方法，一步一步来看tomcat如何启动的。
```
@SpringBootApplication
public class TestApplication {
    public static void main(String[] args) {
        SpringApplication.run(TestApplication.class);
    }
}
```
查看run方法里做了什么。
```
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
            //封装参数
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //准备springboot运行相关的环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
            //确定spring的上下文信息，这里是webservlet类型，通过反射得到AnnotationConfigServletWebServerApplicationContext
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class<?>[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //这里真正调到spring里，进行bean的初始化
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```
这里，初始化的`context`是一个`AnnotationConfigServletWebServerApplicationContext`。我们看一下该类的继承关系。
![AnnotationConfigServletWebServerApplicationContext](/images/AnnotationConfigServletWebServerApplicationContext.png)
本篇日记重点看的就是`refreshContext(context)`，这个真正的进入spring bean的处理。也在这个方法里，进行了tomcat的初始化及启动。继续走读这个方法的代码。一步一步跟下去，就进入了`AbstractApplicationContext`的`refresh`方法中。方法的代码如下：
```
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```
通过spring的注释，可以大致猜测，关于tomcat的初始化，应该在方法`onRefresh`中实现。当然，这里是通过debug源码，看调用栈信息能确切的知道这个方法就是tomcat初始化的入口。这个方法实现在子类里。上面在创建spring的`context`时，我们看到，这里创建的是一个`ServletWebServerApplicationContext`，`onRefresh`就在该类中实现。进入该方法，看到如下代码：
```
	@Override
	protected void onRefresh() {
		super.onRefresh();
		try {
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
	}
```
`createWebServer`就是创建webserver的代码。代码：
```
private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			ServletWebServerFactory factory = getWebServerFactory();
			this.webServer = factory.getWebServer(getSelfInitializer());
			getBeanFactory().registerSingleton("webServerGracefulShutdown",
					new WebServerGracefulShutdownLifecycle(this.webServer));
			getBeanFactory().registerSingleton("webServerStartStop",
					new WebServerStartStopLifecycle(this, this.webServer));
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
		initPropertySources();
	}
```
`ServletWebServerFactory`为自动装配进来，此时在`BeanFactory`中已经存在了。具体代码配置在spring-boot-autoconfigure包的META-INF/spring.factories中。
```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
......
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
```
`ServletWebServerFactoryAutoConfiguration`中import了一些类，其中impot了EmbeddedTomcat。
```
@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedTomcat {

		@Bean
		TomcatServletWebServerFactory tomcatServletWebServerFactory(
				ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
				ObjectProvider<TomcatContextCustomizer> contextCustomizers,
				ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
			TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
			factory.getTomcatConnectorCustomizers()
					.addAll(connectorCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getTomcatContextCustomizers()
					.addAll(contextCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getTomcatProtocolHandlerCustomizers()
					.addAll(protocolHandlerCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

	}
```
在这里生成了一个`TomcatServletWebServerFactory`的bean。
再次回到`createWebServer`中，`this.webServer = factory.getWebServer(getSelfInitializer());`从factory中生产出一个webserver。我们看这个webServer是如何生产出来的。
```
public WebServer getWebServer(ServletContextInitializer... initializers) {
		if (this.disableMBeanRegistry) {
			Registry.disableRegistry();
		}
		Tomcat tomcat = new Tomcat();
		File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
		tomcat.setBaseDir(baseDir.getAbsolutePath());
		Connector connector = new Connector(this.protocol);
		connector.setThrowOnFailure(true);
		tomcat.getService().addConnector(connector);
		customizeConnector(connector);
		tomcat.setConnector(connector);
		tomcat.getHost().setAutoDeploy(false);
		configureEngine(tomcat.getEngine());
		for (Connector additionalConnector : this.additionalTomcatConnectors) {
			tomcat.getService().addConnector(additionalConnector);
		}
		prepareContext(tomcat.getHost(), initializers);
		return getTomcatWebServer(tomcat);
	}
```
上面这段代码，就正式进入到了tomcat的领域了。为了能更好的阅读代码，我们需要了解tomcat的体系结构。这里通过代码的阅读和分析，总结了一张结构图。

![tomcat结构](/images/tomcat结构.png)

tomcat的初始化，无非就是将图中的各个部分初始化，塞到对应的位置。有了这张图，代码读起来就比较轻松了，代码也相对简单，可以自己按照图中的结构，进行走读。这里不再记录。
`getWebServer`代码走完，tomcat就初始化完成了，那么是不是就可以接收请求了呢。当然不行。socket还没起来，tomcat的工作线程还没起来。这部分功能不在上面的`onRefresh`中，而在接下来的`finishRefresh`中，只有所有的准备工作都已经完成了，请求才能接进来。这部分功能将在下一篇日记中走读。
至此，springboot的内嵌tomcat初始化完成。这里只记录了主流程的初始化，其中一些配置类信息如`ServerProperties`等的初始化并没有在这篇日记中体现。