> Spring Cloud 2021.0.5

### 相关类

先认识下相关的几个类

* RoutePredicateFactory，断言工厂，用于创建具体的断言。
* GatewayFilterFactory，过滤器工厂，用于创建具体的过滤器。
* Predicate，断言接口。
* GatewayFilter，过滤器接口。
* RouteDefinition，路由定义对象，在yml里配置的路由规则其实就是配置它，包含一组断言工厂和过滤器工厂。
* Route, 路由对象，包含了一组断言规则列表和过滤器列表。
* RouteDefinitionLocator，用于获取一组RouteDefinition，最常见的就是从yml里配置，或者基于服务发现的默认路由配置。
* RouteLocator，用于把RouteDefinition转换成真正的Route对象。

因此其实主要看RouteDefinitionLocator、RouteLocator这两个类的实现。

### RouteDefinitionLocator

该接口主要用来定义如何去获取一个`RouteDefinition`列表，常用的有以下3个实现。

* PropertiesRouteDefinitionLocator，从配置中获取，如application.yml文件，默认开启。
* DiscoveryClientRouteDefinitionLocator，从注册中心获取服务列表，基于服务发现的默认路由配置。需要配置`spring.cloud.gateway.discovery.locator.enabled=true`，默认false。
* CompositeRouteDefinitionLocator，看名字就知道了，把其它的`RouteDefinitionLocator`组合在一起，也是Spring Cloud Gateway默认装配的`RouteDefinitionLocator` bean。加了`@Primary`注解。

#### PropertiesRouteDefinitionLocator

```java
public class PropertiesRouteDefinitionLocator implements RouteDefinitionLocator {

    private final GatewayProperties properties;

    public PropertiesRouteDefinitionLocator(GatewayProperties properties) {
        this.properties = properties;
    }
	
    /**
     * 直接从配置获取
     */
    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(this.properties.getRoutes());
    }

}
```

#### DiscoveryClientRouteDefinitionLocator

当`spring.cloud.gateway.discovery.locator.enabled=true`时会装配的bean。想想如果我们自己在配置文件中明确配置使用服务名称作为前缀匹配的路由，应该要怎么配置。那么这个类干的就是这个事情。

假设有个服务为nacos-service

```yml
spring:
  application:
    name: gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      routes:
        - id: nacos-service-route
          uri: lb://nacos-service
          predicates:
            - Path=/nacos-service/**
          filters:
            - RewritePath=/nacos-service/?(?<segment>.*), /$\{segment}
            # 或者使用下面这个
            #- StripPrefix=1
```

当访问/nacos-service/info/port时，将被路由拦截，并且过滤器会重写路径，变成/info/port，即去掉第一段路径。

最终访问的url其实变成了lb://nacos-service/info/port，其中lb://nacos-service会被负载均衡器替换成服务对应的真实地址。

下面来看下`DiscoveryClientRouteDefinitionLocator`的实现，其实就是从注册中心获取所有的服务列表，然后挨个加入Path断言以及去掉第一段路径的过滤器。

先看自动配置类

GatewayDiscoveryClientAutoConfiguration.java

