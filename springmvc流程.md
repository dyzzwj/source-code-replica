



#一、DispatcherServlet的初始化



默认情况下 **在Tomcat容器启动时，SpringMVC（DispatcherServlet）并没有加载，而是第一次请求到来时才进行加载。**

这是由于DispatcherServlet的load-on-startup默认为-1

load-on-startup 这个元素的含义是在服务器启动的时候就加载这个servlet(实例化并调用init()方法).这个元素中的可选内容必须为一个整数,表明了这个servlet被加载的先后顺序.**当是一个负数时或者没有指定时，则表示服务器在该servlet被调用时才加载**。当值为0或者大于0时，表示服务器在启动时就加载这个servlet.该容器肯定可以保证被标记为更小的整数的servlet比被标记为更大的整数的servlet更先被调用

**如果需要启动时就加载DispatcherServlet  spring.mvc.servlet.load-on-startup=1即可**





DispatcherServlet本质上是一个HttpServlet，这就是为什么需要在web.xml通过url-mapping为DispatcherServlet配置映射请求的原因，DispatcherServlet具有Servlet的生命周期 init()；service()；destroy()

```java
DispatcherServlet extends FrameworkServlet extends HttpServletBean extends HttpServlet
```

我们从HttpServletBean开始看，HttpServletBean重写了其父类GenericServlet的init方法，我们来看看init到底做了什么（在启动Tomcat的时候会进入init方法）

##1、HttpServletBean.init()：初始化web.xml参数

```java
public final void init() throws ServletException {
   if (logger.isDebugEnabled()) {
      logger.debug("Initializing servlet '" + getServletName() + "'");
   }

   // Set bean properties from init parameters.
   try {
     	//ServletConfigPropertyValues负责取到web.xml中contextConfigLocation，并addPropertyValue()；
      PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
     	//BeanWrapper是一个实体包装类，简单地说，BeanWrapper提供分析和操作JavaBean的方案，如值的set/get方法、描述的set/get方法以及属性的可读可写性
      BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
     	//ResourceLoader读取到servletContext和classLoader，servletContext装载了我们刚才的dispatcher-servlet.xml,classLoader找到我们的字节码文件并追踪到我们的jar包路径
      ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
      bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
      initBeanWrapper(bw);
     	//这里构造BeanWrapper，使用setPropertyValues设置PropertyValues，利用Spring依赖注入的特性初始化属性，读取web.xml的contextConfigLocation属性用于构造Spring上下文
      bw.setPropertyValues(pvs, true);
     	//用BeanWrapper的最大好处在于，我们不需要再在HttpServletBean中定义contextConfigLocation属性，并声明调用set/get方法，BeanWrapper已经帮我们做好了
   }
   catch (BeansException ex) {
      if (logger.isErrorEnabled()) {
         logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
      }
      throw ex;
   }

   //留给子类实现
   initServletBean();

   if (logger.isDebugEnabled()) {
      logger.debug("Servlet '" + getServletName() + "' configured successfully");
   }
}
```

web.xml

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springConfig/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

##2、FrameworkServlet.initServletBean()：将上下文赋予当前Servlet

我们看FrameworkServlet这个类，该类继承自HttpServletBean,看FrameworkServlet的initServletBean()方法，



```java
protected final void initServletBean() throws ServletException {
   getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
   if (this.logger.isInfoEnabled()) {
      this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
   }
   long startTime = System.currentTimeMillis();

   try {
     	//webApplicationContext是FrameworkServlet的上下文，initWebApplicationContext()方法为当前Servlet初始化上下文

      this.webApplicationContext = initWebApplicationContext();
     	//initFrameworkServlet()交由FrameworkServlet子类实现，默认实现为空，该方法会在bean属性和上下文加载后被调用
      initFrameworkServlet();
   }
   catch (ServletException ex) {
      this.logger.error("Context initialization failed", ex);
      throw ex;
   }
   catch (RuntimeException ex) {
      this.logger.error("Context initialization failed", ex);
      throw ex;
   }

   if (this.logger.isInfoEnabled()) {
      long elapsedTime = System.currentTimeMillis() - startTime;
      this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
            elapsedTime + " ms");
   }
}
```

FrameworkServlet.initWebApplicationContext()：

