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
---
上篇文章走读了springboot中消息处理进入到servlet里了，这次，具体走读了请求消息在servlet里是怎么处理的。这里主要补充servlet和springmvc相关的知识。
<!--more-->
### servlet相关知识

#### servlet生命周期

servlet生命周期在servlet的代码里能很清楚的体现出来。代码为：

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
从代码中可以看出，servlet生命周期是init-->service-->destory;
#### 具体过程
> 1、加载与实例化（new）
     servlet在启动时或第一次接收到请求时，会到内存中去查询一下是否有servlet实例，有则取出来，没有则new一个出来。
> 2、初始化（init）
    在创建servlet之后，会调用init方法，进行初始化，该步主要是为接收处理请求做一些必要的准备，只会执行一次。
>3、提供服务（service）
    servlet实例接收客户端的ServletRequest信息，通过request的信息，调用相应的doXXX()方法，并返回response。
>4、销毁（destroy）
    servlet容器在关闭时，销毁servlet实例。只会执行一次

后面以SpringMVC为例来具体说明servlet的生命周期。

### SpringMVC相关知识

这里只简单的罗列一下SpringMVC中M，V，C 的关系。如图：
![SpringMVC](/images/springMVC框架图.png)
>1、客户端发送request请求进入到分发器中，DispatcherServlet中。
>2、分发器通过uri到控制器映射中去查询相应的处理器（HandlerMapping）。
>3、分发器拿到对应的处理器之后，调用处理器响应的接口，返回数据及对应的视图。
>4、分发器拿到处理器返回的modelAndView后，查找相应的视同解析器进行渲染。
>5、视图将渲染好的结果返回给终端用户进行展示。

下面通过源码的方式看springboot中关于springMVC和servlet相关的实现。

### 请求消息在SpringMVC中的流向

源码阅读接上篇文章，现在到了servlet的处理了。在这之前，看servlet在springboot中是如何完成实例化及初始化的。