```java
public class GatewayDiscoveryClientAutoConfiguration {

    /**
     * 加入一个Path断言, 参数使用了Spel表达式，后面会将serviceId替换成真实的服务名称
     */
    public static List<PredicateDefinition> initPredicates() {
        ArrayList<PredicateDefinition> definitions = new ArrayList<>();
        // TODO: add a predicate that matches the url at /serviceId?

        // add a predicate that matches the url at /serviceId/**
        PredicateDefinition predicate = new PredicateDefinition();
        predicate.setName(normalizeRoutePredicateName(PathRoutePredicateFactory.class));
        predicate.addArg(PATTERN_KEY, "'/'+serviceId+'/**'");
        definitions.add(predicate);
        return definitions;
    }

    /**
     * 加入一个去掉第一段路径的过滤器, 参数使用了Spel表达式，后面会将serviceId替换成真实的服务名称
     */
    public static List<FilterDefinition> initFilters() {
        ArrayList<FilterDefinition> definitions = new ArrayList<>();

        // add a filter that removes /serviceId by default
        FilterDefinition filter = new FilterDefinition();
        filter.setName(normalizeFilterFactoryName(RewritePathGatewayFilterFactory.class));
        String regex = "'/' + serviceId + '/?(?<remaining>.*)'";
        String replacement = "'/${remaining}'";
        filter.addArg(REGEXP_KEY, regex);
        filter.addArg(REPLACEMENT_KEY, replacement);
        definitions.add(filter);

        return definitions;
    }

    @Bean
    public DiscoveryLocatorProperties discoveryLocatorProperties() {
        DiscoveryLocatorProperties properties = new DiscoveryLocatorProperties();
        properties.setPredicates(initPredicates());
        properties.setFilters(initFilters());
        return properties;
    }


    /**
     * 注册DiscoveryClientRouteDefinitionLocator bean
     */
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnProperty(value = "spring.cloud.discovery.reactive.enabled", matchIfMissing = true)
    public static class ReactiveDiscoveryClientRouteDefinitionLocatorConfiguration {

        @Bean
        @ConditionalOnProperty(name = "spring.cloud.gateway.discovery.locator.enabled")
        public DiscoveryClientRouteDefinitionLocator discoveryClientRouteDefinitionLocator(
                ReactiveDiscoveryClient discoveryClient, DiscoveryLocatorProperties properties) {
            return new DiscoveryClientRouteDefinitionLocator(discoveryClient, properties);
        }
    }
}
```

DiscoveryClientRouteDefinitionLocator.java

```java

/**
 * 1. 将从注册中心拉取的服务列表转换成RouteDefinition
 * 2. 将过滤器工厂，断言工厂参数中的Spel表达式中的serviceId替换成真实的服务名
 */
@Override
public Flux<RouteDefinition> getRouteDefinitions() {

    SpelExpressionParser parser = new SpelExpressionParser();
    Expression includeExpr = parser.parseExpression(properties.getIncludeExpression());
    Expression urlExpr = parser.parseExpression(properties.getUrlExpression());

    Predicate<ServiceInstance> includePredicate;
    if (properties.getIncludeExpression() == null || "true".equalsIgnoreCase(properties.getIncludeExpression())) {
        includePredicate = instance -> true;
    }
    else {
        includePredicate = instance -> {
            Boolean include = includeExpr.getValue(evalCtxt, instance, Boolean.class);
            if (include == null) {
                return false;
            }
            return include;
        };
    }

    return serviceInstances.filter(instances -> !instances.isEmpty()).flatMap(Flux::fromIterable)
            .filter(includePredicate).collectMap(ServiceInstance::getServiceId)
            // remove duplicates
            .flatMapMany(map -> Flux.fromIterable(map.values())).map(instance -> {
                RouteDefinition routeDefinition = buildRouteDefinition(urlExpr, instance);

                final ServiceInstance instanceForEval = new DelegatingServiceInstance(instance, properties);

                for (PredicateDefinition original : this.properties.getPredicates()) {
                    PredicateDefinition predicate = new PredicateDefinition();
                    predicate.setName(original.getName());
                    for (Map.Entry<String, String> entry : original.getArgs().entrySet()) {
                        String value = getValueFromExpr(evalCtxt, parser, instanceForEval, entry);
                        predicate.addArg(entry.getKey(), value);
                    }
                    routeDefinition.getPredicates().add(predicate);
                }

                for (FilterDefinition original : this.properties.getFilters()) {
                    FilterDefinition filter = new FilterDefinition();
                    filter.setName(original.getName());
                    for (Map.Entry<String, String> entry : original.getArgs().entrySet()) {
                        String value = getValueFromExpr(evalCtxt, parser, instanceForEval, entry);
                        filter.addArg(entry.getKey(), value);
                    }
                    routeDefinition.getFilters().add(filter);
                }

                return routeDefinition;
            });
}

protected RouteDefinition buildRouteDefinition(Expression urlExpr, ServiceInstance serviceInstance) {
    String serviceId = serviceInstance.getServiceId();
    RouteDefinition routeDefinition = new RouteDefinition();
    routeDefinition.setId(this.routeIdPrefix + serviceId);
    String uri = urlExpr.getValue(this.evalCtxt, serviceInstance, String.class);
    routeDefinition.setUri(URI.create(uri));
    // add instance metadata
    routeDefinition.setMetadata(new LinkedHashMap<>(serviceInstance.getMetadata()));
    return routeDefinition;
}
```

