### 简单介绍

SpringMVC异步请求是基于Servlet的异步请求API封装实现，主要有以下实现方式。

| Controller方法返回类型 | 使用场景                                                     |
| ---------------------- | ------------------------------------------------------------ |
| Callable               | 简单异步执行                                                 |
| WebAsyncTask           | Callable增强版，可配置超时、回调                             |
| DeferredResult         | 外部线程控制结果                                             |
| ResponseBodyEmitter    | 流式响应，可发送多次                                         |
| SseEmitter             | Server-Sent Events，继承ResponseBodyEmitter。<br/>扩展了Content-Type响应头为text/event-stream以及响应数据的格式 |
| Flux                   | Reactor响应式流                                              |

其实SpringMVC底层只有`WebAsyncTask`、`DeferredResult`这两种方式，其它的都是基于它们实现。

其中`Callable`基于`WebAsyncTask`，而`ResponseBodyEmitter`、`SseEmitter`、`Flux`基于`DeferredResult`。

`WebAsyncTask`以及`DeferredResult`异同:

* 首先它们俩都是可以设置超时，以及各种回调(超时、完成等)。
* 最大的区别就是底层的执行，对于`WebAsyncTask`，交给容器中的`AsyncTaskExecutor`来执行。而`DeferredResult`则更灵活自由，由开发者自己设置结果，自己控制何时结束。

### 参数配置(超时、底层线程池、拦截器)

#### 超时配置

```yaml
spring:
  mvc:
    async:
      request-timeout: 30000
```

#### 底层线程池

SpringBoot中会自动寻找名为`applicationTaskExecutor`的`AsyncTaskExecutor`。其实也是`@Async`异步注解默认的Bean。

#### 手动配置

实现`WebMvcConfigurer`接口中的`configureAsyncSupport`方法即可，更灵活，还能配置拦截器。

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        configurer.setDefaultTimeout(30000L);
        configurer.setTaskExecutor();
        configurer.registerCallableInterceptors();
        configurer.registerDeferredResultInterceptors()
    }
}
```

#### Spring Boot自动配置位置

WebMvcConfigurationSupport.java

```java
@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
        @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
        @Qualifier("mvcConversionService") FormattingConversionService conversionService,
        @Qualifier("mvcValidator") Validator validator) {

    RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
    adapter.setContentNegotiationManager(contentNegotiationManager);
    adapter.setMessageConverters(getMessageConverters());
    adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer(conversionService, validator));
    adapter.setCustomArgumentResolvers(getArgumentResolvers());
    adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

    if (jackson2Present) {
        adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
        adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
    }

    AsyncSupportConfigurer configurer = getAsyncSupportConfigurer();
    // 设置线程池
    if (configurer.getTaskExecutor() != null) {
        adapter.setTaskExecutor(configurer.getTaskExecutor());
    }
    // 设置异步过程超时时间
    if (configurer.getTimeout() != null) {
        adapter.setAsyncRequestTimeout(configurer.getTimeout());
    }
    // 拦截器(用于触发回调)
    adapter.setCallableInterceptors(configurer.getCallableInterceptors());
    adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

    return adapter;
}

protected AsyncSupportConfigurer getAsyncSupportConfigurer() {
    if (this.asyncSupportConfigurer == null) {
        this.asyncSupportConfigurer = new AsyncSupportConfigurer();
        // 暴露接口给用户配置AsyncSupportConfigurer
        configureAsyncSupport(this.asyncSupportConfigurer);
    }
    return this.asyncSupportConfigurer;
}


```

SpringBoot中有一个默认的`WebMvcConfigurer`

WebMvcAutoConfiguration -> WebMvcAutoConfigurationAdapter.java

```java
@Override
public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
    if (this.beanFactory.containsBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)) {
        Object taskExecutor = this.beanFactory
            .getBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME);
        if (taskExecutor instanceof AsyncTaskExecutor) {
            configurer.setTaskExecutor(((AsyncTaskExecutor) taskExecutor));
        }
    }
    Duration timeout = this.mvcProperties.getAsync().getRequestTimeout();
    if (timeout != null) {
        configurer.setDefaultTimeout(timeout.toMillis());
    }
}
```

### 使用示例

#### Callable

```java
@GetMapping("/callable")
public Callable<Map<String, Object>> callable() {
    return () -> Map.of("name", "zhangsan");
}
```

无法单独设置超时，以及回调。

#### WebAsyncTask

```java
 /**
 * 模拟超时以及异步任务执行异常
 */
