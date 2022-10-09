### 声明

> 源码基于Spring Boot 2.3.12.RELEASE 

### 背景

此文的目的主要想弄明白为什么在Spring Boot中注册`Servlet`、`Filter`、`Listener`组件时需要加上`@ServletComponentScan`注解才能生效。

### 启动分析

Spring Boot应用程序的启动类一般如下所示

* jar包启动

```java
@SpringBootApplication
public class BootApplication {

    public static void main(String[] args) {
        SpringApplication.run(BootApplication.class, args);
    }
}
```

* war包启动

```java
@SpringBootApplication
public class BootApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(BootApplication.class);
    }
}
```

下面主要分析jar包启动流程，war包其实最终走的也是jar包启动逻辑。只不过入口不是在main方法中调用而已，当war包部署在外部容器时，servlet容器会通过SPI寻找`ServletContainerInitializer`接口实现，然后调用它的`onStartup`方法。在`spring-web`jar包中，`META-INF/service`目录下便指定了一个实现类`SpringServletContainerInitializer`，入口便是该类了，该类会调用`SpringApplication`中的run方法。

现在来看`SpringApplication`

```java
public static ConfigurableApplicationContext run(Class<?> primarySource,
                                                 String... args) {
    return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources,
                                                 String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

构造方法

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 配置类，一般为启动类
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断应用程序类型，主要根据类路径存不存在指定的类来推断，如SERVLET、REACTIVE、NONE
    // 不同的类型会对应不同的ApplicationContext实现
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 从spring.factories文件中获取ApplicationContextInitializer实现
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 从spring.factories文件中获取ApplicationListener实现
    // 注意这里的监听器不会注册到Spring容器中，而是在Spring Boot启动中依次触发，独立于Srping容器上下文
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断启动类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

核心实例run方法

```java
public ConfigurableApplicationContext run(String... args) {
    // 用于记录启动时间
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    // Spring容器
    ConfigurableApplicationContext context = null;
    configureHeadlessProperty();
    // 用于管理SpringBoot启动过程中的声明周期，目前只有一个实现类
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // starting生命周期
    listeners.starting();
    try {
        // 封装参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 构建环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        // 创建容器
        context = createApplicationContext();
        // 配置容器
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 刷新容器，实际调用applicationContext.refresh方法
        refreshContext(context);
        // 空实现
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // started生命周期
        listeners.started(context);
        // 调用ApplicationRunner、CommandLineRunner
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // running生命周期
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

`getRunListeners`方法

```java
/**
 * 从spring.factories获取SpringApplicationRunListener实现，管理启动过程中的生命周期
 * 目前只有EventPublishingRunListener一个实现，用于广播事件，与前文构造方法中提到的监听器一起工作
 * 比如加载application.yml文件到环境中，就是通过它实现的
 */
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger,
                                             getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

`prepareEnvironment`方法

```java
private ConfigurableEnvironment prepareEnvironment(
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
    // Create and configure the environment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    /*
     * 触发environmentPrepared生命周期，会广播ApplicationEnvironmentPreparedEvent事件
     * 从而触发ConfigFileApplicationListener监听器，加载application.yml文件的属性到
     * 环境中
     */
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                                                                                               deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

`createApplicationContext`方法

```java
/**
 * 创建ApplicationContext对象
 * servlet环境，实现类AnnotationConfigServletWebServerApplicationContext
 * 响应式环境， 实现类AnnotationConfigReactiveWebServerApplicationContext
 * 其它, 实现类AnnotationConfigApplicationContext
 */
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
                case SERVLET:
                    contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

`prepareContext`方法

```java
/**
 * 主要对ApplicationContext做一些初始化
 */
private void prepareContext(
    ConfigurableApplicationContext context,
    ConfigurableEnvironment environment,
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments,
    Banner printedBanner) {
    // 设置环境
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    // 调用前文构造方法中提到的ApplicationContextInitializer实例，对context做一些配置
    applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 注册配置类
    load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);
}
```

`refreshContext`方法

```java
private void refreshContext(ConfigurableApplicationContext context) {
    // 注册钩子，jvm退出时会回调
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
    /*
     * 刷新容器，这里会创建tomcat容器，实例化IOC容器中所有的单例bean
     */
    refresh((ApplicationContext) context);
}

protected void refresh(ConfigurableApplicationContext applicationContext) {
    applicationContext.refresh();
}
```

### tomcat内嵌容器创建

从`createApplicationContext`方法可知，servlet环境中，Spring容器实现类为`AnnotationConfigServletWebServerApplicationContext`，便来看看它的refresh方法。

该方法继承自父类`AbstractApplicationContext`，也就是Spring IOC容器的核心方法。

```java
@Override
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

            /**
             * 子类实现
             */
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // 初始化单例bean
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

再看`AnnotationConfigServletWebServerApplicationContext`的`onRefresh`方法。继承自`ServletWebServerApplicationContext`。

```java
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        // 创建WebServer
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
```

```java
private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    // 内嵌容器这两个都是null，外部容器servletContext不为null
    if (webServer == null && servletContext == null) {
        // 从IOC容器中获取工厂
        ServletWebServerFactory factory = getWebServerFactory();
        // 创建WebServer，其中getSelfInitializer是注册servlet，filter组件的核心入口
        this.webServer = factory.getWebServer(getSelfInitializer());
        getBeanFactory().registerSingleton("webServerGracefulShutdown",
                                           new WebServerGracefulShutdownLifecycle(this.webServer));
        getBeanFactory().registerSingleton("webServerStartStop",
                                           new WebServerStartStopLifecycle(this, this.webServer));
    }
    else if (servletContext != null) {
        try {
            // 外部容器手动调用
            getSelfInitializer().onStartup(servletContext);
        }
        catch (ServletException ex) {
            throw new ApplicationContextException("Cannot initialize servlet context", ex);
        }
    }
    initPropertySources();
}
```

```java
/**
 * 从IOC容器获取WebServer工厂
 * WebServer工厂的自动配置类ServletWebServerFactoryAutoConfiguration
 * 主要注册了tomcat、jetty、Undertow这3个servlet容器工厂，默认使用tomcat
 */
protected ServletWebServerFactory getWebServerFactory() {
    // Use bean names so that we don't consider the hierarchy
    String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
    if (beanNames.length == 0) {
        throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to missing "
                                              + "ServletWebServerFactory bean.");
    }
    if (beanNames.length > 1) {
        throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to multiple "
                                              + "ServletWebServerFactory beans : " + StringUtils.arrayToCommaDelimitedString(beanNames));
    }
    return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
}

```

`getWebServer`方法，以tomcat为例，`TomcatServletWebServerFactory.java`

```java
@Override
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

```java
protected void prepareContext(Host host, ServletContextInitializer[] initializers) {
    File documentRoot = getValidDocumentRoot();
    TomcatEmbeddedContext context = new TomcatEmbeddedContext();
    if (documentRoot != null) {
        context.setResources(new LoaderHidingResourceRoot(context));
    }
    context.setName(getContextPath());
    context.setDisplayName(getDisplayName());
    context.setPath(getContextPath());
    File docBase = (documentRoot != null) ? documentRoot : createTempDir("tomcat-docbase");
    // 熟悉的名字，tomcat部署应用程序的根目录
    context.setDocBase(docBase.getAbsolutePath());
    context.addLifecycleListener(new FixContextListener());
    context.setParentClassLoader((this.resourceLoader != null) ? this.resourceLoader.getClassLoader()
                                 : ClassUtils.getDefaultClassLoader());
    resetDefaultLocaleMapping(context);
    addLocaleMappings(context);
    try {
        context.setCreateUploadTargets(true);
    }
    catch (NoSuchMethodError ex) {
        // Tomcat is < 8.5.39. Continue.
    }
    configureTldPatterns(context);
    WebappLoader loader = new WebappLoader();
    loader.setLoaderClass(TomcatEmbeddedWebappClassLoader.class.getName());
    loader.setDelegate(true);
    context.setLoader(loader);
    if (isRegisterDefaultServlet()) {
        addDefaultServlet(context);
    }
    if (shouldRegisterJspServlet()) {
        addJspServlet(context);
        addJasperInitializer(context);
    }
    context.addLifecycleListener(new StaticResourceConfigurer(context));
    ServletContextInitializer[] initializersToUse = mergeInitializers(initializers);
    host.addChild(context);
    configureContext(context, initializersToUse);
    postProcessContext(context);
}
```

```java
protected void configureContext(Context context, 
                                ServletContextInitializer[] initializers) {
    // ServletContainerInitializer实现，并设置ServletContextInitializer列表
    TomcatStarter starter = new TomcatStarter(initializers);
    if (context instanceof TomcatEmbeddedContext) {
        TomcatEmbeddedContext embeddedContext = (TomcatEmbeddedContext) context;
        embeddedContext.setStarter(starter);
        embeddedContext.setFailCtxIfServletStartFails(true);
    }
    /*
     * 添加ServletContainerInitializer实现
     * tomcat启动时会异步调用ServletContainerInitializer实例的onStartup方法
     * 而TomcatStarter内部维护了ServletContextInitializer列表，依次调用
     * ServletContextInitializer实例的onStartup方法，从而注册Servlet、Filter组件
     */
    context.addServletContainerInitializer(starter, NO_CLASSES);
    for (LifecycleListener lifecycleListener : this.contextLifecycleListeners) {
        context.addLifecycleListener(lifecycleListener);
    }
    for (Valve valve : this.contextValves) {
        context.getPipeline().addValve(valve);
    }
    for (ErrorPage errorPage : getErrorPages()) {
        org.apache.tomcat.util.descriptor.web.ErrorPage tomcatErrorPage = new org.apache.tomcat.util.descriptor.web.ErrorPage();
        tomcatErrorPage.setLocation(errorPage.getPath());
        tomcatErrorPage.setErrorCode(errorPage.getStatusCode());
        tomcatErrorPage.setExceptionType(errorPage.getExceptionName());
        context.addErrorPage(tomcatErrorPage);
    }
    for (MimeMappings.Mapping mapping : getMimeMappings()) {
        context.addMimeMapping(mapping.getExtension(), mapping.getMimeType());
    }
    configureSession(context);
    new DisableReferenceClearingContextCustomizer().customize(context);
    for (TomcatContextCustomizer customizer : this.tomcatContextCustomizers) {
        customizer.customize(context);
    }
}
```

```java
protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
    return new TomcatWebServer(tomcat, getPort() >= 0, getShutdown());
}
```

```java
public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
    Assert.notNull(tomcat, "Tomcat Server must not be null");
    this.tomcat = tomcat;
    this.autoStart = autoStart;
    this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL) ? new GracefulShutdown(tomcat) : null;
    initialize();
}

