---
layout:     post
title:      Spring MVC 学习笔记（一个请求的旅行过程）
subtitle:   Spring MVC 学习笔记（一个请求的旅行过程）
date:       2019-02-03
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- Spring MVC
- java
---

## 一个请求的旅行过程
Spring MVC 处理请求的流程图如下

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15300766829012.jpg)

 
如上，每一个重要的步骤上面都有编号。我先来简单分析一下上面的流程，然后再向大家介绍图中出现的一些组件。我们从第一步开始，首先，用户的浏览器发出了一个请求，这个请求经过互联网到达了我们的服务器。Servlet 容器首先接待了这个请求，并将该请求委托给 DispatcherServlet 进行处理。接着 DispatcherServlet 将该请求传给了处理器映射组件 HandlerMapping，并获取到适合该请求的拦截器和处理器。在获取到处理器后，DispatcherServlet 还不能直接调用处理器的逻辑，需要进行对处理器进行适配。处理器适配成功后，DispatcherServlet 通过处理器适配器 HandlerAdapter 调用处理器的逻辑，并获取返回值 ModelAndView。之后，DispatcherServlet 需要根据 ModelAndView 解析视图。解析视图的工作由 ViewResolver 完成，若能解析成功，ViewResolver 会返回相应的视图对象 View。在获取到具体的 View 对象后，最后一步要做的事情就是由 View 渲染视图，并将渲染结果返回给用户。
1. 接收请求
2. 映射处理器
3. 处理器适配
4. 调用处理器方法
5. 解析处理器方法返回的结果视图
6. 渲染视图
7. 响应请求

以上就是 Spring MVC 处理请求的全过程，上面的流程进行了一定的简化，比如拦截器的执行时机就没说。不过这并不影响大家对主过程的理解。下来来简单介绍一下图中出现的一些组件：

 组        件	  |  说明
----------------|------
DispatcherServlet	 |Spring MVC 的核心组件，是请求的入口，负责协调各个组件工作
HandlerMapping	  |内部维护了一些 <访问路径, 处理器> 映射，负责为请求找到合适的处理器
HandlerAdapter	  |处理器的适配器。Spring 中的处理器的实现多变，比如用户处理器可以实现 Controller 接口，也可以用 @RequestMapping 注解将方法作为一个处理器等，这就导致 Spring 不知道怎么调用用户的处理器逻辑。所以这里需要一个处理器适配器，由处理器适配器去调用处理器的逻辑
ViewResolver |视图解析器的用途不难理解，用于将视图名称解析为视图对象 View。
View |视图对象用于将模板渲染成 html 或其他类型的文件。比如 InternalResourceView 可将 jsp 渲染成 html。

从上面的流程中可以看出，Spring MVC 对各个组件的职责划分的比较清晰。DispatcherServlet 负责协调，其他组件则各自做分内之事，互不干扰。经过这样的职责划分，代码会便于维护。同时对于源码阅读者来说，也会很友好。可以降低理解源码的难度，使大家能够快速理清主逻辑。这一点值得我们学习。

## Servlet

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15300899315187.jpg)
   
#### Servlet 与 ServletConfig
先来看看 Servlet 接口的定义，如下：

```java
public interface Servlet {

    public void init(ServletConfig config) throws ServletException;

    public ServletConfig getServletConfig();
    
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;
   
    public String getServletInfo();
    
    public void destroy();
}
```

init 方法会在容器启动时由容器调用，也可能会在 Servlet 第一次被使用时调用，调用时机取决 load-on-start 的配置。容器调用 init 方法时，会向其传入一个 ServletConfig 参数。ServletConfig 是什么呢？顾名思义，ServletConfig 是一个和 Servlet 配置相关的接口。举个例子说明一下，我们在配置 Spring MVC 的 DispatcherServlet 时，会通过 ServletConfig 将配置文件的位置告知 DispatcherServlet。比如：

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:application-web.xml</param-value>
    </init-param>