@GetMapping("/webAsyncTask")
public WebAsyncTask<String> webAsyncTask(boolean throwExp){
    Callable<String> callable = () -> {
        if (throwExp) {
            throw new IllegalArgumentException("模拟异常");
        }
        Thread.sleep(5000);
        return "success";
    };
    WebAsyncTask<String> task = new WebAsyncTask<>(1000, callable);
    task.onTimeout(() -> "timeout");
    return task;
}
```

可单独设置超时以及回调，包装了下Callable

* 正常返回success

* 若超时返回timeout，如果自己没有设置超时结果，默认是`AsyncRequestTimeoutException`
* 执行异常时，异步任务的结果是IllegalArgumentException("模拟异常")，此时响应取决于统一异常返回是如何处理IllegalArgumentException

底层的实现细节

```java
Future<?> future = this.taskExecutor.submit(() -> {
    Object result = null;
    try {
        interceptorChain.applyPreProcess(this.asyncWebRequest, callable);
        result = callable.call();
    }
    catch (Throwable ex) {
        result = ex;
    }
    finally {
        result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, result);
    }
    // 设置结果
    setConcurrentResultAndDispatch(result);
});
```

#### DeferredResult

```java
@GetMapping("/deferredResult")
public DeferredResult<String> deferredResult(@RequestParam(defaultValue = "10000") long timeout, boolean throwExp) {
    DeferredResult<String> result = new DeferredResult<>(timeout);
    Thread t = new Thread(() -> {
        try {
            if (throwExp) {
                throw new IllegalArgumentException("模拟异常");
            }
            Thread.sleep(3000);
            // 设置成功结果
            result.setResult("success");
        } catch (Exception e) {
            if (e instanceof InterruptedException) {
                log.info("线程被中断, 超时了");
            } else {
                // 设置异常结果
                result.setErrorResult(e);
            }
        }
    });
    // 超时了, 中断业务线程
    result.onTimeout(() -> {
        t.interrupt();
        // 或者直接在DeferredResult构造方法中指定超时的结果
        result.setResult("timeout");
    });
    t.start();
    return result;
}
```

可单独设置超时以及回调

* 正常返回success

* 若超时返回timeout，如果自己没有设置超时结果，默认是`AsyncRequestTimeoutException`
* 执行异常时，异步任务的结果是IllegalArgumentException("模拟异常")，此时响应取决于统一异常返回是如何处理IllegalArgumentException

`setResult`或者`setErrorResult`有幂等性保证，重复设置没有效果。

`setResult`或者`setErrorResult`都会触发`DeferredResultHandler`执行，正是通过这个机制，SpringMVC可以设置一个ResultHandler来结束异步过程。

**注: 有一个坑就是，必须要处理业务逻辑的异常，如果发生异常没有处理，相当于没有调用`setResult`或者`setErrorResult`。则会等到超时，进入超时回调。**

#### ResponseBodyEmitter

用于服务器持续输出数据。

超时了就不能调用send方法了，调用send会抛出异常

```java
@GetMapping("/responseBodyEmitter")
public ResponseBodyEmitter responseBodyEmitter(@RequestParam(defaultValue = "10000") long timeout, boolean throwExp) {
    ResponseBodyEmitter emitter = new ResponseBodyEmitter(timeout);
    new Thread(() -> {
        try {
            if (throwExp) {
                throw new IllegalArgumentException("模拟异常");
            }
            for (int i = 0; i < 5; i++) {
                /*
                 * send方法会向响应流写入数据，并且flush
                 * 可指定MediaType, 用于挑选HttpMessageConverter, 不会影响响应头Content-Type
                 */
                log.info("message-{}", i);
                emitter.send("message-" + i, MediaType.TEXT_PLAIN);
                Thread.sleep(1000);
            }
            // 通知Spring MVC完成异步过程
            emitter.complete();
        } catch (Exception e) {
            // 以异常结束
            emitter.completeWithError(e);
        }
    }).start();
    // 超时回调
    emitter.onTimeout(() -> {
        // 这里不能调用send, 此时emitter complete属性为true了, 调用会报错
        emitter.completeWithError(new RuntimeException("timeout"));
    });
    return emitter;
}
```

`ResponseBodyEmitter`内部有一个`handler`属性，Spring MVC会在`ResponseBodyEmitterReturnValueHandler`的`handleReturnValue`方法中对其进行初始化，对应的实现为`HttpMessageConvertingHandler`，内部持有响应流以及`DeferredResult`对象。

* `ResponseBodyEmitter.send`：会先检查自己的状态属性`complete`是否为true，如果已完成则会抛异常。否则调用`handler`的`send`方法，完成一次数据的write以及flush
* `ResponseBodyEmitter.complete`：修改属性`complete`为true，并调用`handler`的`complete`方法，进而调用内部`DeferredResult`对象的`setResult(null)`，这里还有一个坑，如果send发生异常，会修改`sendFailed`为true，导致`complete`以及`completeWithError`没有作用，直接结束。
* `ResponseBodyEmitter.completeWithError`：修改属性`complete`为true，并调用`handler`的`completeWithError`，进而调用内部`DeferredResult`对象的`setErrorResult(e)`

* 超时回调内部包了一层`DefaultCallback`，发生超时时会先设置`complete`为true，然后再调用用户传入的callback，如果自己不修改结果，默认是`AsyncRequestTimeoutException`

  ```java
  private final DefaultCallback timeoutCallback = new DefaultCallback();
  
  public synchronized void onTimeout(Runnable callback) {
      this.timeoutCallback.setDelegate(callback);
  }
  
  private class DefaultCallback implements Runnable {
  
      @Nullable
      private Runnable delegate;
  
      public void setDelegate(Runnable delegate) {
          this.delegate = delegate;
      }
  
      @Override
      public void run() {
          ResponseBodyEmitter.this.complete = true;
          if (this.delegate != null) {
              this.delegate.run();
          }
      }
  }
  
  ```

* 默认不会设置响应头`Content-Type`

#### SseEmitter

SSE(Server Sent Event)，用于专门支持Content-Type: text/event-stream。

`SseEmitter`继承了`ResponseBodyEmitter`

* 重写了`extendResponse`方法，设置Content-Type为text/event-stream
* 重写了send方法，主要是修改响应数据格式，以符合SSE的数据格式要求

```java
@GetMapping(value = "/sse")
public SseEmitter sse(@RequestParam(defaultValue = "10000") long timeout, boolean throwExp) {
    SseEmitter emitter = new SseEmitter(timeout);
    new Thread(() -> {
        try {
            if (throwExp) {
                throw new IllegalArgumentException("模拟异常");
            }
            for (int i = 0; i < 5; i++) {
                /*
                 * send方法会向响应流写入数据，并且flush
                 */
                emitter.send(SseEmitter.event().id(i + "").name("create").data("message-" + i, MediaType.TEXT_PLAIN));
                Thread.sleep(1000);
            }
            // 通知Spring MVC完成异步过程
            emitter.complete();
        } catch (Exception e) {
            // 以异常结束
            emitter.completeWithError(e);
        }
    }).start();
    // 超时回调
    emitter.onTimeout(() -> {
        // 这里不能调用send, 此时emitter complete属性为true了, 调用会报错
        emitter.completeWithError(new RuntimeException("timeout"));
    });
    return emitter;
}
```

正常的响应结果如下:

```tex
id:1
event:create
data:message-1

