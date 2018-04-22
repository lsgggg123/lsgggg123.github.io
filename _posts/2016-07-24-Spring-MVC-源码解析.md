---
layout: post
title: 'Spring MVC 源码解析'
subtitle: '记录了自己调试SpringMVC源码的过程'
date: 2016-07-24
categories: 技术
tags: 技术 java Spring SpringMVC
---

Spring MVC是SpringFrameWork提供的构建Web应用程序的全功能MVC模块，是一个Web MVC框架。由于Spring MVC出色的运行效率、优秀的设计、和Spring的无缝对接等等原因使得它打败了其他同类型竞争者（典型代表例如Strust2），成为当前web开发中应用最多的框架。今天我们来从Spring MVC框架的源码来深度分析这个框架的运行原理。

### Spring MVC的架构图
![image](/assets/img/20180422/e1b3f9748e1b40ff9edbd2b2d5142d54.jpg)

### 入口配置
我们知道，集成Spring MVC第一步要在项目的webapp/WEB-INF目录下的web.xml里面配置一个Spring MVC的前端控制器DispatcherServlet，DispatcherServlet控制着http请求是否交给Spring MVC来处理，因此DispatcherServlet作为Spring MVC的入口，是整个Spring MVC的核心，我们的源码分析也是从DispatcherServlet开始的。下面是一段典型的Spring MVC配置文件：


```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>classpath:spring/spring.xml</param-value>
</context-param>
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<servlet>
	<servlet-name>SpringMVC</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<init-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath:spring/springmvc.xml</param-value>
	</init-param>
	<load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
	<servlet-name>SpringMVC</servlet-name>
	<url-pattern>/</url-pattern>
</servlet-mapping>
```

通过这个配置文件，我们可以得知DispatcherServlet的本质是一个Servlet。我们查看它的继承树可以查到它继承自HttpServlet，HttpServlet又实现了接口Servlet，所以说DispatcherServlet是一个Servlet。

![image](/assets/img/20180422/5dadce8c3af24cdd80aa57529c673692.png)

提到Servlet，大家一定都不陌生，我们的Java Web学习之旅就是从Jsp/Servlet开始的。Spring MVC虽然复杂，但是核心的DispacherServlet却是一个Servlet，一定遵从Servlet生命周期的三个阶段，就是所谓的“init-service-destroy”。这里要多说一句，一般的Servlet的生命周期，init初始化是在这个Servlet第一次访问的时候做的，只初始化一次，持续提供服务，直到Web容器停止才会调用destroy销毁方法，而DispatcherServlet因为初始化的时候做的操作太多，一般来说会配置一个<load-on-startup>1</load-on-startup>，这样容器启动的时候就可以初始化这个Servlet（调用Servlet的init()方法）。下面我们就看一下DispacherServlet是怎么进行初始化的。

### DispatcherServlet的初始化
DispatcherServlet的init方法实现在父类HttpServletBean中，代码如下（省略了部分的log和注释，下同）。我们加一个断点，然后启动容器来看一下代码的执行情况。


```java
@Override
public final void init() throws ServletException {
	try {
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
		ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
		bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
		initBeanWrapper(bw);
		bw.setPropertyValues(pvs, true);
	} catch (BeansException ex) {
		logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
		throw ex;
	}

	initServletBean();
}
```

主要的代码分成两个部分，一个在try-catch中，一个在initServletBean()中，这两部分的代码由HttpServletBean的子类，DispatcherServlet的父类FrameworkServlet实现。

#### a) try-catch快：
这段的代码主要从web.xml的init-param节点读取DispatcherServlet的配置，比如读取了Spring MVC配置文件的位置。然后把DispatcherServlet封装成BeanWrapper，BeanWrapper的主要功能，就是对任何一个bean，进行属性的设置和方法的调用。最后通过BeanWrapper对DispatcherServlet做一些初始化工作，把init-param节点的配置设置到DispatcherServlet成员属性里面，具体来说，就是Spring MVC配置文件的路径了。

![image](/assets/img/20180422/a4bf3f520c6b49b3ae9dea44444cfe46.png)

#### b）initServletBean()：

非常重要，这段代码的主要作用是容器上下文的建立。这段代码在FrameworkServlet里面：


```java
@Override
protected final void initServletBean() throws ServletException {
	long startTime = System.currentTimeMillis();

	try {
		this.webApplicationContext = initWebApplicationContext();
		initFrameworkServlet();
	} catch (ServletException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	} catch (RuntimeException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	}
}
```

