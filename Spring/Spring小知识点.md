### @Configuration

从spring-context5.2版本开始，加了一个`proxyBeanMethods`属性

```java
public @interface Configuration {

    
    @AliasFor(annotation = Component.class)
    String value() default "";

    /**
     * 该属性为true时，被@Configuration注解的配置类将会使用CGLIB增强，
     * 这样使得直接调用@Bean注解的方法将会返回同一个实例
     * 如果不需要这个特性，可以设置成false，默认为true
     * @since 5.2
     */
    boolean proxyBeanMethods() default true;

}
```

例子：

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig {
    
}
```

当打印bean的class属性时，为`class com.wangtao.springboottest.config.AppConfig`，即没有被代理。

```java
@Configuration
public class AppConfig {
    
}
```

当打印bean的class属性时，为`class com.wangtao.springboottest.config.AppConfig$$EnhancerBySpringCGLIB$$b73b4e3`，发现被CGLIB代理。

再来看看方法行为

```java
@Configuration
public class AppConfig {

    @Bean
    public A beanA() {
        return new A();
    }

    @Bean
    public B beanB() {
        // 直接调用方法，而不是注入的方式
        A a = beanA();
        return new B(a);
    }

    public static class A {

    }

    public static class B {
        public A a;
        public B(A a) {
            this.a = a;
        }
    }
}
```

会发现容器中的A实例与B实例的成员变量a是同一个对象。当然如果手动设置`proxyBeanMethods = false`，那么就是两个对象了。

最后，增强配置类的逻辑位于`ConfigurationClassPostProcessor`类中

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    int factoryId = System.identityHashCode(beanFactory);
    if (this.factoriesPostProcessed.contains(factoryId)) {
        throw new IllegalStateException(
                "postProcessBeanFactory already called on this post-processor against " + beanFactory);
    }
    this.factoriesPostProcessed.add(factoryId);
    if (!this.registriesPostProcessed.contains(factoryId)) {
        // BeanDefinitionRegistryPostProcessor hook apparently not supported...
        // Simply call processConfigurationClasses lazily at this point then.
        processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
    }
    // 增强配置类
    enhanceConfigurationClasses(beanFactory);
    beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
```

注:
**只有被`@Configuration`注解的配置类并且`proxyBeanMethods=true`时才会被增强，其他情况都不会有该特性，如`@Compoment`注解的类等。**

### @Bean

使用@Bean注解注册的bean不会被当成配置类来解析。

比如以下例子

```java
/**
 * 注意: 该类不要放在Spring Boot扫描的包下面, 因为要使用@Bean的方式注册到容器中
 */
@Configuration
public class AppConfig {

    @Bean
    public A beanA() {
        return new A();
    }
}

/**
 * 注意: 该类需要放在Spring Boot扫描的包下面，作为配置类
 */
@Configuration
public class MainConfig {

    /**
     * 注册AppConfig
     */
    @Bean
    public AppConfig appConfig() {
        return new AppConfig();
    }
}
```

以上例子beanA不会被注入到Spring容器中。如果你需要AppConfig也被作为一个配置类来解析，可以使用`@Import`注解。

```java
@Import(AppConfig.class)
@Configuration
public class MainConfig {

}
```

这么写beanA会被注入到Spring容器中。

### @Import

`@Import`注解主要作用用于导入一个配置类，不过呢，它导入的类分为三种情况。

- 导入的类实现了`ImportSelector`接口，导入的类本身不会被注册到容器，实际注册的类为`selectImports`方法返回的类，注意返回的类也会按照这3种情况递归处理。
- 导入的类实现了`ImportBeanDefinitionRegistrar`接口，导入的类本身不会被注册到容器，实际注册的类为`registerBeanDefinitions`方法中收到注册的类。
- 除上述以外情况，导入的类将会被注册到容器中，并且会作为一个配置类继续解析，从而注册更多的bean到容器中(导入的类本身可以不用@Configuration进行注解)。

例子：

```java
@Import({A.class, B.class})
@Configuration
public class AppConfig {

}

/**
 * 不在Spring Boot包扫描中，通过@Import注解导入
 */
@Configuration
public class A {

}

public class B implements ImportSelector {
    @Override  
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {  
        return new String[] {C.class.getName()};  
    }
}

/**
 * 不在Spring Boot包扫描中，通过B的selectImports方法导入
 */
@Configuration
public class C {

}
```