id:2
event:create
data:message-2

id:3
event:create
data:message-3

id:4
event:create
data:message-4
```

#### Flux

在SpringMVC中，也是支持Flux的，和spring-webflux不一样，把它转成DefferdResult来处理。

```java
@GetMapping()
public Flux<String> flux(){
    return Flux.interval(Duration.ofSeconds(1))
        .map(
            i -> "data-" + i
        );

}
```

### HandlerMethodReturnValueHandler对应关系

| 返回类型            | HandlerMethodReturnValueHandler        | 作用                              |
| ------------------- | -------------------------------------- | --------------------------------- |
| Callable            | CallableMethodReturnValueHandler       | 处理Callable                      |
| WebAsyncTask        | AsyncTaskMethodReturnValueHandler      | 处理WebAsyncTask                  |
| DeferredResult      | DeferredResultMethodReturnValueHandler | 处理DeferredResult                |
| ResponseBodyEmitter | ResponseBodyEmitterReturnValueHandler  | 处理流式响应                      |
| SseEmitter          | ResponseBodyEmitterReturnValueHandler  | SseEmitter继承ResponseBodyEmitter |
| Flux                | ResponseBodyEmitterReturnValueHandler  | 适配Reactive类型                  |

### 底层实现原理

大致流程如下:

* `RequestMappingHandlerAdapter`初始化`WebAsyncManager`(设置超时，底层的线程池、拦截器)，并将其放到`Request`的`Attribute`中。
* 执行`Controller`方法，返回结果对象(Callable、WebAsyncTask等)
* 根据返回对象找到对应的`HandlerMethodReturnValueHandler`，从`Request`拿到`WebAsyncManager`
* 调用`WebAsyncManager`中的`startCallableProcessing(Callable<?> callable, Object... processingContext)`或者         `startCallableProcessing(final WebAsyncTask<?> webAsyncTask, Object... processingContext)`或者`startDeferredResultProcessing(final DeferredResult<?> deferredResult, Object... processingContext)`开启异步请求，底层实际调用Servlet API。
* 无论是正常完成、还是超时等，把结果存储到`WebAsyncManager`中，然后调用`AsyncContext.dispatch()`将请求转发给自己处理
* 此时再次经过SpringMVC处理，过`DispatcherServlet` -> `RequestMappingHandlerAdapter`，发现`Request`的`Attribute`存在`WebAsyncManager`对象，如何包含结果，直接返回这个结果(结果是异常，则抛异常)，而不是去调用对应Controller中的方法，避免死循环。
* 接下来就和正常的同步请求逻辑一样，通过`HandlerMethodReturnValueHandler`将结果写到响应流中。

以DeferredResult为例

RequestMappingHandlerAdapter.java

```java
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

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

        // 创建异步请求对象
        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);

        // 创建WebAsyncManager， 如果存在则直接从request的Attribute中拿
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        // 一系列初始化
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

        // 如果已经有结果了
        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            LogFormatUtils.traceDebug(logger, traceOn -> {
                String formatted = LogFormatUtils.formatValue(result, !traceOn);
                return "Resume with async result [" + formatted + "]";
            });
            // 包装一个执行方法，直接返回结果，如果result是异常，则抛出异常
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }
        // 执行方法(Controller中的方法或者上面重新包装的方法)，将结果转给HandlerMethodReturnValueHandler
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

