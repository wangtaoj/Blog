### 简要介绍

Brave 是 Zipkin 官方的 Java 分布式追踪库，是 Sleuth 的底层实现。

Brave 主要负责三件事：

1. **创建和管理 Trace / Span**
2. **上下文传播（线程 / 进程 / RPC）**
3. **把 Span 上报给后端（Zipkin 等）**

Spring Cloud Sleuth 自动整合了这些能力，而不需接触这些底层API。

### 概念

#### Trace（调用链）

一次完整请求 = 一个 Trace

用户请求 -》Service A -》Service B -》Service C

这整条链路就是 **一个 Trace**

traceId：整条链路唯一 ID，在所有服务中 **保持不变**

#### Span

**每一段调用 = 一个 Span**，**Trace**就是由众多的**Span**组合而成。

**Span**可以是一个Http调用、RPC调用、亦或者是一次DB调用、MQ消息发送等等。

**Span**可以有多个子**Span**，从而形成一个链路。

每个 **Span** 有：

* traceId

- spanId
- parentSpanId
- 开始时间 / 结束时间
- 标签（tag）

**Span本身不存traceId、spanId、parentSpanId，这些属性都在TraceContext中，每一个Span都会有一个与之关联的TraceContext**

traceId：整条请求唯一

spanId：当前这一步的 ID

#### TraceContext

整个链路追踪的上下文，当在当前span创建子span时，会复制当前上下文共有的东西来创建一个**新的TraceContext**。

比如traceId，extra(存储额外的业务数据)，spanId作为子span的parentSpanId。

**整个链路追踪的本质就是TraceContext的传播，TraceContext默认存储在ThreadLocal中，同一线程创建子Span可直接获取到，同一进程不同线程，则需要传递TraceContext到子线程中，不同进程传播则需要将TraceContext序列化进行发送，比如Http调用，就是把值通过多个请求头发送。**

### API使用

#### Tracing

可以理解成一个工厂类，可以全局唯一，用于配置和创建**Tracer**

```java
/**
 * 获取Tracer，每一次调用都是同一个Tracer对象
 */
public Tracer tracer();

/**
 * 获取CurrentTraceContext，用于定义如何存储TraceContext, 默认是ThreadLocal
 */
public CurrentTraceContext currentTraceContext();

/**
 * 一个好用的静态方法，用于获取最近一次创建的Tracing
 * 创建Tracing时会把它记录到CURRENT中
 */
public static Tracing current() {
    return CURRENT.get();
}

/**
 * 一个好用的静态方法, 用于获取当前的Tracer对象
 */
public static Tracer currentTracer() {
    Tracing tracing = current();
    return tracing != null ? tracing.tracer() : null;
}
```

#### Tracer

分布式链路的顶层API，用于创建`Span`，可全局唯一。

```java
/**
 * 创建一个root span，即链路的起点，全新的traceId，没有parentSpanId
 */
public Span newTrace();

/**
 * 创建一个子span，会复制parent的一些属性traceId, extra等, parentSpanId等于parent的spanId
 */
public Span newChild(TraceContext parent);

/**
 * 一个好用的API，正常会直接调用nextSpan，而不会直接使用newTrace或者newChild方法
 * 它会判断当前有没有TraceContext，没有则调用newTrace创建一个root span
 * 否则会把当前获取的traceContext当做参数调用newChild创建一个child span
 */
public Span nextSpan();

/**
 * 获取当前span，是根据当前的TraceContext而创建出来的，因此每次调用返回的是一个新对象
 */
public Span currentSpan();

/**
 * 将Span中的TraceContext进行存储，默认是ThreadLocal。如何存储是CurrentTraceContext接口干的事情
 * 其中还会伴随次要动作，比如把traceId放到MDC中，以便日志打印，不过需要配置装饰器
 * SpanInScope的close方法则会清除作用域，将ThreadLocal中的TraceContext还原到上一次的值，上一次没有则就是null
 * 当然同样也会伴随次要动作恢复，互相对应。
 */
public SpanInScope withSpanInScope(@Nullable Span span);

```

#### TraceContext

```java
public long traceId();

/**
 * traceId字符串表示
 */
public String traceIdString();

public long spanId();

public String spanIdString();

public long parentId();

public String parentIdString();

/**
 * 用于存储额外的业务数据, 使用BaggageFeild时, 存储的值就是在这里
 */
public List<Object> extra();
```