### RouteLocator

该接口主要用于把`RouteDefinition`对象转换成真实的`Route`路由对象。

主要有以下实现

* RouteDefinitionRouteLocator，主要实现，其它的都是基于此类增强功能。
* CompositeRouteLocator，用于把其它`RouteLocator`组合在一起。
* CachingRouteLocator，增加了缓存功能，**默认装载的bean实现**。

GatewayAutoConfiguration.java

```java
@Bean
public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
        List<GatewayFilterFactory> gatewayFilters, List<RoutePredicateFactory> predicates,
        RouteDefinitionLocator routeDefinitionLocator, ConfigurationService configurationService) {
    return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates, gatewayFilters, properties,
            configurationService);
}

/**
 * CachingRouteLocator持有CompositeRouteLocator引用
 * CompositeRouteLocator持有一个RouteDefinitionRouteLocator
 */
@Bean
@Primary
@ConditionalOnMissingBean(name = "cachedCompositeRouteLocator")
public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
    return new CachingRouteLocator(new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
}
```

#### RouteDefinitionRouteLocator

在Spring Cloud Gateway中，该类就是用于把`RouteDefinition`对象转换成真实的`Route`路由对象。

初始化

```java
/**
 * 将容器中的断言工厂，过滤器工厂放到Map中，key为工厂名称前缀
 * 如PathRoutePredicateFactory, 则key=Path
 * 这些断言工厂和过滤器工厂基本都在GatewayAutoConfiguration自动配置类注册到Spring容器中的
 */
public RouteDefinitionRouteLocator(RouteDefinitionLocator routeDefinitionLocator,
            List<RoutePredicateFactory> predicates, List<GatewayFilterFactory> gatewayFilterFactories,
            GatewayProperties gatewayProperties, ConfigurationService configurationService) {
    this.routeDefinitionLocator = routeDefinitionLocator;
    this.configurationService = configurationService;
    initFactories(predicates);
    gatewayFilterFactories.forEach(factory -> this.gatewayFilterFactories.put(factory.name(), factory));
    this.gatewayProperties = gatewayProperties;
}
```

将`RouteDefinition`对象转换成真实的`Route`路由对象

```java
@Override
public Flux<Route> getRoutes() {
    /*
     * 关键代码
     * 先使用routeDefinitionLocator获取所有的RouteDefinition
     * 挨个转换成Route
     */
    Flux<Route> routes = this.routeDefinitionLocator.getRouteDefinitions().map(this::convertToRoute);

    if (!gatewayProperties.isFailOnRouteDefinitionError()) {
        // instead of letting error bubble up, continue
        routes = routes.onErrorContinue((error, obj) -> {
            if (logger.isWarnEnabled()) {
                logger.warn("RouteDefinition id " + ((RouteDefinition) obj).getId()
                        + " will be ignored. Definition has invalid configs, " + error.getMessage());
            }
        });
    }

    return routes.map(route -> {
        if (logger.isDebugEnabled()) {
            logger.debug("RouteDefinition matched: " + route.getId());
        }
        return route;
    });
}

/**
 * 调用断言工厂以及过滤器工厂的apply方法生成断言和过滤器
 */
private Route convertToRoute(RouteDefinition routeDefinition) {
    AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
    List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);

    return Route.async(routeDefinition).asyncPredicate(predicate).replaceFilters(gatewayFilters).build();
}
```

获取单个路由的过滤器列表