private void initialize() throws WebServerException {
    logger.info("Tomcat initialized with port(s): " + getPortsDescription(false));
    synchronized (this.monitor) {
        try {
            addInstanceIdToEngineName();

            Context context = findContext();
            context.addLifecycleListener((event) -> {
                if (context.equals(event.getSource()) && Lifecycle.START_EVENT.equals(event.getType())) {
                    // Remove service connectors so that protocol binding doesn't
                    // happen when the service is started.
                    removeServiceConnectors();
                }
            });

            // 该方法会导致ServletContainerInitializer实例的onStartup执行
            // Start the server to trigger initialization listeners
            this.tomcat.start();

            // We can re-throw failure exception directly in the main thread
            rethrowDeferredStartupExceptions();

            try {
                ContextBindings.bindClassLoader(context, context.getNamingToken(), getClass().getClassLoader());
            }
            catch (NamingException ex) {
                // Naming is not enabled. Continue
            }

            // Unlike Jetty, all Tomcat threads are daemon threads. We create a
            // blocking non-daemon to stop immediate shutdown
            startDaemonAwaitThread();
        }
        catch (Exception ex) {
            stopSilently();
            destroySilently();
            throw new WebServerException("Unable to start embedded Tomcat", ex);
        }
    }
}
```

### 总结

下面主要总结下比较在意的点

* `SpringApplication`在创建时会从spring.factories文件中获取`ApplicationContextInitializer`以及`ApplicationListener`实现，其中`ApplicationContextInitializer`可以对`ApplicationContext`做一些定制。
* run方法中接着从spring.factories文件中获取`SpringApplicationRunListener`实现，目前只有一个实现类，即`EventPublishingRunListener`，与前面`ApplicationListener`搭配使用，用于广播事件以及监听。这里的事件监听与`ApplicationContext`中的事件监听是相互独立的，毕竟有些事件触发时，应用上下文`ApplicationContext`还没有初始化好。其中application.yml属性文件加载到`Environment`中便是借助它实现的，`SpringApplication`中`prepareEnvironment`方法执行了`environmentPrepared`方法，从而广播了`ApplicationEnvironmentPreparedEvent`事件，而`ConfigFileApplicationListener`监听了该事件，会去执行application.yml属性文件的加载逻辑。
* `Environment`初始化了之后，会创建`ApplicationContext`，对于`Servlet`环境，无论时内嵌容器还是运行在外部容器，实现类都是`AnnotationConfigServletWebServerApplicationContext`。
* 接着会调用`refresh`方法对`ApplicationContext`进行初始化，而`refresh`方法内部会调用`onRefresh`方法，然后再实例化单例bean。`AnnotationConfigServletWebServerApplicationContext`类在`onRefresh`方法中会实例化tomcat容器(默认是tomcat)，其中添加了一个`TomcatStarter`类，这是一个`ServletContainerInitializer`实现，tomcat启动时会异步调用`ServletContainerInitializer`实例的`onStartup`方法，而`TomcatStarter`的`onStartup`方法则会执行Spring Boot中`ServletContextInitializer`实例的`onStartup`方法，Spring Boot中注册`Servlet`、`Filter`、`Listener`便是通过`ServletContextInitializer`实现。这么做的目的是为了统一jar包运行和war包运行的差异，tomcat注册时只会在应用的部署目录搜索带有`@WebServlet`、`@WebFilter`注解的类，war包运行时这没有任何问题，但是jar包运行时，我们的类文件并没有在tomcat指定的部署目录中，因此不会被注册。

### 黑魔法

为了验证上面所说的想法，来做以下一个实验，在上述tomcat创建过程时，会指定tomcat的docBase目录，即应用程序的部署目录，默认情况下是一个临时目录，为了方便测试，便指定一个目录。

```java
/**
 * 实现WebServerFactoryCustomizer接口，可自定义WebServerFactory
 * 处理逻辑在ServletWebServerFactoryAutoConfiguration自动配置类中，
 * 往容器中注册了WebServerFactoryCustomizerBeanPostProcessor
 */