#### CurrentTraceContext

存储`TraceContext`的容器，经典实现为`ThreadLocalCurrentTraceContext`，也就是把`TraceContext`存储到`ThreadLocal`中，方便在线程中获取。

```java
/**
 * 获取当前的TraceContext
 */
public abstract @Nullable TraceContext get();

/**
 * 将TraceContext进行存储，Tracer的withSpanInScope方法内部就是调用newScope方法
 * Scope的close方法就是还原newScope动作, SpanInScope的close方法内部就是调用Scope的close方法
 */
public abstract Scope newScope(@Nullable TraceContext context);
```

#### Span

````java
/**
 * 记录开始时间
 */
public abstract Span start();

/**
 * 结束span，计算span的持续时间
 */ 
public abstract void finish();

/**
 * 获取与之关联的TraceContext
 */
public abstract TraceContext context();
````



### 使用示例

#### 基本使用

```java
@Configuration(proxyBeanMethods = false)
public class BraveConfig {

    @Bean
    public Tracing tracing() {
        CurrentTraceContext currentTraceContext = ThreadLocalCurrentTraceContext.newBuilder().build();
        return Tracing.newBuilder()
            .localServiceName("brave-api-demo")
            .currentTraceContext(currentTraceContext)
            .build();
    }

    @Bean
    public Tracer tracer(Tracing tracing) {
        return tracing.tracer();
    }
}
```

```java
@SpringBootTest
public class BraveApiTest {

    @Autowired
    public Tracer tracer;

    @Test
    public void testTraceApi() {
        // 创建根span
        Span rootSpan = tracer.nextSpan().name("root").start();
        // 开启作用域, 将rootSpan中的traceContext对象放到threadLocal中
        // SpanInScope的close会关闭作用域，还原上一次threadLocal中的traceContext值
        try (Tracer.SpanInScope ignored = tracer.withSpanInScope(rootSpan)) {
            // 从ThreadLocal中获取TraceContext
            Assertions.assertNotNull(Tracing.current().currentTraceContext().get());
            // 根据当前绑定的traceContext恢复span，这里会创建一个新的，因此不能使用==判断
            Span currentSpan = tracer.currentSpan();
            Assertions.assertEquals(rootSpan, currentSpan);

            // 创建子span
            Span childSpan = tracer.nextSpan().name("child").start();
            Assertions.assertEquals(rootSpan.context().traceIdString(), childSpan.context().traceIdString());
            Assertions.assertEquals(rootSpan.context().spanIdString(), childSpan.context().parentIdString());
            childSpan.finish();
        } finally {
            rootSpan.finish();
        }
        // 作用域外, 无法获取到绑定到ThreadLocal中的TraceContext对象
        Assertions.assertNull(Tracing.current().currentTraceContext().get());
        Span currentSpan = tracer.currentSpan();
        Assertions.assertNull(currentSpan);
    }
}
```

更简洁的API

```java
/*
 * 创建span，并绑定traceContext对象到ThreadLocal中
 * finish方法还原ThreadLocal中的traceContext
 *
 * 内部自动绑定, 并将还原动作赋予到finish方法中，没有start方法，内部自动start
 * 根据当前有没有TraceContext来创建root span还是child span
 */
ScopedSpan root = tracer.startScopedSpan("root");
// doSomething() 
root.finish();
```

#### ScopeDecorator

有时候想要在绑定TraceContext到ThreadLocal中时，发生一些附带的动作，就可以使用ScopeDecorator。比如把traceId、spanId放到MDC中，供日志系统打印。

```java
@Configuration(proxyBeanMethods = false)
public class BraveConfig {

    @Bean
    public Tracing tracing() {
        // MDCScopeDecorator默认添加了BaggageFields.TRACE_ID、BaggageFields.SPAN_ID，
        // 即会把traceId、spanId到MDC中
        // 可通过add方法添加别的数据, 比如parentId, 或者自定义的数据
        CurrentTraceContext.ScopeDecorator scopeDecorator = MDCScopeDecorator.newBuilder()
            .add(CorrelationScopeConfig.SingleCorrelationField.create(BaggageFields.PARENT_ID))
            .build();
        CurrentTraceContext currentTraceContext = ThreadLocalCurrentTraceContext.newBuilder()
            .addScopeDecorator(scopeDecorator)
            .build();
        return Tracing.newBuilder()
            .localServiceName("brave-api-demo")
            .currentTraceContext(currentTraceContext)
            .build();
    }

    @Bean
    public Tracer tracer(Tracing tracing) {
        return tracing.tracer();
    }
}
```

