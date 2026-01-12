### 概述

Spring Cloud Sleuth是Spring Boot 2.x的分布式链路追踪组件，不适用于Spring Boot 3。它的底层是zipkin的brave。

### 基本使用

只需要在pom中引入spring-cloud-sleuth-starter，你便可以获得链路追踪能力。

* 默认的日志打印格式，包含traceId、spanId

```tex
# 其中service-producer为service name, 取值spring.application.name
# ba3ca79a343923b7为traceId
# 761f1e8c321706bb为spanId
2026-01-12 21:12:50.321  INFO [service-producer,ba3ca79a343923b7,761f1e8c321706bb] 89772 --- [nio-9081-exec-1] c.w.producer.controller.UserController   : =============get=============
```

* 自动整合Spring cloud open feign、RestTemplate，使用它们发送请求时会携带traceId、spanId等头部，用于给链路的下一环节使用。
* 自动装饰Executor(JDK)、TaskExecutor(Spring)线程池，线程池必须注册到容器中才能被代理。

* spring cloud gateway中使用有局限性

```java
@Slf4j
@Component
public class LogFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String requestUrl = exchange.getRequest().getURI().getPath();
        log.info("request url: {}", requestUrl);
        return chain.filter(exchange);
    }

    /**
     * 必须放在ReactiveLoadBalancerClientFilter之前执行
     * 从nacos注册中心获取实例时会切换线程到boundedElastic-1
     * 导致traceContext丢失
     */
    @Override
    public int getOrder() {
        return ReactiveLoadBalancerClientFilter.LOAD_BALANCER_CLIENT_FILTER_ORDER - 1;
    }
}
```

### 底层原理

#### 自动配置类

BraveAutoConfiguration，会自动配置一些Brave的对象，比如Tracing、Tracer等

#### 默认日志实现

TraceEnvironmentPostProcessor

```java
@Override
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    Map<String, Object> map = new HashMap<String, Object>();
    // This doesn't work with all logging systems but it's a useful default so you see
    // traces in logs without having to configure it.
    if (Boolean.parseBoolean(environment.getProperty("spring.sleuth.enabled", "true"))) {
        if (Boolean
                .parseBoolean(environment.getProperty("spring.sleuth.default-logging-pattern-enabled", "true"))) {
            map.put("logging.pattern.level", "%5p [${spring.zipkin.service.name:"
                    + "${spring.application.name:}},%X{traceId:-},%X{spanId:-}]");
        }
        String neverRefereshables = environment.getProperty("spring.cloud.refresh.never-refreshable",
                "com.zaxxer.hikari.HikariDataSource");
        map.put("spring.cloud.refresh.never-refreshable",
                neverRefereshables + ",org.springframework.cloud.sleuth.instrument.jdbc.DataSourceWrapper");
    }
    addOrReplace(environment.getPropertySources(), map);
}
```

可以看到它改写了logging.pattern.level属性，在日志等级后面拼接了serveName、traceId、spanId。

而对于logging.pattern.level属性，Spring Boot会把环境中的这个属性写到Jvm系统属性中LOG_LEVEL_PATTERN(注意这里${}还是spring取变量的语法，不是logback的，会进行变量替换再写到Jvm系统属性中)，这样logback日志配置文件可以直接使用这个变量了，Spring Boot默认的日志配置文件为defaults.xml，该文件就用到了这个系统属性变量。

#### 异步线程池包装

ExecutorBeanPostProcessor

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (!ExecutorInstrumentor.isApplicableForInstrumentation(bean)) {
        return bean;
    }
    return new ExecutorInstrumentor(() -> sleuthAsyncProperties().getIgnoredBeans(), this.beanFactory)
            .instrument(bean, beanName);
}

/**
 * 只要是Executor的子类(去掉包装后的类型)都会进行包装
 */
public static boolean isApplicableForInstrumentation(Object bean) {
    return bean instanceof Executor && !(bean instanceof LazyTraceThreadPoolTaskExecutor
            || bean instanceof TraceableScheduledExecutorService || bean instanceof TraceableExecutorService
            || bean instanceof LazyTraceAsyncTaskExecutor || bean instanceof LazyTraceExecutor);
}
```

#### Open feign或者RestTemplate

调用下游时会自动添加相关头部

x-b3-parentspanid: [64dbb232e0c0ce25]
x-b3-sampled: [0]
x-b3-spanid: [c212e696bd8d5a56]
x-b3-traceid: [042125a08fb299ad]

### 高级使用

传递业务逻辑字段

在当前jvm内部传递，可以使用baggage local fields，不同进程之间传递，使用baggage remote fields

baggage意为行李。

remote fields需要上下游两个服务都配置

```yaml
spring:
  sleuth:
    baggage:
      remote-fields:
        - userId
        - userName
```

上游服务传递值

```java
// 往下游传递userId
BaggageField.getByName("userId").updateValue("123456");
// 往下游传递userName
BaggageField.getByName("userName").updateValue("zhangsan");
// 使用open feign调用下游服务
```

下游服务自动接收值

```java
String userId = BaggageField.getByName("userId").getValue();
String userName = BaggageField.getByName("userName").getValue();
```

### 代码示例

https://github.com/wangtaoj/spring-cloud-sleuth-learning