@Component
public class WebServerFactoryConfig implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setDocumentRoot(new File("D:\\Deploy"));
    }
}
```

```java
package com.wangtao.tomcat;

import java.io.IOException;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@WebServlet({"/test"})
public class TestServlet extends HttpServlet {

    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("=================TestServlet============");
        System.out.println(req.getRequestURI());
    }
}
```

然后往D:\\Deploy目录中增加以下内容WEB-INF\classes\com\wangtao\tomcat\TestServlet.class，**记得项目中本身不要有这个类**，要的效果就是tomcat能不能加载到这个类。

测试类

```java
@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String hello() {
        // tomcat类加载器
        System.out.println(Thread.currentThread().getContextClassLoader());
        TomcatEmbeddedWebappClassLoader classLoader = (TomcatEmbeddedWebappClassLoader) Thread.currentThread().getContextClassLoader();
        // file:/D:/Deploy/WEB-INF/classes/
        Arrays.stream(classLoader.getURLs()).forEach(System.out::println);
        // true
        System.out.println(ClassUtils.isPresent("com.wangtao.tomcat.FirstFilter", classLoader));
        // 系统类加载器
        System.out.println(HelloController.class.getClassLoader());
        // false
        System.out.println(ClassUtils.isPresent("com.wangtao.tomcat.FirstFilter", HelloController.class.getClassLoader()));
        return "Hello, Spring Boot!";
    }
}
```

从结果中可以看到，内嵌tomcat确实可以加载到部署目录下的类文件，当开开心心的去访问这个`Servlet`时却发现访问不了，然而tomcat类加载器确实可以加载到它呀，经过多番查找资料，原来tomcat扫描web.xml以及`@WebServlet`等这些注解是通过`ContextConfig`组件实现的，但是Spring Boot创建tomcat时并没有加入该组件，于是虽然可以加载到类，但是相当于是一个普通的类罢了，并没有注册成`Servlet`。于是改写配置，如下所示

```java
@Component
public class WebServerFactoryConfig implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        // 指定docBase目录
        factory.setDocumentRoot(new File("D:\\Deploy"));
        /* 
         * 添加生命周期组件，用于解析扫描web.xml、@WebFilter、@WebServlet等注解
         * 从类路径中使用SPI扫描ServletContainerInitializer实现添加到tomcat中
         */
        factory.addContextLifecycleListeners(new ContextConfig());
    }
}
```

再次访问，便能看到控制台的打印了。