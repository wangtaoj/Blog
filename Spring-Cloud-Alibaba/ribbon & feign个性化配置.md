### 代码例子

[项目地址](https://github.com/wangtaoj/SpringCloudAlibaba)，见ribbon-example、feign-example

### 原理基本介绍

一个微服务中可能会调用多个微服务提供的服务，ribbon和feign允许都具体某一个微服务进行配置，这基于Spring中父子容器这一概念实现。比如服务A即调用了服务B的方法，又调用了服务C的方法。假定服务B、服务C的服务名依次为serviceB、serviceC。那么会为serviceB、serviceC各自创建一个ApplicationContext，互不干扰。当然应用中本身的ApplicationContext作为它们的父容器，子容器可以访问父容器的bean，而父容器不能访问子容器的bean，对于ribbon来说，则体现在`SpringClientFactory`类，它继承了`NamedContextFactory`类，是一个创建父子容器的工具设施。

### ribbon原生使用

当引入`spring-cloud-starter-netflix-ribbon`依赖时，在spring.factories文件中加一个自动配置类`RibbonAutoConfiguration`，会为应用中注入一个`LoadBalancerClient`的实现`RibbonLoadBalancerClient`，我们可以使用该接口来选择服务，具体怎么选择，交给具体的负载均衡策略实现。

```java
package com.wangtao.ribbonexample.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.net.URI;

@RequestMapping("/ribbon")
@RestController
public class RibbonController {

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @GetMapping("/chooseServer")
    public String chooseServer() {
        ServiceInstance serviceInstance = loadBalancerClient.choose("nacos-producer");
        System.out.println("getInstanceId: " + serviceInstance.getInstanceId());
        System.out.println("getServiceId: " + serviceInstance.getServiceId());
        System.out.println("getHost: " + serviceInstance.getHost());
        System.out.println("getPort: " + serviceInstance.getPort());
        System.out.println("getScheme: " + serviceInstance.getScheme());
        System.out.println("getUri: " + serviceInstance.getUri());
		
        // 将nacos-producer换成具体的IP+端口
        URI originalUri = URI.create("https://nacos-producer/demo");
        System.out.println(loadBalancerClient.reconstructURI(serviceInstance, originalUri));
        return serviceInstance.getUri().toString();
    }
}

```



### ribbon配置负载均衡策略

`IRule`是ribbon负载均衡策略的顶级接口，只要在容器中创建一个实现类型，即可自定义ribbon的负载均衡策略。ribbon提供了很多的实现，比如随机策略(RandomRule)、轮询策略(RoundRobinRule)、区域可用策略(ZoneAvoidanceRule)等，具体可使用IDE查看IRule的实现类。其中ZoneAvoidanceRule是ribbon的默认策略，它会经过ZoneAvoidancePredicate、AvailabilityPredicate过滤之后，对于剩下的服务列表，使用轮询的规则。

下面就来实现这么一个需求，调用nacos-producer服务中的方法使用随机策略，其它服务中的方法使用轮询策略，特指RoundRobinRule。

```java
@Configuration
public class RibbonGlobalConfig {

    /**
     * 随机策略
     */
    @Bean
    public IRule randomRule() {
        return new RandomRule();
    }
}

```

```java
public class RibbonNacosProducerConfig {

    /**
     * 线性轮询策略
     * @Primary很重要，不然在创建ILoadBalancer会发生错误，依赖IRule实现，因为无法找到唯一的实现
     * 看下文解析
     */
    @Primary
    @Bean
    public IRule roundRobinRule() {
        return new RoundRobinRule();
    }
}

```

```java
@RibbonClients(
        value = {@RibbonClient(value = "nacos-producer", configuration = RibbonNacosProducerConfig.class)}
)
@EnableDiscoveryClient
@SpringBootApplication
public class RibbonExampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(RibbonExampleApplication.class, args);
    }
}
```

其中`RibbonGlobalConfig`作为全局策略，因为它注册到了应用的容器中，被`@Configuration`标记了，并且能被扫描到。而`RibbonNacosProducerConfig`仅仅作为**nacos-producer**子容器的bean，只对**nacos-producer**有效，它没有被`@Configuration`标记，也就意味着没有注册到应用中的容器里，而是通过`@RibbonClient`注解将它放入了**nacos-producer**子容器中。当然你也可以把`RibbonNacosProducerConfig`放到一个不会被应用扫描到的包中，然后加上`@Configuration`，表示这是一个配置类。只要不被应用中的容器扫描到。

**如果要验证配置是否生效，可以在具体策略的类中`choose`方法加入一个断点debug即可。**

**其中这里还有一个坑，如果说没有配置`RibbonGlobalConfig`这个全局的，那么会很正常的工作，这里执行的时候将会报一个bean创建一个异常，原因是无法找到一个唯一的`IRule`实现。因为我以为如果在子容器中找到了`IRule`的实现，就不会再去父容器中找了，但是事实上会例举出所有的实现，包括父容器。因此在`RibbonNacosProducerConfig`的配置上我加了一个`@Primary`。**

关于这个问题，可以用下面这段代码来复现该问题

```java
interface A {}

class A1 implements A {}

class A2 implements A {}

class B {
    private A a;
    
    public B(A a) {
        this.a  = a;
    }
} 

/**
 * 该代码模拟一个父子容器，在创建B时将发生错误，因为无法确定唯一的bean
 * 但是如果去掉B的注册
 * child.getBean(A.Class)是可以拿到A2的实现的
 * 仅仅是依赖注入时会例举出所有的实现，说实话这点不是很能理解Spring的做法，为啥不和getBean方法一样
 * 优先子容器的bean。
 */
public void test() {
        AnnotationConfigApplicationContext parent = new AnnotationConfigApplicationContext();
        parent.register(A1.class);
        parent.refresh();

        AnnotationConfigApplicationContext child = new AnnotationConfigApplicationContext();
        child.setParent(parent);
        child.register(A2.class);
        child.register(B.class);
        child.refresh();
 }
```

### feign特殊化配置

其中feign和ribbon一样，也是有父子容器的配置概念，就不多赘述了。feign体现在`FeignContext`这个类，当然了肯定也是继承了`NamedContextFactory`

```java
/**
 * 类上不加@Configuration注解, 是不想让该配置全局生效
 */
public class NacosProducerFeignConfig {

    /**
     * 配置feign的转换日志格式, 默认不打印
     * 注: 还需要打开logback的日志级别, feign接口是UserService
     * 所以将UserService的logback级别配成debug
     */
    @Bean
    public Logger.Level feginLoggerLevel() {
        return Logger.Level.FULL;
    }

    /**
     * 配置超时时间
     */
    @Bean
    public Request.Options requestOption() {
        return new Request.Options(10, TimeUnit.SECONDS, 60, TimeUnit.SECONDS, true);
    }
}

```

然后在@FeginClient中引入

```java
/**
 * 其中value或name属性为服务名称
 * contextId为子容器的名字, 默认等于value
 * FeignClient注解的configuration属性指定的配置类都是在子容器创建的
 *
 * 会创建两个bean
 * 1. beanName=[contextId].FeignClientSpecification
 *    class=org.springframework.cloud.openfeign.FeignClientSpecification
 * 2. beanName=com.wangtao.feignexample.feign.UserService
 *    class=org.springframework.cloud.openfeign.FeignClientFactoryBean
 *    FeignClientFactoryBean是一个FactoryBean, 从容器通过getObject方法可以获取
 *    到UserService的实例, 且该bean会被@Primary修饰(@FeignClient注解的primary默认为true),
 *    因此存在fallback实现类时, 注入时不用指定名称也不会报错
 */
@FeignClient(value = "nacos-producer", path = "/api/users", configuration = NacosProducerFeignConfig.class)
public interface UserService {

}
```