看方法的名字我们也能猜出大概，这个方法是用来再次初始化一个WebApplicationContext，WebApplicationContext是ApplicationContext的子类，我们都知道，ApplicationContext是Spring的核心，称为应用上下文，也就是传说中的Spring容器，专门管理bean的。这里为什么要说再次初始化一个？因为Web项目启动的时候，web.xml里面配置的ContextLoaderListener已经初始化了一个ApplicationContext，这个ApplicationContext叫做Root ApplicationContext，它管理着你在web.xml里面的context-param配置的Spring配置文件里面的所有bean。Root ApplicationContext创建之后，会被放入ServletContext的attribute里面供取用。显然，initWebApplicationContext()这里面还要初始化一个WebApplicationContext，以Root ApplicationContext作为它的父上下文，并初始化Servlet的init-param里面配置的Spring配置文件里面的bean。

这样我们有两个ApplicationContext，一个管理着context-param配置的Spring配置文件里面的bean，另一个管理着init-param配置的Spring配置文件里面的bean。这两个的实现都是XmlWebApplicationContext，那么有区别么？我个人认为使用起来没有区别，DispatcherServlet的初始化的WebApplicationContext以ContextLoaderListener初始化的Root ApplicationContext为父上下文。对于作用范围而言，在DispatcherServlet中可以引用由ContextLoaderListener所创建的ApplicationContext中的Bean，而反过来不行。有的人喜欢在ContextLoaderListener里面就把所有的bean都配置完了，也有人相反，不管如何都不影响使用。我个人倾向于把Service，Dao，Datasource之类的配置在context-param里面的配置文件，而把一些web相关的bean如Controller，HandlerMapping，Interceptor，ViewResolver配置在init-param里面的配置文件。

initWebApplicationContext()方法内的逻辑如下：


```java
protected WebApplicationContext initWebApplicationContext() {
    // 1.
	WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;

    // 2.
	if (this.webApplicationContext != null) {
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				if (cwac.getParent() == null) {
					cwac.setParent(rootContext);
				}
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	
	// 3.
	if (wac == null) {
		wac = findWebApplicationContext();
	}
	
	// 4.
	if (wac == null) {
		wac = createWebApplicationContext(rootContext);
	}
	
	// 6.
	if (!this.refreshEventReceived) {
		onRefresh(wac);
	}
	
	// 7.
	if (this.publishContext) {
		String attrName = getServletContextAttributeName();
		getServletContext().setAttribute(attrName, wac);
	}
	return wac;
}
```

1. 获取Root ApplicationContext，由前文可知，直接去ServletContext可以取到。

2. 如果FrameworkServlet的webApplicationContext属性已经不为空，说明在FrameworkServlet对象创建的时候通过构造器传入了一个webApplicationContext，那就直接使用这个webApplicationContext，若这个webApplicationContext还没被初始化，将Root ApplicationContext设置为它的父上下文，然后将其初始化。

3. 通过wac变量的引用是否为null，判断2. 中是否已经完成上下文的设置，如果wac==null成立，2. 没有执行，这时去servlet context去找找看有没有注册过的。

4. 再次检查wac变量的引用是否为null，如果wac==null成立，说明2. 3. 都没有执行，此时调用createWebApplicationContext(rootContext)方法新建一个webApplicationContext（以Root ApplicationContext为父，Spring MVC的配置文件路径等也会传进去），来管理Spring MVC相关的beans。绝大多数情况下都会新建这个webApplicationContext。

5. 创建的过程：createWebApplicationContext(rootContext) -> configureAndRefreshWebApplicationContext(wac) -> wac.refresh() 这里面有很多非常多的代码，其中就有RequestMappingHandlerMapping，可以在RequestMappingHandlerMapping的afterPropertiesSet()加个断点观察这一过程。 

6. 三种创建webApplicationContext，都会调用onRefresh(wac)，onRefresh方法在DispatcherServlet中，如下：


```java
@Override
protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
	initMultipartResolver(context);
	initLocaleResolver(context);
	initThemeResolver(context);
	initHandlerMappings(context);
	initHandlerAdapters(context);
	initHandlerExceptionResolvers(context);
	initRequestToViewNameTranslator(context);
	initViewResolvers(context);
	initFlashMapManager(context);
}
```

这段代码会把refresh()中收集到的所有的Handlers/Controllers/View-resolvers之类的beans设置到DispatcherServlet的成员属性里面，方便后续的调用。