```java
private List<GatewayFilter> getFilters(RouteDefinition routeDefinition) {
        List<GatewayFilter> filters = new ArrayList<>();

    // 默认过滤器配置, 作用于每一个配置的路由
    if (!this.gatewayProperties.getDefaultFilters().isEmpty()) {
        filters.addAll(loadGatewayFilters(routeDefinition.getId(),
                new ArrayList<>(this.gatewayProperties.getDefaultFilters())));
    }

    // 路由中单独配置的过滤器
    if (!routeDefinition.getFilters().isEmpty()) {
        filters.addAll(loadGatewayFilters(routeDefinition.getId(), new ArrayList<>(routeDefinition.getFilters())));
    }

    AnnotationAwareOrderComparator.sort(filters);
    return filters;
}

List<GatewayFilter> loadGatewayFilters(String id, List<FilterDefinition> filterDefinitions) {
    ArrayList<GatewayFilter> ordered = new ArrayList<>(filterDefinitions.size());
    for (int i = 0; i < filterDefinitions.size(); i++) {
        FilterDefinition definition = filterDefinitions.get(i);
        // 根据名称获取对应的工厂实例
        GatewayFilterFactory factory = this.gatewayFilterFactories.get(definition.getName());
        if (factory == null) {
            throw new IllegalArgumentException(
                    "Unable to find GatewayFilterFactory with name " + definition.getName());
        }
        if (logger.isDebugEnabled()) {
            logger.debug("RouteDefinition " + id + " applying filter " + definition.getArgs() + " to "
                    + definition.getName());
        }

        /*
         * 创建Config配置类，每一个工厂内部都有一个配置类，作为apply方法的参数
         * Gateway为了使得配置统一，外部配置文件配的参数都保存在map中
         * 快捷方式也一样，在FilterDefinition的String构造方法中，如Path=/api/**
         * 会使用=进行切割，将Path赋值给name，=右边的参数以逗号分割放到args map中, key为_genkey_0、_genkey_1
         * 这样子, 然后将args map中的属性绑定到Config配置中的同名属性中
         * 若是快捷方式配置, 因为key是内部生成的, 工厂会实现一个shortcutFieldOrder方法，
         * 来指定每一个位置对应的参数名是啥
         *
         * 另外，eventFunction可以指定一个事件FilterArgsEvent，在Config配置类属性绑定完成后将会触发
         * 其中RequestRateLimiterGatewayFilterFactory内部依赖的RateLimiter对象，就是通过该
         * 事件针对不同的路由绑定参数的，不同的路由使用不同的令牌桶参数。
         * 
         */
        Object configuration = this.configurationService.with(factory)
                .name(definition.getName())
                .properties(definition.getArgs())
                .eventFunction((bound, properties) -> new FilterArgsEvent(
                        // TODO: why explicit cast needed or java compile fails
                        RouteDefinitionRouteLocator.this, id, (Map<String, Object>) properties))
                .bind();

        // some filters require routeId
        // TODO: is there a better place to apply this?
        if (configuration instanceof HasRouteId) {
            HasRouteId hasRouteId = (HasRouteId) configuration;
            hasRouteId.setRouteId(id);
        }
        // 调用apply方法生成过滤器
        GatewayFilter gatewayFilter = factory.apply(configuration);
        if (gatewayFilter instanceof Ordered) {
            ordered.add(gatewayFilter);
        }
        else {
            ordered.add(new OrderedGatewayFilter(gatewayFilter, i + 1));
        }
    }

    return ordered;
}
```

断言工厂创建断言过程是类似的，就不在赘述了。

#### CachingRouteLocator

可以看到上面加载路由时每次调用时都会创建一份新的，那总不能每次请求过来都调用这个方法吧，于是带有缓存的`RouteLocator`来了。