以`DeferredResult`为例，那么对应的是`DeferredResultMethodReturnValueHandler`

DeferredResultMethodReturnValueHandler.java

```java
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue == null) {
        mavContainer.setRequestHandled(true);
        return;
    }

    DeferredResult<?> result;

    if (returnValue instanceof DeferredResult) {
        result = (DeferredResult<?>) returnValue;
    }
    else if (returnValue instanceof ListenableFuture) {
        result = adaptListenableFuture((ListenableFuture<?>) returnValue);
    }
    else if (returnValue instanceof CompletionStage) {
        result = adaptCompletionStage((CompletionStage<?>) returnValue);
    }
    else {
        // Should not happen...
        throw new IllegalStateException("Unexpected return value type: " + returnValue);
    }
	// 发起异步过程
    WebAsyncUtils.getAsyncManager(webRequest).startDeferredResultProcessing(result, mavContainer);
}
```

WebAsyncManager.java

```java
public void startDeferredResultProcessing(final DeferredResult<?> deferredResult, Object... processingContext) throws Exception{

    Assert.notNull(deferredResult, "DeferredResult must not be null");
    Assert.state(this.asyncWebRequest != null, "AsyncWebRequest must not be null");

    // 设置超时
    Long timeout = deferredResult.getTimeoutValue();
    if (timeout != null) {
        this.asyncWebRequest.setTimeout(timeout);
    }

    // 设置拦截器
    List<DeferredResultProcessingInterceptor> interceptors = new ArrayList<>();
    // DeferredResult内部的拦截器，用于回调DeferredResult本身的onTimeout等回调
    interceptors.add(deferredResult.getInterceptor());
    // 全局拦截器
    interceptors.addAll(this.deferredResultInterceptors.values());
    // 一个兜底的超时拦截器, 返回AsyncRequestTimeoutException
    interceptors.add(timeoutDeferredResultInterceptor);

    final DeferredResultInterceptorChain interceptorChain = new DeferredResultInterceptorChain(interceptors);

    // 触发回调
    this.asyncWebRequest.addTimeoutHandler(() -> {
        try {
            interceptorChain.triggerAfterTimeout(this.asyncWebRequest, deferredResult);
        }
        catch (Throwable ex) {
            setConcurrentResultAndDispatch(ex);
        }
    });

    this.asyncWebRequest.addErrorHandler(ex -> {
        if (!this.errorHandlingInProgress) {
            try {
                if (!interceptorChain.triggerAfterError(this.asyncWebRequest, deferredResult, ex)) {
                    return;
                }
                deferredResult.setErrorResult(ex);
            }
            catch (Throwable interceptorEx) {
                setConcurrentResultAndDispatch(interceptorEx);
            }
        }
    });

    this.asyncWebRequest.addCompletionHandler(()
            -> interceptorChain.triggerAfterCompletion(this.asyncWebRequest, deferredResult));

    interceptorChain.applyBeforeConcurrentHandling(this.asyncWebRequest, deferredResult);
    // 开启异步过程(底层调用servlet api，设置超时，并注册监听器)
    startAsyncProcessing(processingContext);

    try {
        interceptorChain.applyPreProcess(this.asyncWebRequest, deferredResult);
        deferredResult.setResultHandler(result -> {
            result = interceptorChain.applyPostProcess(this.asyncWebRequest, deferredResult, result);
            // 保存结果，并转发给当前请求
            setConcurrentResultAndDispatch(result);
        });
    }
    catch (Throwable ex) {
        setConcurrentResultAndDispatch(ex);
    }
}

private void startAsyncProcessing(Object[] processingContext) {
    synchronized (WebAsyncManager.this) {
        this.concurrentResult = RESULT_NONE;
        this.concurrentResultContext = processingContext;
        this.errorHandlingInProgress = false;
    }
    // 调用servlet api
    this.asyncWebRequest.startAsync();

    if (logger.isDebugEnabled()) {
        logger.debug("Started async request");
    }
}

private void setConcurrentResultAndDispatch(Object result) {
    synchronized (WebAsyncManager.this) {
        if (this.concurrentResult != RESULT_NONE) {
            return;
        }
        this.concurrentResult = result;
        this.errorHandlingInProgress = (result instanceof Throwable);
    }

    if (this.asyncWebRequest.isAsyncComplete()) {
        if (logger.isDebugEnabled()) {
            logger.debug("Async result set but request already complete: " + formatRequestUri());
        }
        return;
    }

    if (logger.isDebugEnabled()) {
        boolean isError = result instanceof Throwable;
        logger.debug("Async " + (isError ? "error" : "result set") + ", dispatch to " + formatRequestUri());
    }
    this.asyncWebRequest.dispatch();
}
```

