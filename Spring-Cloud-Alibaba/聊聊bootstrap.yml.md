> Spring Cloud Hoxton.SR12

### 背景

在学习配置中心时(nacos)，关于配置中心的地址等相关属性必须写在bootstrap.yml文件才会生效，而写到application.yml文件里时却不好使，强烈的好奇心使我想弄清楚这其中的困惑，看了相关资料以及源码记录下bootstrap.yml加载的流程。

### 对bootstrap.yml的理解

首先在Spring Boot应用程序里，默认只能够加载application.yml文件，而不能加载到bootstrap.yml配置文件的，这是Spring Cloud基于Spring Boot加载配置文件的扩展机制而另外提供的一种方式，并且bootstrap.yml文件加载的优先级特别高。

[下面是官网的一段解释](https://cloud.spring.io/spring-cloud-commons/multi/multi__spring_cloud_context_application_context_services.html)

>A Spring Cloud application operates by creating a “bootstrap” context, which is a parent context for the main application. It is responsible for loading configuration properties from the external sources and for decrypting properties in the local external configuration files. The two contexts share an `Environment`, which is the source of external properties for any Spring application. By default, bootstrap properties (not `bootstrap.properties` but properties that are loaded during the bootstrap phase) are added with high precedence, so they cannot be overridden by local configuration.

意译过来(基于阅读源码的理解)，在Spring Cloud应用程序中，会创建两个Spring 上下文(容器)，一个是bootstrap上下文，一个是应用程序上下文，且bootstrap上下文是应用程序上下文的父上下文。bootstrap上下文创建时会加载bootstrap.yml文件，应用程序上下文会加载appliation.yml文件，同时应用程序上下文的环境会继承父亲bootstrap上下文的环境，即可以读取到bootstrap.yml配置的属性，但是bootstrap上下文不能读取到应用程序上下文的属性(如application.yml)，**上面文档中写到两个上下文共享一个`Environment`，这里指的不是共享同一个`Environment`实例，而是应用程序上下文创建时会复制bootstrap上下文的属性源，`Environment`中的属性其实是保存在一个个的`PropertySource`中。**默认情况下，**bootstrap properties加载的优先级特别高，且不会被本地配置(application.yml、配置中心等)覆盖掉，不会覆盖仅仅指的是启动属性(也是在bootstrap.yml中配置)，但并不是bootstrap.yml配置的所有属性，如配置中心相关配置，因为拉取远程配置中心的配置类默认情况下只在bootstrap上下文中，因此也只会读取到bootstrap.yml配置的属性，这样可以有效避免被本地配置覆盖，注意拉取到的远程配置中心属性源也只会存在在应用程序上下文中，不会放到bootstrap上下文中。**

### bootstrap.yml加载过程

Spring Boot启动时加载application.yml文件的逻辑是在`ConfigFileApplicationListener`类中实现，`Environment`实例初始化早于上下文(`ApplicationContext`)创建。`ConfigFileApplicationListener`内部会去寻找`spring.factories`文件配置的`EnvironmentPostProcessor`实例列表，并执行它的`postProcessEnvironment`方法。而`ConfigFileApplicationListener`本身自己就实现了`EnvironmentPostProcessor`接口，因此加载逻辑便位于`ConfigFileApplicationListener`类中的`postProcessEnvironment`方法中，默认会加载类路径下的application.yml文件。而Spring Cloud便是用到了这个逻辑去加载bootstrap.yml文件的。

使用Spring Cloud时会包含`spring-cloud-context`依赖，该依赖`spring.factories`文件中指定了`BootstrapApplicationListener`监听器，如上面的`ConfigFileApplicationListener`一样会在Spring Boot启动时会执行，并且它的优先级高于`ConfigFileApplicationListener`，可以比较他们的`order`属性。该类会创建一个id为`bootstrap`的上下文(`AnnotationConfigApplicationContext`)，且主配置类(primaryClass)为`BootstrapImportSelectorConfiguration`，应用程序上下文的主配置类一般为main方法所在的类。最后通过`ParentContextApplicationContextInitializer`将`bootstrap`上下文设置成当前应用程序上下文的父上下文，当然了在创建这个`bootstrap`上下文同样会走一个标准的Spring Boot启动流程，会先创建环境，而在创建创建时，修改了`spring.config.name`的值为`bootstrap`，该属性的值默认为`application`，这样`ConfigFileApplicationListener`将会加载类路径下的`bootstrap.yml`文件了，而应用程序上下文创建时没有修改该属性值，默认加载的还是`application.yml`文件(此时还未加载)。最后合并到应用程序上下文的环境中，位于应用程序环境propertySource列表中的最后，key为`springCloudDefaultProperties`，此时bootstrap上下文流程结束。**创建bootstrap上下文时会给当前应用上下文增加几个`ApplicationContextInitializer`实例，在应用程序上下文环境实例创建结束后, application.yml文件加载后，将springCloudDefaultProperties源挪到最后面。**

### 源码分析

主要记录下关键执行位置

`ConfigFileApplicationListener`

```java
/**
 * 顺序
 */
public static final int DEFAULT_ORDER = Ordered.HIGHEST_PRECEDENCE + 10;

/**
 * 入口
 */
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationEnvironmentPreparedEvent) {
      onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
    }
    if (event instanceof ApplicationPreparedEvent) {
      onApplicationPreparedEvent(event);
    }
}

private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
    List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
    // 添加自己
    postProcessors.add(this);
    // 排序
    AnnotationAwareOrderComparator.sort(postProcessors);
    for (EnvironmentPostProcessor postProcessor : postProcessors) {
        // 执行具体逻辑
        postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
    }
}
```

`BootstrapApplicationListener`

```java
/**
 * 顺序, 可以看到优先级高于ConfigFileApplicationListener
 */
public static final int DEFAULT_ORDER = Ordered.HIGHEST_PRECEDENCE + 5;

/**
 * 入口
 */
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    // 应用程序环境实例
    ConfigurableEnvironment environment = event.getEnvironment();
    // 开关
    if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class,
            true)) {
        return;
    }
    // don't listen to events in a bootstrap context
    if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
        return;
    }
    // 父上下文(bootstrap上下文)
    ConfigurableApplicationContext context = null;
    String configName = environment
            .resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
    for (ApplicationContextInitializer<?> initializer : event.getSpringApplication()
            .getInitializers()) {
        if (initializer instanceof ParentContextApplicationContextInitializer) {
            context = findBootstrapContext(
                    (ParentContextApplicationContextInitializer) initializer,
                    configName);
        }
    }
    if (context == null) {
        // 创建上下文
        context = bootstrapServiceContext(environment, event.getSpringApplication(),
                configName);
        event.getSpringApplication()
                .addListeners(new CloseContextOnFailureApplicationListener(context));
    }
		// 为应用程序上下文添加一些ApplicationContextInitializer
    // 这里会添加一个PropertySourceBootstrapConfiguration，这个类提供了加载远程配置的扩展，像nacos注册中心
    // 就用到了它
    apply(context, event.getSpringApplication(), environment);
}

private ConfigurableApplicationContext bootstrapServiceContext(
            ConfigurableEnvironment environment, final SpringApplication application,
            String configName) {
    // 创建bootstrap上下文环境
    StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
    MutablePropertySources bootstrapProperties = bootstrapEnvironment
            .getPropertySources();
    for (PropertySource<?> source : bootstrapProperties) {
        bootstrapProperties.remove(source.getName());
    }
    String configLocation = environment
            .resolvePlaceholders("${spring.cloud.bootstrap.location:}");
    String configAdditionalLocation = environment
            .resolvePlaceholders("${spring.cloud.bootstrap.additional-location:}");
    Map<String, Object> bootstrapMap = new HashMap<>();
    // 修改了配置文件名称为bootstrap，默认为application
    bootstrapMap.put("spring.config.name", configName);
    // if an app (or test) uses spring.main.web-application-type=reactive, bootstrap
    // will fail
    // force the environment to use none, because if though it is set below in the
    // builder
    // the environment overrides it
    bootstrapMap.put("spring.main.web-application-type", "none");
    if (StringUtils.hasText(configLocation)) {
        bootstrapMap.put("spring.config.location", configLocation);
    }
    if (StringUtils.hasText(configAdditionalLocation)) {
        bootstrapMap.put("spring.config.additional-location",
                configAdditionalLocation);
    }
    bootstrapProperties.addFirst(
            new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));
    for (PropertySource<?> source : environment.getPropertySources()) {
        if (source instanceof StubPropertySource) {
            continue;
        }
        bootstrapProperties.addLast(source);
    }
    // TODO: is it possible or sensible to share a ResourceLoader?
    SpringApplicationBuilder builder = new SpringApplicationBuilder()
            .profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
            .environment(bootstrapEnvironment)
            // Don't use the default properties in this builder
            .registerShutdownHook(false).logStartupInfo(false)
            .web(WebApplicationType.NONE);
    final SpringApplication builderApplication = builder.application();
    if (builderApplication.getMainApplicationClass() == null) {
        // gh_425:
        // SpringApplication cannot deduce the MainApplicationClass here
        // if it is booted from SpringBootServletInitializer due to the
        // absense of the "main" method in stackTraces.
        // But luckily this method's second parameter "application" here
        // carries the real MainApplicationClass which has been explicitly
        // set by SpringBootServletInitializer itself already.
        builder.main(application.getMainApplicationClass());
    }
    if (environment.getPropertySources().contains("refreshArgs")) {
        // If we are doing a context refresh, really we only want to refresh the
        // Environment, and there are some toxic listeners (like the
        // LoggingApplicationListener) that affect global static state, so we need a
        // way to switch those off.
        builderApplication
                .setListeners(filterListeners(builderApplication.getListeners()));
    }
    // 这个便是bootstrap上下文的主配置类，可以通过它引入一些别的bean到容器中
    builder.sources(BootstrapImportSelectorConfiguration.class);
    // 创建上下文，走完整的spring boot启动流程，这里会加载bootstrap.yml文件
    final ConfigurableApplicationContext context = builder.run();
    // gh-214 using spring.application.name=bootstrap to set the context id via
    // `ContextIdApplicationContextInitializer` prevents apps from getting the actual
    // spring.application.name
    // during the bootstrap phase.
    context.setId("bootstrap");
    // 为当前应用程序上下文添加ApplicationContextInitializer
    // 这步会设置成当前应用程序上下文的父上下文, 以及将bootstrap.yml属性源挪到application.yml属性源后面
    // 当然这里只是添加，具体执行还得在Spring Boot执行ApplicationContextInitializer时执行
    // Make the bootstrap context a parent of the app context
    addAncestorInitializer(application, context);
    // It only has properties in it now that we don't want in the parent so remove
    // it (and it will be added back later)
    bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
    // 合并到应用程序环境中的propertySource列表中
    mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
    return context;
}
```

这里展开说几点，bootstrap上下文的主配置类是`BootstrapImportSelectorConfiguration`，会从`spring.factories`文件中寻找key=`BootstrapConfiguration`实例注册到容器中，类似于自动配置，因为bootstrap上下文没有bean被`@SpringBootApplication`标记，也就不会打开自动配置功能。因此往bootstrap容器添加bean需要借助这种方式。在spring-cloud-context包的`spring.factories`文件中指定了一个`PropertySourceBootstrapConfiguration`，它将会被注册到bootstrap容器中，同时它也是一个`ApplicationContextInitializer`，通过`BootstrapApplicationListener`通过`apply`方法添加到应用程序上下文的`ApplicationContextInitializer`列表中，便可以定制应用程序的上下文了。而配置中心拉取远程配置的过程便是在该类中实现。

### 案例-nacos配置中心

通过上面分析，知道配置中心拉取远程配置信息的入口在`PropertySourceBootstrapConfiguration`，下面便来看看里面内容

`PropertySourceBootstrapConfiguration`

```java
/**
 * 这里的上下文是应用程序上下文，不是bootstrap上下文
 * 不过它是bootstrap上下文中的一个bean
 */
public void initialize(ConfigurableApplicationContext applicationContext) {
    List<PropertySource<?>> composite = new ArrayList<>();
    AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
    boolean empty = true;
    // 应用程序环境实例
    ConfigurableEnvironment environment = applicationContext.getEnvironment();
    /*
     * 加载逻辑委托给了PropertySourceLocator实现类
     * 那么便寻找下bootstrap上下文的PropertySourceLocator bean，而往bootstrap上下文注册bean的方式
     * 基本上是在spring.factories文件中指定key=BootstrapConfiguration的实例列表
     * 于是往spring-cloud-starter-alibaba-nacos-config依赖中寻找，发现以下配置
     * org.springframework.cloud.bootstrap.BootstrapConfiguration=\
     * com.alibaba.cloud.nacos.NacosConfigBootstrapConfiguration
     */
    for (PropertySourceLocator locator : this.propertySourceLocators) {
        Collection<PropertySource<?>> source = locator.locateCollection(environment);
        if (source == null || source.size() == 0) {
            continue;
        }
        List<PropertySource<?>> sourceList = new ArrayList<>();
        for (PropertySource<?> p : source) {
            if (p instanceof EnumerablePropertySource) {
                EnumerablePropertySource<?> enumerable = (EnumerablePropertySource<?>) p;
                sourceList.add(new BootstrapPropertySource<>(enumerable));
            }
            else {
                sourceList.add(new SimpleBootstrapPropertySource(p));
            }
        }
        logger.info("Located property source: " + sourceList);
        composite.addAll(sourceList);
        empty = false;
    }
    if (!empty) {
        MutablePropertySources propertySources = environment.getPropertySources();
        String logConfig = environment.resolvePlaceholders("${logging.config:}");
        LogFile logFile = LogFile.get(environment);
        for (PropertySource<?> p : environment.getPropertySources()) {
            if (p.getName().startsWith(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
                propertySources.remove(p.getName());
            }
        }
        // 添加到应用程序环境propertySource列表最前面
        insertPropertySources(propertySources, composite);
        reinitializeLoggingSystem(environment, logConfig, logFile);
        setLogLevels(applicationContext, environment);
        handleIncludedProfiles(environment);
    }
}
```

`NacosConfigBootstrapConfiguration`

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(name = "spring.cloud.nacos.config.enabled", matchIfMissing = true)
public class NacosConfigBootstrapConfiguration {

    /**
     * nacos config的一些配置属性
     * 比如注册中心地址，名称空间，分组，文件名等
     * 此类是被注册到bootstrap上下文中，所以这些配置必须写到bootstrap.yml文件中
     * 而不能写到application.yml文件里
     */
    @Bean
    @ConditionalOnMissingBean
    public NacosConfigProperties nacosConfigProperties() {
        return new NacosConfigProperties();
    }

    @Bean
    @ConditionalOnMissingBean
    public NacosConfigManager nacosConfigManager(
            NacosConfigProperties nacosConfigProperties) {
        return new NacosConfigManager(nacosConfigProperties);
    }

    /**
     * 加载外部配置的PropertySourceLocator
     */
    @Bean
    public NacosPropertySourceLocator nacosPropertySourceLocator(
            NacosConfigManager nacosConfigManager) {
        return new NacosPropertySourceLocator(nacosConfigManager);
    }
}

```

`NacosPropertySourceLocator`

```java
public NacosPropertySourceLocator(NacosConfigManager nacosConfigManager) {
        this.nacosConfigManager = nacosConfigManager;
    this.nacosConfigProperties = nacosConfigManager.getNacosConfigProperties();
}

/**
 * 为应用程序的环境实例
 */
@Override
public PropertySource<?> locate(Environment env) {
    nacosConfigProperties.setEnvironment(env);
    ConfigService configService = nacosConfigManager.getConfigService();

    if (null == configService) {
        log.warn("no instance of config service found, can't load config from nacos");
        return null;
    }
    long timeout = nacosConfigProperties.getTimeout();
    nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
            timeout);
    // 配置中心文件名称(dataId)
    String name = nacosConfigProperties.getName();

    String dataIdPrefix = nacosConfigProperties.getPrefix();
    if (StringUtils.isEmpty(dataIdPrefix)) {
        dataIdPrefix = name;
    }
    // 默认取服务名(该配置从应用程序环境中读取，因此可不写在bootstrap.yml文件中)
    if (StringUtils.isEmpty(dataIdPrefix)) {
        dataIdPrefix = env.getProperty("spring.application.name");
    }

    CompositePropertySource composite = new CompositePropertySource(
            NACOS_PROPERTY_SOURCE_NAME);
    // nacos config shared
    loadSharedConfiguration(composite);
    // nacos config ext
    loadExtConfiguration(composite);
    // nacos config application
    loadApplicationConfiguration(composite, dataIdPrefix, nacosConfigProperties, env);
    return composite;
}
```