```java
protected WebApplicationContext initWebApplicationContext() {
   //获取跟上下文
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   //初始化一个空的上下文
   WebApplicationContext wac = null;

   if (this.webApplicationContext != null) {
			//只有上下文实例在构造的时候注入才会进入本分支
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
   if (wac == null) {
  		//用来查看该Servlet是否已经设置上下文，返回null
      wac = findWebApplicationContext();
   }
   if (wac == null) {
      //参数是我们在initWebApplicationContext()中得到的rootContext(根上下文)，为FrameworkServlet初始化上下文,设置id,environment,configLocation等属性
      wac = createWebApplicationContext(rootContext);
   }

   if (!this.refreshEventReceived) {//refreshEventReceived = false
    	//是为了防止构造注入上下文的时候没有刷新，去手动刷新，在DispatcherServlet有实现
      onRefresh(wac);
   }

   if (this.publishContext) {
      // Publish the context as a servlet context attribute.
      String attrName = getServletContextAttributeName();
     	//为当前Servlet设置上下文
      getServletContext().setAttribute(attrName, wac);
      if (this.logger.isDebugEnabled()) {
         this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
               "' as ServletContext attribute with name [" + attrName + "]");
      }
   }

   return wac;
}
```

##3、DispatcherServlet.onRefresh()：

```java
protected void onRefresh(ApplicationContext context) {
   initStrategies(context);
}
```

DispatcherServlet.initStrategies()：

```java
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



#二、DispatcherServlet处理请求



##springmvc执行请求流程：

![springmvc执行流程](images/springmvc执行流程.png)

 1、用户发送请求至前端控制器DispatcherServlet
 2、DispatcherServlet对请求URL进行解析，得到请求资源标识符（URI）。然后根据该URI调用处理器映射器HandlerMapping。
 3、处理器映射器根据uri找到具体的处理器，生成处理器执行链HandlerExecutionChain(包括处理器对象和处理器拦截器)，并返回给DispatcherServlet。
 4、DispatcherServlet根据处理器Handler调用处理器适配器HandlerAdapter执行一系列的操作，如：参数封装，数据格式转换，数据验证等操作

- ​	HttpMessageConveter： 将请求消息（如Json、xml等数据）转换成一个对象，将对象转换为指定的响应信息
-    数据转换：对请求消息进行数据转换。如String转换成Integer、Double等
-    数据格式化：对请求消息进行数据格式化。 如将字符串转换成格式化数字或格式化日期等
-    数据验证： 验证数据的有效性（长度、格式等），验证结果存储到BindingResult或Error中

 5、HandlerAdapter调用具体的处理器Handler(Controller，也叫页面控制器)。
 6、Handler执行完成返回ModelAndView
 7、HandlerAdapter将Handler执行结果ModelAndView返回到DispatcherServlet
 8、DispatcherServlet将ModelAndView传给ViewReslover视图解析器
 9、ViewReslover解析后返回具体View
 10、DispatcherServlet对View进行渲染视图（即将模型数据model填充至视图中）。
 11、DispatcherServlet响应用户。

##组件说明：

1.DispatcherServlet：前端控制器。用户请求到达前端控制器，它就相当于mvc模式中的c，dispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，dispatcherServlet的存在降低了组件之间的耦合性,系统扩展性提高。由框架实现
 2.HandlerMapping：处理器映射器。HandlerMapping负责根据用户请求的url找到Handler即处理器，springmvc提供了不同的映射器实现不同的映射方式，根据一定的规则去查找,例如：xml配置方式，实现接口方式，注解方式等。由框架实现
 3.Handler：处理器。Handler 是继DispatcherServlet前端控制器的后端控制器，在DispatcherServlet的控制下Handler对具体的用户请求进行处理。由于Handler涉及到具体的用户业务请求，所以一般情况需要程序员根据业务需求开发Handler。
 4.HandlAdapter：处理器适配器。通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。由框架实现。
 5.ModelAndView是springmvc的封装对象，将model和view封装在一起。
 6.ViewResolver：视图解析器。ViewResolver负责将处理结果生成View视图，ViewResolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成View视图对象，最后对View进行渲染将处理结果通过页面展示给用户。
 7View:是springmvc的封装对象，是一个接口, springmvc框架提供了很多的View视图类型，包括：jspview，pdfview,jstlView、freemarkerView、pdfView等。一般情况下需要通过页面标签或页面模版技术将模型数据通过页面展示给用户，需要由程序员根据业务需求开发具体的页面。



FrameworkServlet.processRequest()：

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {

   long startTime = System.currentTimeMillis();
   Throwable failureCause = null;
		//previousLocaleContext获取和当前线程相关的LocaleContext，根据已有请求构造一个新的和当前线程相关的LocaleContext
   LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
   LocaleContext localeContext = buildLocaleContext(request);
		//previousAttributes获取和当前线程绑定的RequestAttributes,为已有请求构造新的ServletRequestAttributes，加入预绑定属性
   RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
   ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);
	//异步请求处理，没有用上
   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
   asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
		//initContextHolders让新构造的RequestAttributes和ServletRequestAttributes和当前线程绑定，加入到ThreadLocal，完成绑定
   initContextHolders(request, localeContext, requestAttributes);

   try {
     	//抽象方法doService由FrameworkServlet子类DispatcherServlet重写
      doService(request, response);
   }
   catch (ServletException ex) {
      failureCause = ex;
      throw ex;
   }
   catch (IOException ex) {
      failureCause = ex;
      throw ex;
   }
   catch (Throwable ex) {
      failureCause = ex;
      throw new NestedServletException("Request processing failed", ex);
   }

   finally {
     	//resetContextHolders方法解除RequestAttributes,ServletRequestAttributes和当前线程的绑定
      resetContextHolders(request, previousLocaleContext, previousAttributes);
      if (requestAttributes != null) {
         requestAttributes.requestCompleted();
      }

      if (logger.isDebugEnabled()) {
         if (failureCause != null) {
            this.logger.debug("Could not complete request", failureCause);
         }
         else {
            if (asyncManager.isConcurrentHandlingStarted()) {
               logger.debug("Leaving response open for concurrent processing");
            }
            else {
               this.logger.debug("Successfully completed request");
            }
         }
      }
			//注册监听事件ServletRequestHandledEvent,在调用上下文的时候产生Event
      publishRequestHandledEvent(request, response, startTime, failureCause);
   }
}
```