我们来看最重要的initHandlerMappings()，无非是通过webApplicationContext.getBean()之类的方法来获取HandlerMapping.class类型的bean，设置到自己的一个成员属性为handlerMappings的List里面。


```java
private void initHandlerMappings(ApplicationContext context) {
	this.handlerMappings = null;

	Map<String, HandlerMapping> matchingBeans =
			BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
	if (!matchingBeans.isEmpty()) {
		this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
		AnnotationAwareOrderComparator.sort(this.handlerMappings);
	}
}
```

最后将webApplicationContext设置到ServletContext里面的attribute里面（步骤7. ），name为org.springframework.web.servlet.FrameworkServlet.CONTEXT + web.xml里面配置的Servlet Name，value为这个webApplicationContext


至此，整个DispatcherServlet完成了初始化，总结起来，创建了一个webApplicationContext，它管理着所有的Spring MVC配置文件配置的bean，收集了所有的handlerMapping，以便后续做url和Controller的映射。


### DispatcherServlet的请求转发

根据我们的Servlet相关的知识，当浏览器发起了一次http请求，如果满足url-mapping里面配置的路径，就会进入到相关Servlet的service方法。这个方法在DispatcherServlet的父类FrameworkServlet里面，FrameworkServlet的service()。service()调用了processRequest()主要做了将当前请求的Locale对象和属性，分别设置到LocaleContextHolder和RequestContextHolder这两个抽象类中的ThreadLocal对象中，也就是分别将这两个东西和请求线程做了绑定，调用doService(request, response)后，发出一个RequestHandledEventd的事件。doService()在DispatcherServlet里面实现，主要把DispatcherServlet当前的一些成员属性例如webApplicationContext等等设置到request里面去，这样就可以在任意处理request的地方使用。然后调用doDispatch(request, response)，doDispatch()这个方法做的就是把url的映射到具体的Controller。我们看下主要的源码：


```java
try {
	processedRequest = checkMultipart(request);
	multipartRequestParsed = (processedRequest != request);

	// Determine handler for the current request.
	mappedHandler = getHandler(processedRequest);
	if (mappedHandler == null || mappedHandler.getHandler() == null) {
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
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

```

#### getHandler(processedRequest)

这句决定当前的请求交给哪个HandlerExecutionChain处理，如果你记性够好的话，还能记得前面的DispatcherServlet初始化的时候已经收集了所有的HandlerMapping放在了DispatcherServlet的一个叫handlerMappings的List里面。getHandler()的实现我想你也能猜出来：遍历这个handlerMappings，通过url找到和哪个注册的HandlerMapping相匹配，然后构造一个HandlerExecutionChain出来，把收集到的HandlerInterceptor加到HandlerExecutionChain里面。之后的请求，就全部交给这个HandlerExecutionChain了。这个在b）里面解析。另外，多说一句，如果访问一个不存在的url或者是静态资源文件，getHandler(processedRequest)都会为null，由response.sendError(404)直接到404页面！！！url我们还可以理解，毕竟没有找到HandlerMapping，但是Spring MVC显然把静态资源文件也当成url去进行映射了，我们不能坐视不理。

我们通常会配置一个<mvc:default-servlet-handler />，专门用来解决资源文件的访问问题。如果配置了default-servlet-handler，不存在的url或者是静态资源文件，都会得到一个DefaultServletHttpRequestHandler，资源可以可以返回具体的资源文件，不存在的url则会返回404页面。

#### HandlerExecutionChain处理过程
getHandlerAdapter(), 拿到的是一个HandlerAdapter，Spring MVC通过HandlerAdapter来实际调用处理函数，就是确定调用哪个类的哪个方法，并且构造方法参数，获得返回值。

HandlerAdapter是哪里来的？如果配置文件里面加了<mvc:annotation-driven />，Spring MVC默认会初始化RequestMappingHandlerAdapter（3.1之前是AnnotationMethodHandlerAdapter），当然初始化的bean不止这一个，感兴趣的参考下文末的链接8。

可以看到很明显的，HandlerExecutionChain先处理interceptors，HandlerAdapter处理Controller方法获得一个ModelAndView，然后HandlerExecutionChain再处理interceptors

我们把最关键的代码拿来，再跟着看下最上方的架构图，是不是一切都清楚了？


```java
if (!mappedHandler.applyPreHandle(processedRequest, response)) {

    return;

}

mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

mappedHandler.applyPostHandle(processedRequest, response, mv);
```

