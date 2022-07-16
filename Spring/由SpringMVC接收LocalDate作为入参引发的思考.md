### 声明

> 源码基于Spring Boot 2.0.4 、Spring 5.0.8

### 背景

SpringMVC接收LocalDate类型作为参数时，需要借助@DateTimeFormat注解来指定日期的格式，代码如下

```java
@RestController
public class ArgumentController {

    @GetMapping("/receiveLocalDate")
    public LocalDate receiveLocalDate(@DateTimeFormat(pattern = "yyyy-MM-dd") 
                                      LocalDate birthday) {
        return birthday;
    }
}
```

但是当在浏览器输入`http://localhost:8080/receiveLocalDate?birthday=1997-05-03`，收到一个错误

**No primary or default constructor found for class java.time.LocalDate**.

于是很自然的修改了下代码，把参数再加上一个`@RequestParam`注解，如下所示

```java
@RestController
public class ArgumentController {

    @GetMapping("/receiveLocalDate")
    public LocalDate receiveLocalDate(@RequestParam
        @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate birthday) {
        return birthday;
    }
}
```

这样子修改之后就能正常接收参数了。对于我以往的理解，`@RequestParam`注解只是可以增加一些功能，如标注参数名字，提供默认值，必输性。省略该注解不会影响到参数接收，就如以下代码

```java
@RestController
public class ArgumentController {

    @GetMapping("/receiveDate")
    public Date receiveDate(@DateTimeFormat(pattern = "yyyy-MM-dd") Date birthday) {
        return birthday;
    }
}
```

这引起了我强烈的迷惑以及探知欲

### 简要原理

SpringMVC封装参数的逻辑主要由`HandlerMethodArgumentResolver`接口处理，代码如下

```java
public interface HandlerMethodArgumentResolver {

	/**
	 * 用于判断支不支持该种参数
	 */
	boolean supportsParameter(MethodParameter parameter);

	/**
	 * 获取参数值
	 */
	Object resolveArgument(MethodParameter parameter,
                           @Nullable ModelAndViewContainer mavContainer,
						NativeWebRequest webRequest, 
                           @Nullable WebDataBinderFactory binderFactory) throws Exception;

}
```

通过IDEA可以找到该接口的很多实现类，很眼熟的如下所示

* `RequestParamMethodArgumentResolver`
* `PathVariableMethodArgumentResolver`
* `RequestResponseBodyMethodProcessor`
* `ServletModelAttributeMethodProcessor`

很容易猜到它们分别用于处理`@RequestParam`、`@PathVariable`、`@RequestBody`标注的参数，以及Pojo对象参数，代码举例如下所示

```java
@GetMapping("/queryUser")
public Integer queryUser(@RequestParam Integer userId) {
    return userId;
}

@GetMapping("/detailUser/{userId}")
public Integer detailUser(@PathVariable Integer userId) {
    return userId;
}

@PostMapping("/createUserByResponseBody")
public User createUserByRequestBody(@RequestBody User user) {
    return user;
}

@PostMapping("/createUser")
public User createUser(User user) {
    return user;
}
```

而对于背景中提出的问题主要涉及`RequestParamMethodArgumentResolver`、`ServletModelAttributeMethodProcessor`，主要看这两个类的`supportsParameter`方法

对应代码如下：

```java
/**
 * RequestParamMethodArgumentResolver
 * 可以看到处理的参数类型如下(大体上, 不细究)
 * 1. @RequestParam标注的参数
 * 2. useDefaultResolution为true时, 简单类型参数也被该类处理， 即使没有加@RequestParam注解
 * 
 * 何为简单类型?(详细请看BeanUtils.isSimpleProperty()方法)
 * 1. 基本类型以及对应的包装类型
 * 2. Date, String, Class, URL等
 */
@Override
public boolean supportsParameter(MethodParameter parameter) {
    if (parameter.hasParameterAnnotation(RequestParam.class)) {
        if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
            RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
            return (requestParam != null && StringUtils.hasText(requestParam.name()));
        }
        else {
            return true;
        }
    }
    else {
        if (parameter.hasParameterAnnotation(RequestPart.class)) {
            return false;
        }
        parameter = parameter.nestedIfOptional();
        if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
            return true;
        }
        else if (this.useDefaultResolution) {
            return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
        }
        else {
            return false;
        }
    }
}

/**
 * ServletModelAttributeMethodProcessor
 * 处理的参数类型如下:
 * 1. 被@ModelAttribute标注的参数
 * 2. annotationNotRequired为ture时, 非简单类型参数, 即使没有被@ModelAttribute标注
 */
@Override
public boolean supportsParameter(MethodParameter parameter) {
    return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
            (this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
}
```

