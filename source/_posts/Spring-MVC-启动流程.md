---
title: Spring MVC 之 DispatcherServlet 请求处理逻辑
date: 2023-12-26 16:13:54
tags:
    - 技术
    - JAVA
categories: 
    - Spring MVC
    - 源码
---



### 组件简介



`DispatcherServlet` 是 Spring MVC 中的核心组件，它会拦截符合特定规则的请求并将其分发到对应的处理器（Controller）进行处理。



#### Spring MVC

在 Spring MVC 应用配置文件中，可以看到以下配置：



```xml
<!-- 配置 DispatcherServlet -->
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- 配置 DispatcherServlet 的配置文件位置 -->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-mvc-config.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<!-- 映射 DispatcherServlet 拦截的 URL -->
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

```







#### Spring Boot



`org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration`



```java

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
      
      // 跟踪 webMvcProperties.getServlet().getPath() ，这段代码，会发现，path 的默认值是： / ，代表拦截所有请求
			DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
					webMvcProperties.getServlet().getPath());
      
			registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
			registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
			multipartConfig.ifAvailable(registration::setMultipartConfig);
			return registration;
		}

	}
```



### 准备阶段



​         根据`DispatcherServlet` 继承体系，可以看出它其实是一个 `Servlet` ，所以自己实例会有自己的生命周期，`init` 便是其中比较重要的用于初始化组件的方法。



```java
	@Override
	public final void init() throws ServletException {

		// Set bean properties from init parameters.
		......

		// Let subclasses do whatever initialization they like.
		initServletBean();
	}
```



```java
@Override
	protected final void initServletBean() throws ServletException {
    ......

		try {
			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
		}
		catch (ServletException | RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			throw ex;
		}
    
    ......
	}

```



```java
protected WebApplicationContext initWebApplicationContext() {
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			
      synchronized (this.onRefreshMonitor) {
				onRefresh(wac);
			}
		}
  
    ......
  
		return wac;
	}
```





```java
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		
    // 以下两个组件的逻辑差不多，都是把 IOC 容器中的 Bean（HandlerMapping / HandlerAdapter ）放到 DispatcherServlet 字段中
    initHandlerMappings(context);
		initHandlerAdapters(context);
		
    initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```



至于是何时把  HandlerMapping / HandlerAdapter  这两种组件加入 IOC 容器中，可以 阅读 `EnableWebMvc` 注解 及其 相关的类



```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```



```java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

}
```



```java
	// 从 WebMvcConfigurationSupport 类中，我们看到 注入了Spring MVC 运行需要的各种组件


  @Bean
	@SuppressWarnings("deprecation")
	public RequestMappingHandlerMapping requestMappingHandlerMapping(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

	}

	@Bean
	@Nullable
	public HandlerMapping viewControllerHandlerMapping(
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

	}


	/**
	 * Return a {@link BeanNameUrlHandlerMapping} ordered at 2 to map URL
	 * paths to controller bean names.
	 */
	@Bean
	public BeanNameUrlHandlerMapping beanNameHandlerMapping(
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

	}


	.....
  
  
```



> 以上便是，DispatchServlet  在接收请求之前的初始化逻辑，主要就是完成组件的初始化，注入响应的属性，方便后续流程。





### 接收请求





`DispatchServlet` 处理 流程的大致框架如下所示，后面仔细分析



```java

protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  try {

    try {
      // Determine handler for the current request.

      // 1、根据请求路径，匹配 controller 中的 handler 方法，以及 查找本次请求需要执行的 拦截器列表，组成一个 HandlerExecutionChain

      mappedHandler = getHandler(processedRequest);
      if (mappedHandler == null) {
        // 1、1 如果 没有 匹配的 handler ，此时需要根据情况判断是 直接抛出异常，还是转发到其他页面
        noHandlerFound(processedRequest, response);
        return;
      }

      // Determine handler adapter for the current request.

      // 2、 找到 HandlerAdapter ，从准备阶段可以看出，此时IOC 容器中 可能是有多个 HandlerAdapter ，具体由哪个执行，需要根据 HandlerAdapter 的 support 方法确定
      HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());


      // 3、 执行 HandlerInterceptor 的 preHandle 方法
      if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
      }

      // Actually invoke the handler.

      // 4、 执行方法，这其中包含 请求参数的处理（比如：url 后面的拼接的参数转换成 pojo 类， @PathVariable / @RequestBody 等等注解的处理）

      mv = ha.handle(processedRequest, response, mappedHandler.getHandler());


      // 5 、 执行 HandlerInterceptor 的 postHandle 方法
      mappedHandler.applyPostHandle(processedRequest, response, mv);
    }

    // 6.1 、 handler 被正常执行，执行 HandlerInterceptor 的 afterCompletion 方
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
  } catch (Exception ex) {
    // 6.2 、 如果执行过程中出现异常，执行 HandlerInterceptor 的 afterCompletion 方法
    triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
  } catch (Throwable err) {
    //  6.3 、 同上
    triggerAfterCompletion(processedRequest, response, mappedHandler,
        new ServletException("Handler processing failed: " + err, err));
  } finally {

  }
}
```





#### HandlerExecutionChain



```java
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





#### HandlerAdapter



````java
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
````





#### 准备执行目标方法



```java
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

    // 1、 准备调用 handler 方法
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				disableContentCachingIfNecessary(webRequest);
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
      // 2、 处理结果
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}
```



```java
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

    // 1.1、 处理请求参数
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Arguments: " + Arrays.toString(args));
		}
    
    // 1.2、 通过反射调用 handler 方法
		return doInvoke(args);
	}
```





##### 处理参数信息



```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		if (ObjectUtils.isEmpty(parameters)) {
			return EMPTY_ARGS;
		}

		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (!this.resolvers.supportsParameter(parameter)) {
				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
			}
			try {
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				// Leave stack trace for later, exception may actually be resolved and handled...
				if (logger.isDebugEnabled()) {
					String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
						logger.debug(formatArgumentError(parameter, exMsg));
					}
				}
				throw ex;
			}
		}
		return args;
	}
```







##### 反射执行 handler 方法

```java

protected Object doInvoke(Object... args) throws Exception {
		Method method = getBridgedMethod();
		try {
			if (KotlinDetector.isSuspendingFunction(method)) {
				return invokeSuspendingFunction(method, getBean(), args);
			}
			return method.invoke(getBean(), args);
		}
		......
		}
	}
```



#### 处理响应结果



```java
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}
```