</servlet>
```

如上，标签内的配置信息最终会被放入 ServletConfig 实现类对象中。DispatcherServlet 通过 ServletConfig 接口中的方法，就能获取到 contextConfigLocation 对应的值。

Servlet 中的 service 方法用于处理请求。当然，一般情况下我们不会直接实现 Servlet 接口，通常是通过继承 HttpServlet 抽象类编写业务逻辑的。Servlet 中接口不多，也不难理解，这里就不多说了。下面我们来看看 ServletConfig 接口定义，如下：

```java
public interface ServletConfig {
    
    public String getServletName();

    public ServletContext getServletContext();

    public String getInitParameter(String name);

    public Enumeration<String> getInitParameterNames();
}
```

先来看看 getServletName 方法，该方法用于获取 servlet 名称，也就是标签中配置的内容。getServletContext 方法用于获取 Servlet 上下文。如果说一个 ServletConfig 对应一个 Servlet，那么一个 ServletContext 则是对应所有的 Servlet。ServletContext 代表当前的 Web 应用，可用于记录一些全局变量，当然它的功能不局限于记录变量。我们可通过标签向 ServletContext 中配置信息，比如在配置 Spring 监听器（ContextLoaderListener）时，就可以通过该标签配置 contextConfigLocation。如下：

```java
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:application.xml</param-value>
</context-param>
```

关于 ServletContext 就先说这么多了，继续介绍 ServletConfig 中的其他方法。getInitParameter 方法用于获取标签中配置的参数值，getInitParameterNames 则是获取所有配置的名称集合，这两个方法用途都不难理解。

以上是 Servlet 与 ServletConfig 两个接口的说明，比较简单。说完这两个接口，我们继续往下看，接下来是 GenericServlet。

#### GenericServlet
GenericServlet 实现了 Servlet 和 ServletConfig 两个接口，为这两个接口中的部分方法提供了简单的实现。比如该类实现了 Servlet 接口中的 void init(ServletConfig) 方法，并在方法体内调用了内部提供了一个无参的 init 方法，子类可覆盖该无参 init 方法。除此之外，GenericServlet 还实现了 ServletConfig 接口中的 getInitParameter 方法，用户可直接调用该方法获取到配置信息。而不用先获取 ServletConfig，然后再调用 ServletConfig 的 getInitParameter 方法获取。下面我们来看看 GenericServlet 部分方法的源码：

```java
public abstract class GenericServlet 
    implements Servlet, ServletConfig, java.io.Serializable {

    // 省略部分代码

    private transient ServletConfig config;
    
    public GenericServlet() { } 
    
    /** 有参 init 方法 */
    public void init(ServletConfig config) throws ServletException {
        this.config = config;
        // 调用内部定义的无参 init 方法
        this.init();
    }

    /** 无参 init 方法，子类可覆盖该方法 */
    public void init() throws ServletException { }

    /** 未给 service 方法提供具体的实现 */
    public abstract void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;

    public void destroy() { }

    /** 通过 getInitParameter 可直接从 ServletConfig 实现类中获取配置信息 */
    public String getInitParameter(String name) {
        ServletConfig sc = getServletConfig();
        if (sc == null) {
            throw new IllegalStateException(
                lStrings.getString("err.servlet_config_not_initialized"));
        }

        return sc.getInitParameter(name);
    } 

    public ServletConfig getServletConfig() {
        return config;
    }
    
    // 省略部分代码
}
```

GenericServlet 是一个协议无关的 servlet，是一个比较原始的实现，通常我们不会直接继承该类。一般情况下，我们都是继承 GenericServlet 的子类 HttpServlet，该类是一个和 HTTP 协议相关的 Servlet。那下面我们来看一下这个类。

## HttpServlet
HttpServlet，从名字上就可看出，这个类是和 HTTP 协议相关。该类的关注点在于怎么处理 HTTP 请求，比如其定义了 doGet 方法处理 GET 类型的请求，定义了 doPost 方法处理 POST 类型的请求等。我们若需要基于 Servlet 写 Web 应用，应继承该类，并覆盖指定的方法。doGet 和 doPost 等方法并不是处理的入口方法，所以这些方法需要由其他方法调用才行。其他方法是哪个方法呢？当然是 service 方法了。下面我们看一下这个方法的实现。如下：

```java
@Override
public void service(ServletRequest req, ServletResponse res)
    throws ServletException, IOException {
    HttpServletRequest  request;
    HttpServletResponse response;
    
    if (!(req instanceof HttpServletRequest &&
            res instanceof HttpServletResponse)) {
        throw new ServletException("non-HTTP request or response");
    }

    request = (HttpServletRequest) req;
    response = (HttpServletResponse) res;

    // 调用重载方法，该重载方法接受 HttpServletRequest 和 HttpServletResponse 类型的参数
    service(request, response);
}