在上述例子中，容器实际只注册了AppConfig、A、C。因为B是一个`ImportSelector`实例，不会被注册到容器中，不过B通过`selectImports`方法把C注册进来了。

### ObjectProvider

* ObjectProvider在注入时即使没有对应的bean，也不会报错，同样也可搭配`@Qualifier`注解注入指定名字的bean
* getIfAvailable方法在有多个候选bean时会抛出`NoUniqueBeanDefinitionException`异常
* getIfUnique方法在有多个候选bean时会返回null

### DefaultSingletonBeanRegistry

#### dependentBeanMap vs dependenciesForBeanMap

dependentBeanMap：记录的是一个bean被哪些其他bean所依赖，谁依赖了我

dependenciesForBeanMap：记录的是一个bean依赖了哪些其他bean，我依赖了谁

```java
@DependsOn("dependentBean2")
@Component
public class DependentBean1 {

    @Autowired
    private DependentBean3 dependentBean3;
}


@Component
public class DependentBean2 {
}

@Component
public class DependentBean3 {
}

@SpringBootTest
public class SpringBootTestApplicationTests {

    @Autowired
    private ConfigurableApplicationContext applicationContext;

    @Test
    public void contextLoads() {
        /*
         * [dependentBean2, dependentBean3]
         * 常见的注入以及DependsOn
         *
         * 注意: 需要ConfigurableApplicationContext才有getBeanFactory()方法
         */
        String[] res = applicationContext.getBeanFactory().getDependenciesForBean("dependentBean1");
        System.out.println(Arrays.toString(res));
        
        // [dependentBean1]
        res = applicationContext.getBeanFactory().getDependentBeans("dependentBean2");
        System.out.println(Arrays.toString(res));
    }

}
```

### 循环引用依赖

#### 为什么需要使用3个Map？

答：为了保证没有循环依赖的bean在创建代理时不会提前，还是在BeanPostProcessor的postProcessAfterInitialization方法阶段创建。

此时必需要有一个工厂方法的Map，如果发生循环引用，则调用这个工厂方法获取代理对象(循环引用必定导致代理提前创建)。无循环引用，因为放的只是一个工厂方法，不会提前创建。

那为啥需要3个，简单实现确实只需要两个Map即可，一个Map存成品对象，一个Map存工厂方法。但是考虑到复杂的循环引用场景，比如A 依赖于 B，B 依赖于A和C、而C有依赖于A时，假设A先被初始化，那么工厂方法会被B以及C先后两次调用，这样子的话，必须要由创建代理的基础类保证每次调用拿到的是同一个代理对象才行，否则B和C注入的A代理就不一致了。所以需要再加入一个Map来存工厂方法创建出来的半成品。

#### 循环依赖最终对象检查

对象创建完成后，会进行一致性检查.

AbstractAutowireCapableBeanFactory.doCreateBean()

```java
// 打开了允许循环引用开关
if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    // 这里earlySingletonReference不等于null，必定发生了循环引用，工厂方法被调用了才会有值
    if (earlySingletonReference != null) {
        /*
         * bean是原始对象, exposedObject经过了层层代理返回的对象
         * 这里其实即便发生了aop代理，也是相等的。
         * 因为AbstractAutoProxyCreator的postProcessAfterInitialization
         * 方法有做判断，如果发现getEarlyBeanReference方法已经创建代理了，就不会再次创建代理，直接跳过了。
         * 
         * 但是若有别的BeanPostProcessor在没有判断时创建了代理，就会导致exposedObject和bean不是同一个对象了。
         * 走到else分支，就会报错了。
         * 比如@async注解标注的对象，就不是由AbstractAutoProxyCreator创建的代理对象，循环依赖时就会报下面错误。
         */
        if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
        }
        else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                    actualDependentBeans.add(dependentBean);
                }
            }
            if (!actualDependentBeans.isEmpty()) {
                throw new BeanCurrentlyInCreationException(beanName,
                        "Bean with name '" + beanName + "' has been injected into other beans [" +
                        StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                        "] in its raw version as part of a circular reference, but has eventually been " +
                        "wrapped. This means that said other beans do not use the final version of the " +
                        "bean. This is often the result of over-eager type matching - consider using " +
                        "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
            }
        }
    }
}
```

**若对象发生代理行为，并且代理不是通过AbstractAutoProxyCreator(其实是SmartInstantiationAwareBeanPostProcessor接口)创建的，那么是不支持循环引用的**