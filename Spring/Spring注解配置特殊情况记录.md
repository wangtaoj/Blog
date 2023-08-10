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