观察ha.handle()的方法参数，可以看到有request，有response，有handler，request可以拿到所有的请求数据，handler可以拿到Controller的具体方法信息，那么调用这个Controller就没有任何困难了，我们还是来跟一下具体的代码吧，经过一些方法调用，ha.handle(...) -> RequestMappingHandlerAdapter.handleInternal(...) -> invokeHandlerMethod()， 我们最终锁定在invocableMethod.invokeAndHandle(webRequest, mavContainer)和getModelAndView(mavContainer, modelFactory, webRequest)这里：


```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	//……代码有省略

	invocableMethod.invokeAndHandle(webRequest, mavContainer);

	return getModelAndView(mavContainer, modelFactory, webRequest);
}
```

invokeAndHandle这个方法的参数webRequest封装了request和response，mavContainer专门用来存放Controller执行之后的Model，还有一些跟View有关，例如Controller返回的view名字。getModelAndView这个方法拿到mavContainer在invokeAndHandle中放置的信息，我们分别跟进这两个方法：


```java
public void invokeAndHandle(ServletWebRequest webRequest,
		ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	setResponseStatus(webRequest);

	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	else if (StringUtils.hasText(this.responseReason)) {
		mavContainer.setRequestHandled(true);
		return;
	}

	mavContainer.setRequestHandled(false);
	try {
		this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	} catch (Exception ex) {
		throw ex;
	}
}
```

invokeForRequest(webRequest, mavContainer, providedArgs)

会通过反射来调用Controller的具体方法，并且拿到Controller方法的返回值。Controller方法中放入Model的值也会收集到mavContainer中，例如下面的Controller就放了一个名字为article对象到Model里面，按照上文说的，这个对象会到mavContainer里面。


```java
@RequestMapping(value="/article/{id}")
public String article(Model model, @PathVariable("id") Integer id) {
    ServiceResult result = articleService.getArticle(id);
    model.addAttribute("article", result.get("article"));
    return "article";
}
```

this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);


```java
public void handleReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    HandlerMethodReturnValueHandler handler = this.selectHandler(returnValue, returnType);
    Assert.notNull(handler, "Unknown return value type [" + returnType.getParameterType().getName() + "]");
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

这一句决定Controller方法调用之后的返回值该如何处理，我拿两个最常用的Controller调用试了一下，一个是返回一个视图的，另一个是@ResponseBody的，前者会把返回的视图名称放进mavContainer里面，后者会直接标记这个请求已经处理了，并且会在进行一些消息转换，这部分将来再写文章讨论，这里就不多说了。

总之，invokeAndHandle(...)，调用了Controller的具体方法，拿到了返回的视图和要放入视图中使用的对象信息，getModelAndView(...) 最终生成一个ModelAndView对象出来。这样，一个Http的请求，是如何被Spring MVC转发的，我想我已经弄明白了（至于你们读者明白不明白，我对自己的语文水平表示堪忧）。

### 渲染ModelAndView

剩下的这部分很简单了，这部分工作在下面的代码中（DispatcherServlet的doDispatch()，上文贴过的代码最多的那堆的最后一行）。


```java
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

跟进去然后来到

```java
DispatcherServlet.render(mv, request, response);
```

里面的逻辑很简单，主要是把ModelAndView里面的一些准备放在视图中使用的对象值，set到request的attribute里面，就不带大家看了。

### 后记
这篇文章本来想写了能够把Spring MVC的一些原理讲清楚，等写完了，我觉得并没有达到这个效果，一方面，我的水平还不够站在一个很高的角度去理解Spring MVC部分设计的原因，而且我也不认为有谁能看了哪一篇文章就掌握了Spring MVC的原理，最主要的还是要靠自己去调试，自己去理解。所以本文就姑且留着给我自己做备忘好了。

### 参考文章：
1. [load-on-startup](http://www.blogjava.net/xzclog/archive/2011/09/29/359789.html)

2. [BeanWrapper](http://my.oschina.net/u/2000201/blog/489752)

3. [spring启动t](http://www.cnblogs.com/davidwang456/archive/2013/03/12/2956125.html)

4. [FrameworkServle](http://www.cnblogs.com/davidwang456/p/4122842.html)

5. [Spring MVC源码剖析](http://my.oschina.net/lichhao/blog/100138)

6. [Difference between applicationcontext and webapplicationcontext](http://stackoverflow.com/questions/11708967/what-is-the-difference-between-applicationcontext-and-webapplicationcontext-in-s)

7. [DefaultServletHttpRequestHandler](http://perfy315.iteye.com/blog/2008763)

8. [mvc:annotation-driven](http://my.oschina.net/HeliosFly/blog/205343)