##1、DispatcherServlet.doService()：先做些准备工作

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
   if (logger.isDebugEnabled()) {
      String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
      logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
            " processing " + request.getMethod() + " request for [" + getRequestUri(request) + "]");
   }

   //如果是include请求，当cleanupAfterInclude为true或者attrName以org.springframework.web.servlet开头时，就保存一份request的attributeNames的快照，doDispatch()结束后，快照会覆盖request中的数据
   Map<String, Object> attributesSnapshot = null;
   if (WebUtils.isIncludeRequest(request)) {
      attributesSnapshot = new HashMap<String, Object>();
      Enumeration<?> attrNames = request.getAttributeNames();
      while (attrNames.hasMoreElements()) {
         String attrName = (String) attrNames.nextElement();
         if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
            attributesSnapshot.put(attrName, request.getAttribute(attrName));
         }
      }
   }

   //将框架相关信息存储至request，方便后面的处理器和视图用到
   request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
   request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
   request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
   request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

   FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
   if (inputFlashMap != null) {
      request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
   }
   request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
   request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

   try {
     	//请求分发
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

##2、doDispatch()：请求分发

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
        	//将request转化为Multipart request，并检查是否解析成功
         processedRequest = checkMultipart(request);
         multipartRequestParsed = (processedRequest != request);

         //步骤3.1-3.4，用于获取包含处理器Handler和拦截器AdapterIntercepters的处理器执行链HandlerExecutionChain
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null || mappedHandler.getHandler() == null) {
            noHandlerFound(processedRequest, response);
            return;
         }

         // Determine handler adapter for the current request.
         //HandlerAdapter获取到各种argumentResolvers,用来解析参数，还能获取到各种returnValueHandlers,用来处理类返回值
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

        
         //通过HandlerAdapter handle方法返回视图模型ModelAndView
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) {
            return;
         }
				 //给ModelAndView设置viewName
         applyDefaultViewName(processedRequest, mv);
         //使用applyPostHandle方法拦给已注册的拦截器放行，我们此时并没有声明拦截器，spring给我们默认生成两个默认已注册的拦截器:ResourceUrlProviderExposingInterceptor,ConversionServiceExposingInterceptor
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



