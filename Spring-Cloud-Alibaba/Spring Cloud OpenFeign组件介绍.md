### 概述

在微服务应用中，服务间 HTTP 调用通常会涉及服务寻址、负载均衡、请求编码、响应解码、超时、异常处理和监控等环节。Spring Cloud OpenFeign 将这些能力组装成一个声明式客户端：开发者定义接口和映射注解，框架创建代理对象，并在调用接口方法时完成 HTTP 请求。

本文介绍 OpenFeign 的主要组件、它们在一次调用中的分工，以及 Spring Cloud 对 Feign 的扩展。示例配置采用当前 Spring Cloud OpenFeign 文档中的属性形式；实际项目应通过 Spring Cloud Alibaba BOM 管理 Spring Cloud 相关依赖版本，避免自行混搭。

参考：[Spring Cloud OpenFeign 官方文档](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html)、[OpenFeign 项目说明](https://github.com/spring-cloud/spring-cloud-openfeign)。

### 一次 Feign 调用经过哪些环节

先看一个典型声明式客户端：

```java
@FeignClient(name = "user-service", path = "/users")
public interface UserClient {

    @GetMapping("/{id}")
    UserDTO getById(@PathVariable("id") Long id);

    @PostMapping
    UserDTO create(@RequestBody CreateUserRequest request);
}
```

应用启动时，`@EnableFeignClients` 扫描 `@FeignClient` 接口，为接口注册一个代理 Bean。业务代码注入 `UserClient` 后，调用 `getById` 并不是直接调用远程方法，而是由代理根据接口元数据构造请求，交给底层 HTTP 客户端执行，再将响应转换为 Java 对象。

可以把主要步骤概括为：

1. `Contract` 解析 Spring MVC 注解和方法参数，形成请求模板。
2. `Encoder` 将请求体转换为 HTTP 内容，`RequestInterceptor` 等组件补充请求头或其他信息。
3. 如果使用服务名而不是固定 URL，Spring Cloud LoadBalancer 根据服务实例列表选择一个实例。
4. `Client` 通过具体 HTTP 实现发出请求。
5. 响应状态和内容交由错误处理、`Decoder` 等组件处理，最终返回接口声明的类型。

熔断、重试、监控等能力会在相应配置和依赖存在时介入这条链路。并非所有能力都由 Feign 核心库默认启用。

### 配置来源和覆盖关系

客户端组件可通过两类常见方式配置：

* Java 配置：在 `@FeignClient(configuration = ...)` 指定的配置中声明 `Encoder`、`Decoder`、`Retryer`、`ErrorDecoder`、`RequestInterceptor` 等 Bean。
* 属性配置：使用 `spring.cloud.openfeign.client.config.<clientName>` 为指定客户端配置超时、日志级别、拦截器类等属性。`<clientName>`可以设置成default，这样所有的客户端都能生效。配置类为`FeignClientProperties`

当前文档中，配置属性默认优先于 Java 配置；若希望 Java 配置优先，可调整 `spring.cloud.openfeign.client.default-to-properties`。属性名与可配置项会随版本演进，升级时应对照当前 Spring Cloud OpenFeign reference 和 Spring Cloud Alibaba 版本兼容矩阵核对。

### Feign 与 Feign.Builder：客户端构建入口

`Feign` 是 OpenFeign 提供的核心入口类，负责提供客户端构建能力。调用 `Feign.builder()` 会得到一个 `Feign.Builder`，随后可通过链式方法装配 `Client`、`Encoder`、`Decoder`、`Contract`、拦截器和日志等组件，并调用 `target(...)` 指定接口类型和目标地址，最终创建该接口的动态代理。

```java
UserApi userApi = Feign.builder()
        .contract(contract)
        .encoder(encoder)
        .decoder(decoder)
        .client(client)
        .requestInterceptor(requestInterceptor)
        .target(UserApi.class, "https://user.example.com");
```

因此，`Feign.Builder` 是组装客户端的构建器，不是一个远程客户端实例；`target` 之后返回的 `UserApi` 才是可以调用的代理。Spring Cloud OpenFeign 会在自动配置中提供和定制 Builder，再结合 `@FeignClient` 的声明、客户端上下文中的组件以及目标地址来生成 Spring Bean。需要在运行时动态创建客户端时，也可以手工使用 Builder；Spring Cloud 还提供了更贴近其客户端配置体系的 `FeignClientBuilder`，它与核心 `Feign.Builder` 不是同一个类。

### ReflectiveFeign：通过反射创建接口代理

OpenFeign 的默认同步构建流程中，`Feign.Builder.internalBuild()` 会创建 `ReflectiveFeign`；`Builder.target(target)` 则会先完成 `build()`，再调用 `newInstance(target)`。所以，`ReflectiveFeign.newInstance` 是把已经解析好的 Feign 配置变成目标接口代理的关键入口。这里创建代理发生在客户端 Bean 初始化阶段，不会在这一步发送远程 HTTP 请求。

`newInstance` 的核心工作可以分成几步：

1. 校验 `Target` 描述的类型是接口。
2. 通过 `Contract` 解析并校验接口方法上的注解，针对每个远程方法创建对应的 `MethodHandler`。同步客户端通常使用 `SynchronousMethodHandler`；它封装后续构造请求、执行请求和处理响应所需的策略。
3. 形成 `Map<Method, MethodHandler>`，把 Java 反射得到的 `Method` 映射到对应处理器。接口默认方法也会使用专门的 `DefaultMethodHandler` 处理。
4. 把目标信息和方法映射交给 `InvocationHandlerFactory` 创建 JDK `InvocationHandler`。
5. 调用 `Proxy.newProxyInstance(...)`，以客户端接口作为代理接口、以上一步的 InvocationHandler 作为调用入口，生成并返回代理对象；随后将默认方法处理器绑定到代理。

可以把这段逻辑简化成下面的伪代码：

```java
Map<Method, MethodHandler> dispatch = parseMethods(target.type());
InvocationHandler handler = invocationHandlerFactory.create(target, dispatch);
T proxy = (T) Proxy.newProxyInstance(
        target.type().getClassLoader(),
        new Class<?>[] { target.type() },
        handler);
bindDefaultMethods(proxy, dispatch);
return proxy;
```

代理创建后，业务代码调用 `userApi.getById(id)` 会进入 InvocationHandler 的 `invoke` 方法。它从 `dispatch` 中找到该 `Method` 对应的 `MethodHandler` 并调用；远程请求的构造和执行发生在这个调用阶段，而不是 `newInstance` 阶段。`equals`、`hashCode`、`toString` 等 `Object` 方法通常由 InvocationHandler 单独处理。

以上是核心同步流程的概念说明。不同 OpenFeign 版本的内部类和方法签名可能变化；可以参见 [Feign.Builder](https://github.com/OpenFeign/feign/blob/master/core/src/main/java/feign/Feign.java) 与 [ReflectiveFeign](https://raw.githubusercontent.com/OpenFeign/feign/master/core/src/main/java/feign/ReflectiveFeign.java) 源码。

### Targeter：决定客户端如何完成 target

`Targeter` 是 Spring Cloud OpenFeign 的扩展接口，处在 Builder 配置完成到客户端代理最终生成之间。它接收 `FeignClientFactoryBean`、已配置的 `Feign.Builder`、客户端上下文和目标描述（`Target.HardCodedTarget`），负责调用 Builder 的 `target(...)`，或根据额外集成采用其他目标构建方式。

默认实现 `DefaultTargeter` 基本只做一件事：调用 `feign.target(target)`，返回目标接口的代理。启用 Spring Cloud CircuitBreaker 集成时，可以使用 `FeignCircuitBreakerTargeter`：对于 CircuitBreaker Builder，它会把客户端名称和熔断配置装入 Builder，并在声明了 `fallback` 或 `fallbackFactory` 时构造对应的回退代理；若传入的不是 CircuitBreaker Builder，则回到普通 `target(...)` 路径。

所以，`Targeter` 本身不负责解析接口注解、序列化参数或发送 HTTP 请求。它是客户端装配流程中的策略点，决定配置好的 Feign Builder 最终如何生成目标客户端。可参见 [Targeter](https://github.com/spring-cloud/spring-cloud-openfeign/blob/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/Targeter.java)、[DefaultTargeter](https://github.com/spring-cloud/spring-cloud-openfeign/blob/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/DefaultTargeter.java) 和 [FeignCircuitBreakerTargeter](https://github.com/spring-cloud/spring-cloud-openfeign/blob/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/FeignCircuitBreakerTargeter.java) 源码。

### @FeignClient 与FeignClientFactory(FeignContext)

`@FeignClient` 用来声明一个远程客户端。常用属性包括：

| 属性 | 作用 |
| --- | --- |
| `name` | 客户端名称；使用服务发现和负载均衡时，它通常也是服务 ID。 |
| `url` | 指定固定目标地址；指定后通常直接访问该地址，不通过服务实例选择。 |
| `contextId` | 客户端配置上下文的标识。多个客户端访问同一服务但采用不同配置时，可用它们避免配置 Bean 冲突。 |
| `path` | 为接口上的所有方法增加统一路径前缀。 |
| `configuration` | 指定该客户端专属的组件配置。 |
| `fallback` / `fallbackFactory` | 指定熔断回退实现；需要启用相应的 CircuitBreaker 集成。 |

Spring Cloud OpenFeign 会按客户端创建命名配置上下文，并在其中组装 `FeignClientsConfiguration` 提供的默认组件以及用户自定义配置。自定义配置可以覆盖默认行为。其设计和 Spring Cloud 中的其他命名客户端类似，目的是让不同远程客户端拥有各自独立的策略。

`FeignClientFactory`(Spring Cloud 2021.x为`FeignContext`) 负责管理这些按客户端名称区分的配置上下文，并从对应上下文中取得构建 Feign 客户端所需的组件。它继承 Spring Cloud 的 `NamedContextFactory`，按需创建或取得客户端上下文，使不同客户端可以使用不同的 `Encoder`、`Decoder`、`Contract`、`Client` 等配置。它本身不执行 HTTP 请求，也不等同于最终注入业务代码的 Feign 代理。

在客户端 Bean 创建过程中，`FeignClientFactoryBean` 会读取 `@FeignClient` 的定义，借助 `FeignClientFactory` 获取相应客户端的配置组件，再配置 Builder 并交给 `Targeter` 生成目标代理。可以简单区分为：`FeignClientFactory` 管理“每个客户端使用哪些组件”，`Feign.Builder` 负责“把组件组装起来”，`Targeter` 决定“按哪种方式生成目标客户端”，最终代理负责“承接接口调用”。

客户端专属配置类需要谨慎放置：如果它被主应用的组件扫描当作全局 `@Configuration` 加载，其中定义的 Bean 可能影响其他客户端。常见做法是将客户端配置放在主扫描路径之外，或明确控制其扫描范围。

### Contract：解析接口注解

Feign 原生通过 `Contract` 定义如何把接口方法转换为请求元数据。Spring Cloud OpenFeign 默认使用 `SpringMvcContract`，所以可以直接在 Feign 接口上使用 `@GetMapping`、`@PostMapping`、`@RequestParam`、`@PathVariable`、`@RequestBody` 等 Spring MVC 注解。

`Contract` 负责理解声明，不负责真正发送网络请求。使用 Spring MVC 注解时，建议在同一个客户端接口中保持注解风格一致，不要混用 Spring MVC 注解和 Feign 原生注解，以免映射规则难以判断。

### Encoder 与 Decoder：请求和响应编解码

`Encoder` 把 Java 参数转换为请求体，例如将 `@RequestBody` 对象序列化成 JSON。Spring Cloud OpenFeign 默认提供 `SpringEncoder`，并结合 Spring 的消息转换机制处理常见内容类型。

`Decoder` 将 HTTP 响应转换为接口方法声明的返回值。默认的 `feignDecoder` 是 `ResponseEntityDecoder`，其内部包装 `SpringDecoder`，因此能够使用 Spring `HttpMessageConverter` 体系处理响应。

当项目需要特殊日期格式、厂商媒体类型、加密报文或自定义响应包装时，可以为对应客户端定制 `Encoder` 或 `Decoder`。如果返回类型是 `ResponseEntity<T>`，默认解码器也能处理响应实体语义。

### Client：执行 HTTP 请求

Feign 的 `Client` 是实际执行 HTTP 请求的抽象。Spring Cloud OpenFeign 会根据 classpath 和配置选择实现：

* 存在 Spring Cloud LoadBalancer 时，可使用 `FeignBlockingLoadBalancerClient` 包装底层客户端，让服务名请求先进行实例选择。
* 配置 Apache HttpClient 5、OkHttp 或 Java 11 HTTP Client 等实现时，底层传输能力由相应客户端提供。
* 没有负载均衡集成时，则使用可用的默认 Feign Client 实现直接请求目标地址。

负载均衡器和 HTTP 客户端是不同层次：LoadBalancer 负责从实例列表中选目标，HTTP Client 负责连接目标并传输数据。项目如果需要按服务名调用并做客户端侧负载均衡，应确认引入了 Spring Cloud LoadBalancer 依赖；OpenFeign starter 对它的支持不意味着所有组合下都自动带入该可选依赖。

连接池、TLS、代理、连接超时等底层网络行为通常由所选 HTTP 客户端决定。配置这些能力时应先确认运行时实际采用的 Client 实现。

### Request.Options：连接与读取超时

`Request.Options` 描述单次请求的连接超时和读取超时等选项。也可以通过客户端属性配置超时，例如：

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:
            # 默认10s
            connectTimeout: 2s
            # 默认1分钟
            readTimeout: 5s
```

超时应结合服务响应时间、网关超时和上游调用链共同设置。读取超时较短会更快失败，但也可能把正常的慢请求判定为失败；超时较长则会占用调用线程和连接更久。

### RequestInterceptor：统一修改请求

`RequestInterceptor` 在请求发送前修改请求模板，常用于注入认证信息、租户标识、链路追踪头或固定公共请求头。可以为特定客户端注册拦截器，也可以通过配置属性设置默认请求头和查询参数。

拦截器不应假设所有调用都来自同一个用户请求线程。涉及安全上下文或 MDC 的信息时，要考虑异步执行、线程切换以及上下文传播方式，并避免把敏感凭证写入日志。

### Logger 与 Logger.Level：请求日志

Spring Cloud OpenFeign 默认提供基于 SLF4J 的 `Slf4jLogger`。日志行为由两个设置共同决定：`Logger.Level` 决定记录多少请求和响应信息；SLF4J 日志分类的级别决定这些日志是否实际输出。Feign 日志只响应 `DEBUG` 级别，且 `Logger.Level` 的默认值是 `NONE`，即默认不记录请求细节。通常要同时设置客户端的 `loggerLevel` 和 Feign 接口对应的日志分类为 `DEBUG`。

| `Logger.Level` | 输出内容 |
| --- | --- |
| `NONE` | 不输出请求日志，默认值。 |
| `BASIC` | 请求方法、URL、响应状态码和执行耗时。 |
| `HEADERS` | `BASIC` 内容，以及请求头和响应头。 |
| `FULL` | 请求头、请求体、响应头、响应体。 |

**YAML 配置**

例如，Feign 接口为 `com.example.user.UserClient`，客户端名称为 `user-service`：

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:
            loggerLevel: full
logging:
  level:
    com.example.user.UserClient: DEBUG
```

`loggerLevel` 按 `@FeignClient` 的客户端名称配置；日志分类使用 Feign 接口的全限定类名。若只设置其中一项，另一项仍可能阻止日志输出：比如 `loggerLevel: full` 但分类级别不是 `DEBUG`，通常看不到这些日志；如果分类设为 `DEBUG` 但 `loggerLevel` 仍是默认 `NONE`，也不会记录请求细节。

**Java 配置**

也可以通过客户端专属配置类提供 `Logger.Level` Bean：

```java
import feign.Logger;
import org.springframework.context.annotation.Bean;

public class UserFeignConfiguration {

    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

并在客户端声明中引用该配置，同时仍要打开接口对应的 DEBUG 日志：

```java
@FeignClient(
        name = "user-service",
        configuration = UserFeignConfiguration.class)
public interface UserClient {
    // 方法定义
}
```

```yaml
logging:
  level:
    com.example.user.UserClient: DEBUG
```

`UserFeignConfiguration` 作为客户端专属配置时，应避免被主应用的组件扫描意外当成全局配置加载，否则其中的 `Logger.Level` Bean 可能影响其他 Feign 客户端。

生产环境启用 `HEADERS` 或 `FULL` 前应检查日志脱敏策略，避免输出令牌、Cookie、个人信息或完整请求体。

### ErrorDecoder：处理非成功响应

默认情况下，非 2xx 响应会经过 `ErrorDecoder`，转换为 Feign 异常。业务可以自定义错误解码器，把远端错误码和响应体映射为本地异常，或按服务约定构造可识别的异常类型。

错误解码器负责解释 HTTP 错误响应。是否重试还取决于解码器抛出的异常类型及 `Retryer` 策略；将所有错误都转换成可重试异常可能导致无意义的重复请求。

### Retryer：重试策略

Feign 原生的重试行为与 Spring Cloud OpenFeign 默认行为不同。Spring Cloud OpenFeign 默认创建 `Retryer.NEVER_RETRY`，遇到 `RetryableException` 后直接抛出，不自动重试；这覆盖了 Feign 原生的默认实现。Feign 原生的 `Retryer.Default` 会按有限次数重试，不能因为它是 Feign 的内置默认实现，就认为 Spring Cloud OpenFeign 默认启用了它。

**`Retryer.Default` 默认策略**

`Retryer` 接口通过 `continueOrPropagate(RetryableException)` 决定是否继续尝试：方法正常返回表示允许循环再次执行，抛出异常则结束并向上抛出。内置 `Retryer.Default` 会维护单次调用的尝试状态，并使用退避等待控制重试节奏。当前上游实现的默认参数是 `period = 100ms`、`maxPeriod = 1s`、`maxAttempts = 5`；等待间隔按约 `1.5` 倍增长并封顶。如果异常带有 `Retry-After` 时间，则优先参考该时间，并受最大等待时间约束。`maxAttempts` 表示总尝试次数，包含首次请求，因此默认最多是 1 次初始请求加 4 次重试，而不是初始请求后再重试 5 次。

**YAML 配置**

按客户端配置 `retryer` 实现类。下面启用内置 `Retryer.Default`，并把尝试总数设为默认值；类名使用内部类的 JVM 二进制名：

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:
            retryer: feign.Retryer$Default
```

配置属性接受 `Retryer` 实现类；该类需要能从 Spring 上下文取得，或者具有可用的无参构造器。若通过配置属性使用内置无参构造器，将使用它的默认尝试次数和退避参数。

**Java 配置**

可在客户端专属配置中注册 `Retryer`。例如显式设定 100 毫秒起始周期、1 秒最大周期和 5 次总尝试：

```java
import feign.Retryer;
import org.springframework.context.annotation.Bean;

public class UserFeignConfiguration {

    @Bean
    Retryer retryer() {
        return new Retryer.Default(100, 1_000, 5);
    }
}
```

```java
@FeignClient(
        name = "user-service",
        configuration = UserFeignConfiguration.class)
public interface UserClient {
    // 方法定义
}
```

较新的 Feign 版本也可以使用 `new feign.DefaultRetryer(100, 1_000, 5)`。示例使用 `Retryer.Default` 以便说明常见的旧 API 名称。`Retryer` 会通过 `clone()` 为每次方法调用创建独立实例，因此自定义实现应正确实现 `clone()`，不要让不同请求共享同一份尝试计数。

**SynchronousMethodHandler 中的重试骨架**

同步代理方法最终会进入 `SynchronousMethodHandler.invoke`。其核心逻辑是先克隆 `Retryer`，然后在循环中执行请求；捕获到 `RetryableException` 后交由 `continueOrPropagate` 决策：

```java
Retryer retryer = this.retryer.clone();
while (true) {
    try {
        return executeAndDecode(template);
    } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        // 正常返回表示策略允许重试；下一轮重新执行请求
    }
}
```

真实源码还会记录重试日志，并处理异常传播策略。网络 `IOException` 会被包装成 `RetryableException`；错误响应是否能进入重试流程，则取决于 `ErrorDecoder` 是否返回 `RetryableException`。普通业务异常以及没有被标记为可重试的 HTTP 错误，不会因为配置了 `Retryer` 就自动重试。Spring Cloud OpenFeign 的 `NEVER_RETRY` 在 `continueOrPropagate` 中直接重新抛出异常，因此上述循环不会执行第二次请求。当前同步调用实现可参见 [`SynchronousMethodHandler`](https://raw.githubusercontent.com/OpenFeign/feign/master/core/src/main/java/feign/SynchronousMethodHandler.java)。

重试会放大流量，也可能重复执行非幂等操作。对创建订单、扣款等请求，只有在请求具备幂等保障、且明确知道失败语义时才应启用重试。还需核对底层负载均衡重试、网关重试和业务重试，避免多层重试叠加。

### CircuitBreaker 与 fallback：熔断和降级

Spring Cloud OpenFeign 通过 Spring Cloud CircuitBreaker 抽象将远程调用包进断路器执行。熔断能力由 CircuitBreaker 实现负责；OpenFeign 的集成层负责把 Feign 方法调用交给对应的断路器，并把降级逻辑接入代理。

#### 启用 CircuitBreaker 集成

需要同时满足两个条件：classpath 中有 Spring Cloud CircuitBreaker API 和具体实现；并将 `spring.cloud.openfeign.circuitbreaker.enabled` 设为 `true`。例如同步应用采用 Resilience4j 时，可以使用 `spring-cloud-starter-circuitbreaker-resilience4j`，版本由 Spring Cloud BOM 管理。该 starter 为同步应用提供 CircuitBreaker 实现。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
        # 可选：启用 CircuitBreaker group
        group:
          enabled: false
```

`enabled` 默认是 `false`。打开它后，OpenFeign 自动配置才会使用 CircuitBreaker 版本的 Builder 和 Targeter。`group.enabled` 默认也是 `false`；开启后会在创建断路器时同时传入客户端分组名，具体组配置能力由所选实现提供。

具体实现的配置项与 Spring Cloud OpenFeign 的集成开关是两回事：例如上面的开关负责将 Feign 调用包装进 CircuitBreaker；Resilience4j 的窗口大小、失败阈值和超时等则由 Resilience4j 的配置管理。可参考 [OpenFeign CircuitBreaker 集成文档](https://docs.spring.io/spring-cloud-openfeign/reference/spring-cloud-openfeign.html#spring-cloud-feign-circuitbreaker) 和 [Resilience4j starter 文档](https://docs.spring.io/spring-cloud-circuitbreaker/reference/spring-cloud-circuitbreaker-resilience4j/starters.html)。

#### FeignCircuitBreaker

`FeignCircuitBreaker` 是 Spring Cloud OpenFeign 提供的集成入口，其中的 `FeignCircuitBreaker.Builder` 继承 Feign 的 `Feign.Builder`。它额外持有 `CircuitBreakerFactory`、客户端名称、group 开关和断路器名称解析器。

普通 Feign Builder 的 `build()` 会使用常规 `InvocationHandlerFactory`；而 `FeignCircuitBreaker.Builder.build(...)` 会设置一个定制的 `InvocationHandlerFactory`，使生成代理时创建 `FeignCircuitBreakerInvocationHandler`。它的 `target(...)` 重载还可以接收普通 fallback 实例或 `FallbackFactory`，再把回退对象交给构建过程。

#### FeignCircuitBreakerTargeter

`FeignCircuitBreakerTargeter` 是 Spring Cloud 的 `Targeter` 实现，参与 `@FeignClient` 代理 Bean 的构建。它检查传入的 Builder 是否为 `FeignCircuitBreaker.Builder`：如果不是，就按普通 Feign 流程调用 `feign.target(target)`；如果是，则注入 CircuitBreakerFactory、客户端标识、group 设置和名称解析器，再根据 `@FeignClient` 声明选择 `target(target)`、`target(target, fallback)` 或 `target(target, fallbackFactory)`。

当 `@FeignClient` 设置了 `fallback` 或 `fallbackFactory` 时，Targeter 会从该命名客户端的 Spring 上下文中取得对应 Bean，并检查它实现了目标 Feign 接口。回退类不是随便创建的普通对象，必须注册为 Spring Bean。

#### FeignCircuitBreakerInvocationHandler

这是实际接收 JDK 动态代理方法调用的 `InvocationHandler`。它保存断路器工厂、客户端名称、目标接口、`Method -> MethodHandler` 分发表，以及可选的 fallback factory。

业务代码调用某个 Feign 方法时，Handler 会解析该方法对应的断路器名称，通过 `CircuitBreakerFactory` 获取断路器，再将原本的 `MethodHandler` 调用包装成 `Supplier`，交给 `CircuitBreaker.run(...)` 执行。未配置降级时只执行 `run(supplier)`；有 `fallback` 时会被包装为 `FallbackFactory`，或者使用 `fallbackFactory` 创建回退实例，最终以 `run(supplier, fallbackFunction)` 执行。断路器打开或调用抛出可处理异常时，CircuitBreaker 根据策略决定是否执行 fallback。

#### CircuitBreakerNameResolver

`CircuitBreakerNameResolver` 是断路器名称解析策略接口，方法签名接收 Feign 客户端名称、`Target<?>` 和被调用的 Java `Method`，返回传给 `CircuitBreakerFactory` 的断路器 ID。`FeignCircuitBreakerInvocationHandler` 在每次调用 Feign 方法时调用它，再用返回的名称创建对应断路器。因此，解析器决定的是断路器实例名称，不负责创建断路器或执行熔断逻辑。

默认解析基于 `Feign.configKey(target.type(), method)`，名称会包含接口方法和参数类型，例如 `UserClient#getById(Long)`。属性 `spring.cloud.openfeign.circuitbreaker.alphanumeric-ids.enabled` 默认是 `true`，默认的字母数字解析器会移除名称中的非字母数字字符，示例会变成 `UserClientgetByIdLong`；设为 `false` 时保留 `Feign.configKey` 格式。配置 Resilience4j 等实现的实例属性时，配置键要与最终生成的断路器 ID 对应。

也可以提供自己的 `CircuitBreakerNameResolver` Bean。比如按服务名和方法名命名：

```java
import feign.Target;
import org.springframework.cloud.openfeign.CircuitBreakerNameResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.lang.reflect.Method;

@Configuration(proxyBeanMethods = false)
public class FeignCircuitBreakerConfiguration {

    @Bean
    CircuitBreakerNameResolver circuitBreakerNameResolver() {
        return (feignClientName, target, method) ->
                feignClientName + "_" + method.getName();
    }
}
```

自动配置在容器中没有自定义 Bean 时才创建默认解析器；提供自定义 Bean 后，它会被 `FeignCircuitBreakerTargeter` 注入并用于该应用的熔断客户端。若想对客户端区别命名，可在 resolver 中组合 `feignClientName`、`target.type()` 和 `method`。源码可参见 [CircuitBreakerNameResolver](https://raw.githubusercontent.com/spring-cloud/spring-cloud-openfeign/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/CircuitBreakerNameResolver.java) 和 [默认自动配置](https://raw.githubusercontent.com/spring-cloud/spring-cloud-openfeign/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/FeignAutoConfiguration.java)。

#### OpenFeign 如何融入 CircuitBreaker 调用链

整体构建和运行过程可以概括为：

1. 应用扫描到 `@FeignClient`，注册 `FeignClientFactoryBean`。
2. 因为开关已启用且 CircuitBreaker 实现已提供 `CircuitBreakerFactory`，自动配置选用 `FeignCircuitBreaker.Builder` 和 `FeignCircuitBreakerTargeter`。
3. FactoryBean 从客户端命名上下文装配 `Contract`、`Client`、编码解码器等组件，Targeter 再把 CircuitBreakerFactory 和可选 fallback 交给 Builder。
4. Builder 构建 Feign 时设置定制的 InvocationHandlerFactory；之后 `ReflectiveFeign.newInstance` 仍负责解析方法并创建 JDK 动态代理，但代理方法映射到 `FeignCircuitBreakerInvocationHandler`。
5. 调用代理方法时，InvocationHandler 以断路器包裹原有 `MethodHandler`。断路器放行后，原有 Feign 方法处理器继续完成请求模板构造、HTTP Client 调用和响应解码；断路器拒绝或执行失败时，按配置调用 fallback。

这说明 CircuitBreaker 集成不替换 Feign 的 Contract、Encoder、Decoder 或 HTTP Client，而是在动态代理的调用入口处包裹既有方法执行。

#### 声明 fallback 与 fallbackFactory

`fallback` 用于返回固定的降级实现，适合不需要异常原因的简单场景：

```java
@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient {
    @GetMapping("/users/{id}")
    UserDTO getById(@PathVariable("id") Long id);
}

@Component
class UserClientFallback implements UserClient {
    @Override
    public UserDTO getById(Long id) {
        return UserDTO.unavailable(id);
    }
}
```

`fallbackFactory` 的 `create(Throwable cause)` 可以获取导致回退的异常，适合记录原因或对不同失败作不同处理。两种方式都要求返回值符合接口语义；避免把服务不可用伪装成正常业务成功。

Spring Cloud CircuitBreaker 的具体实现（例如 Resilience4j）负责熔断状态、窗口和阈值等策略。实现侧可以通过配置属性或 `Customizer` 定制断路器。熔断和重试解决的问题不同：重试用于处理可能短暂恢复的失败，熔断用于阻止持续失败的下游继续消耗本地资源。配置它们时需要结合超时、并发量和服务恢复策略。

源码可参考 [FeignCircuitBreaker](https://raw.githubusercontent.com/spring-cloud/spring-cloud-openfeign/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/FeignCircuitBreaker.java)、[FeignCircuitBreakerTargeter](https://raw.githubusercontent.com/spring-cloud/spring-cloud-openfeign/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/FeignCircuitBreakerTargeter.java) 和 [FeignCircuitBreakerInvocationHandler](https://raw.githubusercontent.com/spring-cloud/spring-cloud-openfeign/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/FeignCircuitBreakerInvocationHandler.java)。

### 常见理解误区

* OpenFeign 是声明式 HTTP 客户端，不是服务注册中心。服务注册和发现通常由 Nacos 等组件提供。
* Feign 的 `Client` 和 Spring Cloud LoadBalancer 分工不同：一个发请求，一个选择服务实例。
* OpenFeign 默认不会自动重试；不要把 Feign 原生默认行为套用到 Spring Cloud OpenFeign。
* `fallback` 不等于所有异常都应该返回默认成功结果；应区分熔断降级和业务错误处理。
* `@FeignClient` 的不同配置最好使用独立 `contextId`，尤其是同一服务需要多套客户端策略时。
* Spring Cloud OpenFeign 项目已被上游标记为 feature-complete，当前重点是修复和维护。新项目选型时应关注官方迁移建议，并结合 Spring Cloud Alibaba 所使用的 Spring Cloud 版本做兼容性评估。

### 小结

OpenFeign 的关键组件可以按职责理解：`Contract` 解析接口声明，`Encoder` 和 `Decoder` 处理数据转换，`Client` 负责 HTTP 传输，LoadBalancer 选择服务实例，拦截器和日志组件处理横切请求行为，`Retryer`、`ErrorDecoder` 与 CircuitBreaker 处理失败路径，`Capability` 扩展装配过程，Micrometer 提供观测能力。

掌握这些组件的边界后，排查问题时就可以沿调用链定位：接口映射是否正确、参数是否编码正确、服务实例是否选对、底层 HTTP 客户端是否按预期工作，以及异常或超时最终由哪一层处理。
