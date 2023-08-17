### 背景

当我们的数据发生变化时，有很多别的业务逻辑需要去做，那么很适合使用事件监听来解耦合。比如目前做过的一个接口，会去修改指令的状态，修改完之后，需要调用持仓、额度等接口，那么每次有新增逻辑都需要来改我的这个接口，这很不方便，我完全可以修改完了之后，直接发布一个事件，让别的模块来监听这个事件，来完成他们的逻辑。

### 使用

使用起来非常简单，基本上就是定义一个event、listener即可。

**当发布事件时，会遍历所有的Listener，挑出符合当前发布事件的Listener，然后执行Listener中的逻辑。**

**编写的Listener需要注册到Spring容器中。**

第一步，声明一个事件

```java
public class IstrnDataStsEvent extends ApplicationEvent {

    /**
     * 数据, 简单起见只使用一个状态
     */
    private final String dataSts;

    public IstrnDataStsEvent(Object source, String dataSts) {
        super(source);
        this.dataSts = dataSts;
    }

    public String getDataSts() {
        return dataSts;
    }
}
```

第二步，监听这个事件，执行自己的逻辑

```java
/**
 * 持仓
 */
@Slf4j
@Component
public class PositionIstrnDataStsListener implements ApplicationListener<IstrnDataStsEvent> {

    @Override
    public void onApplicationEvent(IstrnDataStsEvent event) {
        log.info("position exec logic, dataSts: {}", event.getDataSts());
    }
}
```

```java
/**
 * 额度
 */
@Slf4j
@Component
public class QuotaIstrnDataStsListener implements ApplicationListener<IstrnDataStsEvent> {

    @Override
    public void onApplicationEvent(IstrnDataStsEvent event) {
        log.info("quota exec logic, dataSts: {}", event.getDataSts());
    }
}
```

或者使用`@EventListener`注解方式

```java
@Slf4j
@Component
public class IstrnDataStsListener {

    /**
     * 通过方法参数来决定监听哪个事件
     * 方法参数个数只能是0或者1
     */
    @EventListener
    public void handlePostion(IstrnDataStsEvent event) {
        log.info("position exec logic, dataSts: {}", event.getDataSts());
    }

    /**
     * 通过注解中的classes参数来决定监听哪个事件
     * 使用此方式，如果方法里不需要获取参数时, 可以使用无参方法
     */
    @EventListener(classes = {IstrnDataStsEvent.class})
    public void handleQuota() {
        log.info("=========handleQuota========");
    }
}
```

第三步，发布该事件

```java
@Service
public class IstrnDataStsChangeService implements ApplicationContextAware {

    private ApplicationContext ac;

    public void changeDataSts(String dataSts) {
        ac.publishEvent(new IstrnDataStsEvent(this, dataSts));
    }

    @Override
    public void setApplicationContext(@NonNull ApplicationContext applicationContext) throws BeansException {
        this.ac = applicationContext;
    }
}
```

### 异步执行

默认情况下，监听逻辑是同步顺序执行的，当然Spring也提供异步执行的方式。不过这需要自己注册一个`ApplicationEventMulticaster`，Spring就是使用该接口来注册监听以及发布事件的。

`ApplicationEventMulticaster`的创建在`AbstractApplicationContext.refresh()`方法中完成，里面会调用`initApplicationEventMulticaster`方法。

```java
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 从容器中获取
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    else {
        // 容器中没有，则自己创建
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isTraceEnabled()) {
            logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                    "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
        }
    }
}
```

从初始方法可知，只需要我们自己创建一个`SimpleApplicationEventMulticaster`放到容器中，然后为它设置一个线程池。

```java
@Configuration
public class EventConfig {

    /**
     * 注意bean的名称必须为这个
     */
    @Bean(name = AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME)
    public ApplicationEventMulticaster applicationEventMulticaster(BeanFactory beanFactory) {
        SimpleApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        // 设置线程池, 用于异步执行监听逻辑(线程池参数视情况而定)
        eventMulticaster.setTaskExecutor(Executors.newFixedThreadPool(10));
        return eventMulticaster;
    }
}
```

### 通用事件PayloadApplicationEvent

该类声明了一个泛型，因此可以存任意类型的数据，不过我认为还是自己创建一个专门的事件比较好，这里简单提下，是因为Spring可以通过泛型来区分事件。举个例子

```java
/**
 * 监听PayloadApplicationEvent<Integer>
 */
public class IntegerListener implements ApplicationListener<PayloadApplicationEvent<Integer>> {
    
    @Override
    public void onApplicationEvent(PayloadApplicationEvent<Integer> event) {
        
    }
}
```

```java
/**
 * 监听PayloadApplicationEvent<String>
 */
public class StringListener implements ApplicationListener<PayloadApplicationEvent<String>> {

    @Override
    public void onApplicationEvent(PayloadApplicationEvent<String> event) {

    }
}
```

```java
// 发布事件
ac.publishEvent(new PayloadApplicationEvent<>(this, "string"));
```

在该例子中，**虽然Java中由于泛型擦除`PayloadApplicationEvent<Integer>`和`PayloadApplicationEvent<String>`是同一个类，但是只会触发`StringListener`**。因为Spring内部会去解析Listener的`ResolvableType`，并不是简单比较类型。