```java
@Test
public void testMDCAction() {
    String oldTraceId = "123";
    MDC.put(BaggageFields.TRACE_ID.name(), oldTraceId);
    // 创建span并开启作用域, 绑定TraceContext
    ScopedSpan root = tracer.startScopedSpan("root");
    TraceContext context = root.context();
    Assertions.assertEquals(MDC.get(BaggageFields.TRACE_ID.name()), context.traceIdString());
    Assertions.assertEquals(MDC.get(BaggageFields.SPAN_ID.name()), context.spanIdString());
    Assertions.assertEquals(MDC.get(BaggageFields.PARENT_ID.name()), context.parentIdString());
    root.finish();
    // 作用域结束后MDC中的值会还原
    Assertions.assertEquals(oldTraceId, MDC.get(BaggageFields.TRACE_ID.name()));
}
```

#### BaggageField

baggage意为行李，在brave中是用来传递业务字段(额外数据)。

BaggageField.create()方法创建的BaggageField保存的数据存储在TraceContext的extra字段里，默认配置下创建TraceContext实例时extra是一个空对象，需要增加propagationFactory配置。propagationFactory会根据配置来初始化extra。

```java
@Configuration(proxyBeanMethods = false)
public class BraveConfig {

    @Bean
    public Tracing tracing() {
        // MDCScopeDecorator默认添加了BaggageFields.TRACE_ID、BaggageFields.SPAN_ID
        // 即会把traceId、spanId到MDC中
        // 可通过add方法添加别的数据, 比如parentId, 或者自定义的数据
        CurrentTraceContext.ScopeDecorator scopeDecorator = MDCScopeDecorator.newBuilder()
            .add(CorrelationScopeConfig.SingleCorrelationField.create(BaggageFields.PARENT_ID))
            .build();
        CurrentTraceContext currentTraceContext = ThreadLocalCurrentTraceContext.newBuilder()
            .addScopeDecorator(scopeDecorator)
            .build();
        BaggageField userIdField = BaggageField.create("userId");
        return Tracing.newBuilder()
            .localServiceName("brave-api-demo")
            .currentTraceContext(currentTraceContext)
            .propagationFactory(
                BaggagePropagation.newFactoryBuilder(B3Propagation.FACTORY)
                    .add(BaggagePropagationConfig.SingleBaggageField.remote(userIdField))
                    .build()
            )
            .build();
    }

    @Bean
    public Tracer tracer(Tracing tracing) {
        return tracing.tracer();
    }
}
```

BaggagePropagationConfig.SingleBaggageField.remote()会把数据传递到进程外，比如A服务调用B服务，会把这个字段数据通过请求头发送到B。

BaggagePropagationConfig.SingleBaggageField.local()，进程内部传递。

```java
@Test
public void testBaggageField() {
    ScopedSpan root = tracer.startScopedSpan("root");
    // 作用域内可直接通过静态方法获取, 本质上从traceContext的extra中获取
    BaggageField userIdField = BaggageField.getByName("userId");
    // 初始值为null
    Assertions.assertNull(userIdField.getValue());
    // 更新值, 如果当前没有traceContext, 不会有任何动作, 返回false
    Assertions.assertTrue(userIdField.updateValue("0000000"));

    // 创建子span时，也会复制parent traceContext的extra值，修改不会影响父的值
    ScopedSpan child = tracer.startScopedSpan("child");
    BaggageField userIdFieldChild = BaggageField.getByName("userId");
    Assertions.assertEquals("0000000", userIdFieldChild.getValue());
    // 修改
    Assertions.assertTrue(userIdField.updateValue("0000001"));
    // 结束child
    child.finish();
    userIdField = BaggageField.getByName("userId");
    // 依然还是原来的值
    Assertions.assertEquals("0000000", userIdField.getValue());
    // 结束root
    root.finish();

    // 当前无TraceContext, 更新无效果, 获取无值
    BaggageField userName = BaggageField.create("userName");
    Assertions.assertFalse(userName.updateValue("zhangsan"));
    Assertions.assertNull(userName.getValue());
}
```

BaggageField.create()创建的数据必须在作用域中使用，否则无效。