protected void service(HttpServletRequest req, HttpServletResponse resp)
    throws ServletException, IOException {
    String method = req.getMethod();

    // 处理 GET 请求
    if (method.equals(METHOD_GET)) {
        long lastModified = getLastModified(req);
        if (lastModified == -1) {
            // 调用 doGet 方法
            doGet(req, resp);
        } else {
            long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
            if (ifModifiedSince < lastModified) {
                maybeSetLastModified(resp, lastModified);
                doGet(req, resp);
            } else {
                resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            }
        }

    // 处理 HEAD 请求
    } else if (method.equals(METHOD_HEAD)) {
        long lastModified = getLastModified(req);
        maybeSetLastModified(resp, lastModified);
        doHead(req, resp);

    // 处理 POST 请求
    } else if (method.equals(METHOD_POST)) {
        // 调用 doPost 方法
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
        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[1];
        errArgs[0] = method;
        errMsg = MessageFormat.format(errMsg, errArgs);
        
        resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
    }
}
```

## DispatcherServlet族谱
我在前面说到，DispatcherServlet 是 Spring MVC 的核心。所以在分析这个类的源码前，我们有必要了解一下它的族谱，也就是继承关系图。如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15302319955704.jpg)

如上图，红色框是 Servlet 中的接口和类，蓝色框中则是 Spring 中的接口和类。关于 Servlet 内容前面已经说过，下面来简单介绍一下蓝色框中的接口和类，我们从最顶层的接口开始。

1.Aware
    在 Spring 中，Aware 类型的接口用于向 Spring “索要”一些框架中的信息。比如当某个 bean 实现了 ApplicationContextAware 接口时，Spring 在运行时会将当前的 ApplicationContext 实例通过接口方法 setApplicationContext 传给该 bean。下面举个例子说明，这里我写一个 SystemInfo API，通过该 API 返回一些系统信息。代码如下：

```java
@RestController
@RequestMapping("/systeminfo")
public class SystemInfo implements ApplicationContextAware, EnvironmentAware {

    private ApplicationContext applicationContext;

    private Environment environment;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println(applicationContext.getClass());
        this.applicationContext = applicationContext;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @RequestMapping("/env")
    public String environment() {
        StandardServletEnvironment sse = (StandardServletEnvironment) environment;
        Map<String, Object> envs = sse.getSystemEnvironment();
        StringBuilder sb = new StringBuilder();
        sb.append("-------------------------++ System Environment ++-------------------------\n");

        List<String> list = new ArrayList<>();
        list.addAll(envs.keySet());

        for (int i = 0; i < 5 && i < list.size(); i++) {
            String key = list.get(i);
            Object val = envs.get(key);
            sb.append(String.format("%s = %s\n", key, val.toString()));
        }

        Map<String, Object> props = sse.getSystemProperties();
        sb.append("\n-------------------------++ System Properties ++-------------------------\n");
        list.clear();
        list.addAll(props.keySet());
        for (int i = 0; i < 5 && i < list.size(); i++) {
            String key = list.get(i);
            Object val = props.get(key);
            sb.append(String.format("%s = %s\n", key, val.toString()));
        }

        return sb.toString();
    }