**看到这里就能明白当没有加`@RequestParam`注解时，为什么不能正常接收`LocalDate`类型参数，却能正常接收`Date`类型参数了。因为`LocalDate`不是一个简单类型，当没有被`@RequestParam`注解标注时，是由`ServletModelAttributeMethodProcessor`来处理的，它把`LocalDate`参数当作一个模型对象(简单理解POJO对象），也就是说先会调用构造方法初始化实例对象，然后赋值它的属性，于是报了如上的错误。**

### 参数封装流程

入口在`RequestMappingHandlerAdapter`类中，封装参数的类为`HandlerMethodArgumentResolverComposite`，该类也实现了`HandlerMethodArgumentResolver`接口，使用组合模式，内部维护了一个`List<HandlerMethodArgumentResolver>`。

```java
/**
 * 初始化HandlerMethodArgumentResolverComposite对象
 */
@Override
public void afterPropertiesSet() {
    // Do this first, it may add ResponseBody advice beans
    initControllerAdviceCache();

    if (this.argumentResolvers == null) {
        List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
        this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
    }
    // 略
}

private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

    // Annotation-based argument resolution
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
    resolvers.add(new RequestParamMapMethodArgumentResolver());
    resolvers.add(new PathVariableMethodArgumentResolver());
    resolvers.add(new PathVariableMapMethodArgumentResolver());
    resolvers.add(new MatrixVariableMethodArgumentResolver());
    resolvers.add(new MatrixVariableMapMethodArgumentResolver());
    resolvers.add(new ServletModelAttributeMethodProcessor(false));
    resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new RequestHeaderMapMethodArgumentResolver());
    resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new SessionAttributeMethodArgumentResolver());
    resolvers.add(new RequestAttributeMethodArgumentResolver());

    // Type-based argument resolution
    resolvers.add(new ServletRequestMethodArgumentResolver());
    resolvers.add(new ServletResponseMethodArgumentResolver());
    resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RedirectAttributesMethodArgumentResolver());
    resolvers.add(new ModelMethodProcessor());
    resolvers.add(new MapMethodProcessor());
    resolvers.add(new ErrorsMethodArgumentResolver());
    resolvers.add(new SessionStatusMethodArgumentResolver());
    resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

    // Custom arguments
    if (getCustomArgumentResolvers() != null) {
        resolvers.addAll(getCustomArgumentResolvers());
    }

    // Catch-all
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
    resolvers.add(new ServletModelAttributeMethodProcessor(true));

    return resolvers;
}
```

可以看到`RequestParamMethodArgumentResolver`，`ServletModelAttributeMethodProcessor`这两个处理器被添加了两次，参数为true被放到了List的最后面，用于兜底。所以即使参数没有被`@RequestParam`、`ModelAttribute`标注，也能被处理到。

```java
/**
 * 入口执行方法
 */
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                           HttpServletResponse response, 
                                           HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        // 将argumentResolvers赋值给invocableMethod对象
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            LogFormatUtils.traceDebug(logger, traceOn -> {
                String formatted = LogFormatUtils.formatValue(result, !traceOn);
                return "Resume with async result [" + formatted + "]";
            });
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }

        // 执行方法
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }

        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```

```java
/**
 * ServletInvocableHandlerMethod类
 */
public void invokeAndHandle(ServletWebRequest webRequest, 
                            ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

    // 看invokeForRequest方法
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

@Nullable
public Object invokeForRequest(NativeWebRequest request,
                               @Nullable ModelAndViewContainer mavContainer,
                               Object... providedArgs) throws Exception {

    // 看getMethodArgumentValues方法
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }
    return doInvoke(args);
}

protected Object[] getMethodArgumentValues(NativeWebRequest request,\
                                           @Nullable ModelAndViewContainer mavContainer,
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

