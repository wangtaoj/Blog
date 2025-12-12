`ApplicationContext`会在refresh方法中自动初始化所有的单例bean，但是有时候某些bean可能初始比较耗时又或者某种原因想要一个bean在启动时不初始化，而是等到真正使用这个bean时才完成初始化，那么就可以用到这个机制。

在Spring中可以使用@Lazy注解来达到这个目的。

@Lazy注解可以用在类、方法、构造方法、字段、方法参数上。主要是两种使用方法

其一：用于bean的定义，也就是标注在类上或者@Bean标注的方法上，Spring在初始化容器时会自动跳过被@Lazy标注的Bean

```java
@Lazy
@Component
public class LazyBean {
    
}

@Configuration(proxyBeanMethods = false)
public class Config {
    
    @Lazy
    @Bean
    public LazyBean lazyBean() {
        return new LazyBean();
    }
}
```

初始化所有单例Bean的方法如下

DefaultListableBeanFactory.java

```java
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 单例并且非懒加载
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof SmartFactoryBean<?> smartFactoryBean && smartFactoryBean.isEagerInit()) {
                    getBean(beanName);
                }
            }
            else {
                getBean(beanName);
            }
        }
    }
    省略下面代码
}
```

其二：用于字段注入(包括字段注入，构造方法注入，setter方法注入，以及@Bean方法的参数上)，Spring在注入字段时，如果发现被注入的字段有被@Lazy注解标记，则会注入一个代理对象。这里的判断不会管注入的Bean本身是否有@Lazy标记

```java
@Component
public class CommonBean {
    
    /**
     * 无论LazyBean本身定义是是否有被@Lazy标记，都是注入的代理对象
     */
    @Lazy
    @Autowired
    private LazyBean lazyBean;
}
```

最后注意的点

**一个标记为Lazy的bean如果没有被其他bean依赖，那么只会在真正使用时才会进行初始化，也就是调用它的方法(toString也会，getClass方法不会，因为没有代理getClass方法)。如果有被其它bean依赖注入，除非注入时也标记了@Lazy，否则也会简介导致这个Lazy bean进行初始化。**

**如果手动使用容器的getBean方法去获取这个Lazy Bean，也会导致其初始化，因为getBean方法本身没有是否懒加载的判断。**

**依赖注入时注入的代理对象不会注册到容器中，当调用目标对象方法时，底层会调用getBean方法获取到目标对象，此时把目标对象注册到容器中，当然注册是getBean方法内部行为。**

看以下示例代码

```java
@Lazy
@Component
public class LazyBean {

    public LazyBean() {
        System.out.println("=======lazyBean init=======");
    }
}

public class CommonBean {

    private final LazyBean lazyBean;
	
    // 这里没有使用@Lazy来标注
    public CommonBean(LazyBean lazyBean) {
        this.lazyBean = lazyBean;
        System.out.println("======commonBean init=======");
    }

    public LazyBean getLazyBean() {
        return lazyBean;
    }
}

// 测试代码
@Test
public void lazyTest() {
    System.out.println("=========lazy test=========");
    System.out.println(commonBean.getLazyBean().getClass());

    Map<String, LazyBean> lazyBeanMap = applicationContext.getBeansOfType(LazyBean.class);
    lazyBeanMap.forEach((k, v) -> {
        System.out.println("beanName: " + k + ", value: " + v.getClass());
    });
}
	
```

输出结果:

=======lazyBean init=======

======commonBean init=======

=========lazy test=========

class com.wangtao.springboot3.lazy.LazyBean

beanName: lazyBean, value: class com.wangtao.springboot3.lazy.LazyBean

可以看到即便LazyBean已经被@Lazy标注，但是由于被CommonBean依赖注入，而导致直接初始化了。



再看使用@Lazy注解来注入

```java
@Component
public class CommonBean {

    private final LazyBean lazyBean;

    // 仅仅在这里加了注解
    public CommonBean(@Lazy LazyBean lazyBean) {
        this.lazyBean = lazyBean;
        System.out.println("======commonBean init=======");
    }

    public LazyBean getLazyBean() {
        return lazyBean;
    }
}
```

输出结果

======commonBean init=======

=========lazy test=========

class com.wangtao.springboot3.lazy.LazyBean$$SpringCGLIB$$0

=======lazyBean init=======

beanName: lazyBean, value: class com.wangtao.springboot3.lazy.LazyBean

可以看到在手动调用getBeansOfType方法后才执行了初始化