    @RequestMapping("/beans")
    public String listBeans() {
        ListableBeanFactory lbf = applicationContext;
        String[] beanNames = lbf.getBeanDefinitionNames();
        StringBuilder sb = new StringBuilder();
        sb.append("-------------------------++ Bean Info ++-------------------------\n");
        Arrays.stream(beanNames).forEach(beanName -> {
            Object bean = lbf.getBean(beanName);
            sb.append(String.format("beanName  = %s\n", beanName));
            sb.append(String.format("beanClass = %s\n\n", bean.getClass().toString()));
        });

        return sb.toString();
    }
}
```

如上，SystemInfo 分别实现了 ApplicationContextAware 和 EnvironmentAware 接口，因此它可以在运行时获取到 ApplicationContext 和 Environment 实例。下面我们调一下接口看看结果吧：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15302443594239.jpg)

如上，我们通过接口拿到了环境变量、配置信息以及容器中所有 bean 的数据。这说明，Spring 在运行时向 SystemInfo 中注入了 ApplicationContext 和 Environment 实例。

2.EnvironmentCapable
    EnvironmentCapable 仅包含一个方法定义 getEnvironment，通过该方法可以获取到环境变量对象。我们可以将 EnvironmentCapable 和 EnvironmentAware 接口配合使用，比如下面的实例：

```java
public class EnvironmentHolder implements EnvironmentCapable, EnvironmentAware {

    private Environment environment;

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public Environment getEnvironment() {
        return environment;
    }
}
```

3.HttpServletBean
    HttpServletBean 是 HttpServlet 抽象类的简单拓展。HttpServletBean 覆写了父类中的无参 init 方法，并在该方法中将 ServletConfig 里的配置信息设置到子类对象中，比如 DispatcherServlet。

4.FrameworkServlet
    FrameworkServlet 是 Spring Web 框架中的一个基础类，该类会在初始化时创建一个容器。同时该类覆写了 doGet、doPost 等方法，并将所有类型的请求委托给 doService 方法去处理。doService 是一个抽象方法，需要子类实现。

5.DispatcherServlet
    DispatcherServlet 主要的职责相信大家都比较清楚了，即协调各个组件工作。除此之外，DispatcherServlet 还有一个重要的事情要做，即初始化各种组件，比如 HandlerMapping、HandlerAdapter 等。

## DispatcherServlet 源码简析

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

            // 获取可处理当前请求的处理器 Handler，对应流程图中的步骤②
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null || mappedHandler.getHandler() == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            // 获取可执行处理器逻辑的适配器 HandlerAdapter，对应步骤③
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // 处理 last-modified 消息头
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

            // 执行拦截器 preHandle 方法
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // 调用处理器逻辑，对应步骤④
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            // 如果 controller 未返回 view 名称，这里生成默认的 view 名称
            applyDefaultViewName(processedRequest, mv);

            // 执行拦截器 preHandle 方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        
        // 解析并渲染视图
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
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}

private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {

    boolean errorView = false;

    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }

    if (mv != null && !mv.wasCleared()) {
        // 渲染视图
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isDebugEnabled()) {...
    }

    if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        return;
    }

    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}

protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
    Locale locale = this.localeResolver.resolveLocale(request);
    response.setLocale(locale);

    View view;
    /*
     * 若 mv 中的 view 是 String 类型，即处理器返回的是模板名称，
     * 这里将其解析为具体的 View 对象
     */ 
    if (mv.isReference()) {
        // 解析视图，对应步骤⑤
        view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
        if (view == null) {
            throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
                    "' in servlet with name '" + getServletName() + "'");
        }
    }
    else {
        view = mv.getView();
        if (view == null) {
            throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
                    "View object in servlet with name '" + getServletName() + "'");
        }
    }

    if (logger.isDebugEnabled()) {...}
    try {
        if (mv.getStatus() != null) {
            response.setStatus(mv.getStatus().value());
        }
        // 渲染视图，并将结果返回给用户。对应步骤⑥和⑦
        view.render(mv.getModelInternal(), request, response);
    }
    catch (Exception ex) {
        if (logger.isDebugEnabled()) {...}
        throw ex;
    }
}
```