StandardServletAsyncWebRequest.java

```java
@Override
public void startAsync() {
    Assert.state(getRequest().isAsyncSupported(),
            "Async support must be enabled on a servlet and for all filters involved " +
            "in async request processing. This is done in Java code using the Servlet API " +
            "or by adding \"<async-supported>true</async-supported>\" to servlet and " +
            "filter declarations in web.xml.");
    Assert.state(!isAsyncComplete(), "Async processing has already completed");

    if (isAsyncStarted()) {
        return;
    }
    this.asyncContext = getRequest().startAsync(getRequest(), getResponse());
    this.asyncContext.addListener(this);
    if (this.timeout != null) {
        this.asyncContext.setTimeout(this.timeout);
    }
}

@Override
public void dispatch() {
    Assert.notNull(this.asyncContext, "Cannot dispatch without an AsyncContext");
    this.asyncContext.dispatch();
}
```

### 监听器(回调)触发链路

以最关心的超时回调为例。

回调的源头肯定是Servlet中的`AsyncListener`，SpringMVC中的`StandardServletAsyncWebRequest`实现了`AsyncListener`，并把回调委托给内部的`Runnable`列表。

StandardServletAsyncWebRequest.java

```java
private final List<Runnable> timeoutHandlers = new ArrayList<>();

/**
 * AsyncListener中的超时回调方法
 */
@Override
public void onTimeout(AsyncEvent event) throws IOException {
    this.timeoutHandlers.forEach(Runnable::run);
}

@Override
public void addTimeoutHandler(Runnable timeoutHandler) {
    this.timeoutHandlers.add(timeoutHandler);
}
```

