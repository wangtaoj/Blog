



### 代码示例

[项目地址](https://github.com/wangtaoj/SpringCloudAlibaba)，见gateway-example

[Spring Cloud Gateway官方文档](https://spring.io/projects/spring-cloud-gateway/#learn)

### 网关作用

网关可以为整个系统提供一个统一的入口，这样可以方便做一些统一的事情，比如统计流量、身份验证等。也方便客户端调用时不用记忆众多微服务的地址。

### 简单使用

引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

网关最常用的作用就是把一个请求地址路由到真实的请求地址，`Spring Cloud Gateway`提供一个简单的配置，以服务名为前缀进行拦截路由。配置如下

```yaml
server:
  port: 8084

spring:
  application:
    name: gateway-example
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    gateway:
      discovery:
        locator:
          enabled: true
```

 其中 `spring.cloud.gateway.discovery.locator.enabled=true`将会开启以**服务名为前缀**的路由拦截。

目前网关的访问地址: `http://127.0.0.1:8084`

假设有另一个服务：`nacos-producer`，真实地址：`127.0.0.1:8080`

那么访问`http://127.0.0.1:8084/nacos-producer/api/users/1`这个路径，将被路由到下面这个地址

`http://nacos-producer/api/users/1`，而`nacos-producer`是一个服务名，在请求时将被ribbon替换成真实的ip和port，于是便变成了`http://127.0.0.1:8080/api/users/1`。

### 路由断言工厂

`Spring Cloud Gateway`内置了很多路由断言工厂，具体可见官方文档，本节挑选`PathRoutePredicateFactory`作为使用例子，看名字就知道它是一个以路径做为匹配规则的路由断言工厂。

路由断言工厂的顶层接口为`RoutePredicateFactory`，抽象实现为`AbstractRoutePredicateFactory`。具体的实现一般都会继承`AbstractRoutePredicateFactory`类。

`Spring Cloud Gateway`的配置属性类为`GatewayProperties`，为路由断言工厂提供了两种配置方式，一种是传统的属性名字，另外一种是快捷方式(一行配置)，下面就以`PathRoutePredicateFactory`为例。

实现的目标：

当请求路径以/api/users/开头，就路由到`nacos-producer`服务中

先看属性名方式:

```java
@ConfigurationProperties(GatewayProperties.PREFIX)
@Validated
public class GatewayProperties {

	public static final String PREFIX = "spring.cloud.gateway";

    /**
     * 配置路由列表
     */
	private List<RouteDefinition> routes = new ArrayList<>();
    
    // ...略
}

@Validated
public class RouteDefinition {

	private String id;

    /**
     * 配置路由断言工厂
     */
	@NotEmpty
	@Valid
	private List<PredicateDefinition> predicates = new ArrayList<>();

	@Valid
	private List<FilterDefinition> filters = new ArrayList<>();

	@NotNull
	private URI uri;

	private Map<String, Object> metadata = new HashMap<>();

	private int order = 0;
    
    // ...略
}

public class PredicateDefinition {
    
	/**
	 * 路由断言工厂名称前缀
	 * PathRoutePredicateFactory: name=Path
	 * HeaderRoutePredicateFactory: name=Header
	 */
	@NotNull
	private String name;

    /**
     * 参数信息
     * map的key具体取决于路由断言工厂类中的Config类中的属性名
     * 如HeaderRoutePredicateFactory: key1=header  key2=regexp
     * 如PathRoutePredicateFactory: key1=patterns  key2=matchOptionalTrailingSeparator
     */
	private Map<String, String> args = new LinkedHashMap<>();
}
```

```yaml
spring:
  application:
    name: gateway-example
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    gateway:
      routes:
          # 路由ID
        - id: nacos-producer-route
          # 路由到哪个地址，存在注册中心时可以使用服务名, lb://是使用负载均衡(LoadBalance)
          uri: lb://nacos-producer
          predicates:
              # 断言工厂：Path即使用PathRoutePredicateFactory工厂
            - name: Path
              args:
              	# List<String>：可使用[val1,val2]形式
                patterns: /api/users/**
```

再看快捷方式:

```yaml
spring:
  application:
    name: gateway-example
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    gateway:
      routes:
      	  # 路由ID
        - id: nacos-producer-route
          # 路由到哪个地址，存在注册中心时可以使用服务名, lb://是使用负载均衡(LoadBalance)
          uri: lb://nacos-producer
          # 断言工厂：Path即使用PathRoutePredicateFactory工厂
          # =符号后面的是参数值，多个值用逗号分割
          predicates:
            - Path=/api/users/**
```

### 快捷方式绑定值规则逻辑

首先有一个前提，所有的路由断言工厂实现类名称全部以`RoutePredicateFactory`结尾，而名称的前缀部分则将作为使用哪个断言工厂的依据，这一点在上面的配置中已经体现，`PathRoutePredicateFactory`则为`Path`。

然后看参数部分的绑定逻辑，每一个断言工厂都有一个Config类，用于保存参数信息。这里有一个很重要的接口，接口名为`ShortcutConfigurable`，主要解读这个类的作用就能明白绑定逻辑了。

我们的路由断言工厂顶层接口`RoutePredicateFactory`是继承了`ShortcutConfigurable`接口的

```java
/**
 * 这个接口定义了两个重要的方法，如下所示，要想实现快捷方式配置，具体的路由断言工厂实现类
 * 都要重写这两个方法
 */
public interface ShortcutConfigurable {

    /**
     * 用于描述如何封装参数信息
     * 有3个值可选
     * DEFAULT、GATHER_LIST、GATHER_LIST_TAIL_FLAG
     */
	default ShortcutType shortcutType() {
		return ShortcutType.DEFAULT;
	}

    /**
     * 用于描述参数名字信息，有顺序的 
     */
	default List<String> shortcutFieldOrder() {
		return Collections.emptyList();
	}
    
    enum ShortcutType {
		
        /**
         * 默认行为, 将Map中的参数逐一映射
         */
		DEFAULT,

        /**
         * 将Map中的参数聚合成一个参数, 变成一个List, 赋值给shortcutFieldOrder方法
         * 第0一个元素指定的参数名
         */
		GATHER_LIST,

		/**
		 * 如果最后一个参数是boolean值，则将这个参数赋值给shortcutFieldOrder方法第1个
		 * 元素指定的参数名, 其它值聚合成一个List赋值给shortcutFieldOrder方法
         * 第0一个元素指定的参数名
		 */
		GATHER_LIST_TAIL_FLAG;
    
}
```

下面以具体例子来解释它们的区别

假设yaml中如下配置

`XXX=name1,name2`,true

例1

```java
ShortcutType shortcutType() {
    return ShortcutType.DEFAULT;
}

default List<String> shortcutFieldOrder() {
    return [arg1, arg2, arg3]
}

class Config {
    // name1
    private String arg1;
    // name2
    private String arg2;
    // true
    private boolean arg3;
}
```

例2

```java
ShortcutType shortcutType() {
    return ShortcutType.GATHER_LIST;
}

default List<String> shortcutFieldOrder() {
    return [args]
}

class Config {
    // [name1,name2,true]
    private List<String> args;
}
```

例3

```java
ShortcutType shortcutType() {
    return ShortcutType.GATHER_LIST_TAIL_FLAG;
}

default List<String> shortcutFieldOrder() {
    return [args, flag]
}

class Config {
    // [name1,name2]
    private List<String> args;
    // true
    private boolean flag;
}
```

### 自定义路由断言工厂

本节将自定义一个`PathPrefixRoutePredicateFactory`路由断言工厂，即请求包含某个路径前缀则进行拦截。

注意事项

* 名称必须以`RoutePredicateFactory`结尾
* 继承AbstractRoutePredicateFactory
* 实现`shortcutType`、`shortcutFieldOrder`方法以便支持快捷方式配置
* 实现`apply`方法来完成自定义逻辑的编写(即包含某一个前缀)
* 注册到Spring容器中

```java
package com.wangtao.gatewayexample.predicate;

import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.cloud.gateway.handler.predicate.GatewayPredicate;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.server.ServerWebExchange;

import java.net.URI;
import java.util.Collections;
import java.util.List;
import java.util.function.Predicate;

public class PathPrefixRoutePredicateFactory extends AbstractRoutePredicateFactory<PathPrefixRoutePredicateFactory.Config> {

    private static final String ARG_KEY = "prefix";

    public PathPrefixRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList(ARG_KEY);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return new GatewayPredicate() {
            @Override
            public boolean test(ServerWebExchange exchange) {
                URI uri = exchange.getRequest().getURI();
                return uri.getPath().startsWith(config.getPrefix());
            }
        };
    }

    @Validated
    public static class Config {

        private String prefix;

        public String getPrefix() {
            return prefix;
        }

        public void setPrefix(String prefix) {
            this.prefix = prefix;
        }
    }
}

```

配置使用

```yaml
spring:
  application:
    name: gateway-example
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    gateway:
      routes:
       - id: nacos-producer-custom-route
         uri: lb://nacos-producer
         predicates:
           - PathPrefix=/api
```

或者

```yaml
spring:
  application:
    name: gateway-example
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    gateway:
      routes:
       - id: nacos-producer-custom-route
         uri: lb://nacos-producer
         predicates:
           - name: PathPrefix
           	 args:
           	   prefix: /api
             
```

### 过滤器工厂

`Spring Cloud Gateway`中内置了许多的过滤器工厂(官网中例举了30个)，当路由断言通过后，会执行**该路由**配置的一系列过滤器，这样我们可以修正或添加一些请求信息，比如增加参数，请求头等操作。其中配置方式与路由断言工厂一样，这里不再赘述。

过滤器工厂顶层接口为`GatewayFilterFactory`

本节将在请求头信息增加一个token，用于验证，将会用到`AddRequestHeaderGatewayFilterFactory`类

```yaml
spring:
  application:
    name: gateway-example
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    gateway:
      routes:
       - id: nacos-producer-custom-route
         uri: lb://nacos-producer
         predicates:
           - PathPrefix=/api
         filters:
           - AddRequestHeader=customToken,waston
```

### 访问日志配置

通过JVM启动参数打开，不能通过`application.yaml`文件配置

` -Dreactor.netty.http.server.accessLogEnabled=true `

### 跨域配置

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsWebFilter;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

/**
 * 全局跨域配置
 */
@Configuration
public class CorsConfig {

    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        // 允许的请求头
        corsConfiguration.addAllowedHeader("*");
        // 允许的请求源 （如：http://localhost:8080）
        corsConfiguration.addAllowedOrigin("*");
        // 允许的请求方法 ==> GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE
        corsConfiguration.addAllowedMethod("*");
        // URL 映射 （如： /admin/**）
        UrlBasedCorsConfigurationSource urlBasedCorsConfigurationSource = new UrlBasedCorsConfigurationSource();
        urlBasedCorsConfigurationSource.registerCorsConfiguration("/**", corsConfiguration);
        return new CorsWebFilter(urlBasedCorsConfigurationSource);
    }
}
```