```java
public class CachingRouteLocator
        implements Ordered, RouteLocator, ApplicationListener<RefreshRoutesEvent>, ApplicationEventPublisherAware {

    private static final Log log = LogFactory.getLog(CachingRouteLocator.class);

    private static final String CACHE_KEY = "routes";

    private final RouteLocator delegate;

    private final Flux<Route> routes;

    private final Map<String, List> cache = new ConcurrentHashMap<>();

    private ApplicationEventPublisher applicationEventPublisher;

    public CachingRouteLocator(RouteLocator delegate) {
        this.delegate = delegate;
        routes = CacheFlux.lookup(cache, CACHE_KEY, Route.class).onCacheMissResume(this::fetch);
    }

    /**
     * 获取路由列表，这个方法每次都会创建新的
     */
    private Flux<Route> fetch() {
        return this.delegate.getRoutes().sort(AnnotationAwareOrderComparator.INSTANCE);
    }

    /**
     * 直接获取成员变量
     */
    @Override
    public Flux<Route> getRoutes() {
        return this.routes;
    }

    /**
     * Clears the routes cache.
     * @return routes flux
     */
    public Flux<Route> refresh() {
        this.cache.clear();
        return this.routes;
    }

    /**
     * 监听刷新事件, 收到该事件之后，重新创建路由
     */
    @Override
    public void onApplicationEvent(RefreshRoutesEvent event) {
        try {
            fetch().collect(Collectors.toList()).subscribe(
                    list -> Flux.fromIterable(list).materialize().collect(Collectors.toList()).subscribe(signals -> {
                        applicationEventPublisher.publishEvent(new RefreshRoutesResultEvent(this));
                        cache.put(CACHE_KEY, signals);
                    }, this::handleRefreshError), this::handleRefreshError);
        }
        catch (Throwable e) {
            handleRefreshError(e);
        }
    }

    private void handleRefreshError(Throwable throwable) {
        if (log.isErrorEnabled()) {
            log.error("Refresh routes error !!!", throwable);
        }
        applicationEventPublisher.publishEvent(new RefreshRoutesResultEvent(this, throwable));
    }

    @Override
    public int getOrder() {
        return 0;
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }

}
```

### 动态路由

在`CachingRouteLocator`中，我们知道，如果想要重新创建路由只需要发布一个`RefreshRoutesEvent`即可。因此当我们使用配置中心时，当配置中心配置发生变化事只需要发布该事件即可(注意先后顺序，当配置中心的变化更新到`Environment`实例之后再发布，此时创建路由时便能从`GatewayProperties`读取到最新的配置信息，因为`@ConfigurationProperties`注解的bean是支持动态刷新的)。

另外当使用基于服务发现的路由时，如果一个新的服务注册到注册中心时，也可以发布这个事件来让gateway重新创建路由。

gateway提供了一个更上层的监听类来实现上面所说的场景。

RouteRefreshListener.java

```java
public class RouteRefreshListener implements ApplicationListener<ApplicationEvent> {

    private final ApplicationEventPublisher publisher;

    private HeartbeatMonitor monitor = new HeartbeatMonitor();

    public RouteRefreshListener(ApplicationEventPublisher publisher) {
        Assert.notNull(publisher, "publisher may not be null");
        this.publisher = publisher;
    }

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // Spring容器初始化完成之后会发布该事件
        if (event instanceof ContextRefreshedEvent) {
            ContextRefreshedEvent refreshedEvent = (ContextRefreshedEvent) event;
            if (!WebServerApplicationContext.hasServerNamespace(refreshedEvent.getApplicationContext(), "management")) {
                reset();
            }
        }
        /*
         * 配置中心发生变化后@RefreshScope或@ConfigurationProperties标注的bean刷新完之后
         * 会发布RefreshScopeRefreshedEvent事件
         * 服务注册会发布InstanceRegisteredEvent
         */
        else if (event instanceof RefreshScopeRefreshedEvent || event instanceof InstanceRegisteredEvent) {
            reset();
        }
        else if (event instanceof ParentHeartbeatEvent) {
            ParentHeartbeatEvent e = (ParentHeartbeatEvent) event;
            resetIfNeeded(e.getValue());
        }
        /*
         * 心跳事件, 如nacos实现，当spring.cloud.gateway.discovery.locator.enabled=true
         * 即开启了基于服务发现的路由时，nacos依赖包中会注册一个bean，默认每30s发布该事件
         * 这样使得gateway有机会对于新注册的服务加入到路由配置中
         */
        else if (event instanceof HeartbeatEvent) {
            HeartbeatEvent e = (HeartbeatEvent) event;
            resetIfNeeded(e.getValue());
        }
    }

    private void resetIfNeeded(Object value) {
        if (this.monitor.update(value)) {
            reset();
        }
    }

    /**
     * 发布RefreshRoutesEvent事件，重置路由
     */
    private void reset() {
        this.publisher.publishEvent(new RefreshRoutesEvent(this));
    }

}
```