因此只要往`timeoutHandlers`添加handler即可。

WebAsyncManager.java

```java
public void startDeferredResultProcessing(final DeferredResult<?> deferredResult, Object... processingContext) throws Exception{

    
    // 设置拦截器
    List<DeferredResultProcessingInterceptor> interceptors = new ArrayList<>();
    // DeferredResult内部的拦截器，用于回调DeferredResult本身的onTimeout等回调
    interceptors.add(deferredResult.getInterceptor());
    // 全局拦截器
    interceptors.addAll(this.deferredResultInterceptors.values());
    // 一个兜底的超时拦截器, 返回AsyncRequestTimeoutException
    interceptors.add(timeoutDeferredResultInterceptor);

    final DeferredResultInterceptorChain interceptorChain = new DeferredResultInterceptorChain(interceptors);

    // 添加回调handler
    this.asyncWebRequest.addTimeoutHandler(() -> {
        try {
            interceptorChain.triggerAfterTimeout(this.asyncWebRequest, deferredResult);
        }
        catch (Throwable ex) {
            setConcurrentResultAndDispatch(ex);
        }
    });
    // 省略

}
```

可以看到，又把回调委托给了`DeferredResultInterceptorChain`(拦截器链)，进而调用`DeferredResultProcessingInterceptor`拦截器中的`handleTimeout`。

而拦截器链注册了这么一个拦截器

DeferredResult.java

```java
final DeferredResultProcessingInterceptor getInterceptor() {
    return new DeferredResultProcessingInterceptor() {
        @Override
        public <S> boolean handleTimeout(NativeWebRequest request, DeferredResult<S> deferredResult) {
            boolean continueProcessing = true;
            try {
                if (timeoutCallback != null) {
                    // 触发DeferredResult内部的timeoutCallback
                    timeoutCallback.run();
                }
            }
            finally {
                Object value = timeoutResult.get();
                if (value != RESULT_NONE) {
                    continueProcessing = false;
                    try {
                        setResultInternal(value);
                    }
                    catch (Throwable ex) {
                        logger.debug("Failed to handle timeout result", ex);
                    }
                }
            }
            return continueProcessing;
        }
        @Override
        public <S> boolean handleError(NativeWebRequest request, DeferredResult<S> deferredResult, Throwable t) {
            try {
                if (errorCallback != null) {
                    errorCallback.accept(t);
                }
            }
            finally {
                try {
                    setResultInternal(t);
                }
                catch (Throwable ex) {
                    logger.debug("Failed to handle error result", ex);
                }
            }
            return false;
        }
        @Override
        public <S> void afterCompletion(NativeWebRequest request, DeferredResult<S> deferredResult) {
            expired = true;
            if (completionCallback != null) {
                completionCallback.run();
            }
        }
    };
}
```

servlet容器 -> StandardServletAsyncWebRequest.onTimeout(是一个AsyncListener) -> 委托给timeoutHandlers(StandardServletAsyncWebRequest内部) -> DeferredResultInterceptorChain -> DeferredResultProcessingInterceptor -> DeferredResult.timeoutCallback(通过DeferredResult.onTimeout设置)

### 谷歌浏览器流式接收

对于Content-Type为text/html、text/event-stream的响应，浏览器可以一边实时渲染一边接收数据，不用等到所有数据都响应完毕才一起展示。