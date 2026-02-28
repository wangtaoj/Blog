### 基本介绍

在Spring容器关闭时(调用close方法)会自动销毁bean，这样使得一些bean可以释放自己的资源，比如各种连接池。

实现方式主要有如下几种

* 实现Lifecycle(SmartLifecycle)接口的stop方法
* 实现DisposableBean接口destroy方法
* @Bean注解的destroyMethod属性
* 使用@PreDestroy注解一个销毁方法，在解析bean定义时，将该注解标注的方法名作为BeanDefinition中的destroyMethodName
* 实现Closeable或者AutoCloseable接口
* 实现ApplicationListerner，这个只是为了从监听器列表中移除这个bean，毕竟都要销毁了，肯定是不再需要监听事件了

上述除了Lifecycle接口是单独一个机制外，其它的都会包装成DisposableBeanAdapter(它也实现了DisposableBean)，用来统一执行各种方式指定的销毁方法。在bean初始化后注册到DefaultSingletonBeanRegistry的disposableBeans字段(就是一个Map)中。

其中

* Lifecycle在DisposableBeanAdapter之前执行。
* SmartLifecycle能指定顺序执行，而DisposableBeanAdapter方式无法灵活指定顺序，只能靠bean的依赖关系来干预执行顺序，比如A依赖于B，那么初始化时B会先初始化完成，A后初始化完成，销毁时就要反着来，先销毁A再销毁B。

AbstractApplicationContext.java

```java
protected void doClose() {
    ...省略
    if (this.lifecycleProcessor != null) {
        try {
            this.lifecycleProcessor.onClose();
        }
        catch (Throwable ex) {
            logger.warn("Exception thrown from LifecycleProcessor on context close", ex);
        }
    }

    // Destroy all cached singletons in the context's BeanFactory.
    destroyBeans();
    ...省略
}
```

destroyBeans方法内会触发disposableBeans这个map中所有的bean的销毁，同时会从singletonObjects(缓存单例bean的Map)移除。

### 一次优雅停机改造踩的坑

应用部署在k8s中，pod关闭时会先发送SIGNTERM信号(kill -15 pid)，等待一段时间容器还没停止时再发送SIGNKILL(kill -9 pid)强制终止进程。Spring Boot应用会响应SIGNTERM信号，关闭Spring容器。但是每次日志中都会出现一个错误

```tex
org.springframework.beans.factory.BeanCreationNotAllowedException: Error creating bean with name 'sqlSessionFactoryBean': Singleton bean creation not allowed while singletons of this factory are in destruction (Do not request a bean from a BeanFactory in a destroy method implementation!)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:220) [spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:322) ~[spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:207) ~[spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.event.AbstractApplicationEventMulticaster.retrieveApplicationListeners(AbstractApplicationEventMulticaster.java:247) ~[spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.event.AbstractApplicationEventMulticaster.getApplicationListeners(AbstractApplicationEventMulticaster.java:204) ~[spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:134) ~[spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:404) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:410) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:361) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.doClose(AbstractApplicationContext.java:1013) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.close(AbstractApplicationContext.java:979) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.cloud.context.named.NamedContextFactory.destroy(NamedContextFactory.java:93) [spring-cloud-context-2.2.9.RELEASE.jar:2.2.9.RELEASE]
	at org.springframework.beans.factory.support.DisposableBeanAdapter.destroy(DisposableBeanAdapter.java:199) [spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.destroyBean(DefaultSingletonBeanRegistry.java:587) [spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.destroySingleton(DefaultSingletonBeanRegistry.java:559) [spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.destroySingleton(DefaultListableBeanFactory.java:1092) [spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.destroySingletons(DefaultSingletonBeanRegistry.java:520) [spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.destroySingletons(DefaultListableBeanFactory.java:1085) [spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.destroyBeans(AbstractApplicationContext.java:1061) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.doClose(AbstractApplicationContext.java:1030) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]
	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.doClose(ServletWebServerApplicationContext.java:170) [spring-boot-2.3.12.RELEASE.jar:2.3.12.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext$1.run(AbstractApplicationContext.java:949) [spring-context-5.2.15.RELEASE.jar:5.2.15.RELEASE]

```

错误很简单，就是容器销毁时不能再创建bean。但是引起这个错误的真实原因是错综复杂的。

从错误堆栈分析，是因为doClose方法中会发布一个ContextClosedEvent事件导致的，需要ApplicationListener来执行这个事件，sqlSessionFactoryBean这个bean是Mybatis的SqlSessionFactoryBean这个类，它是实现了ApplicationListener接口的，但是由于它已经销毁了，所以getBean方法中报了上述那个错误。但是问题是发布ContextClosedEvent事件是在销毁bean之前发生的。

因为这个应用是Spring Cloud项目，Open Feign的实现是借助了NamedContextFactory来实现的，会为每一个serviceId创建一个子容器。应用容器是这个子容器的父容器，在应用容器中，**SqlSessionFactoryBean先于NamedContextFactory销毁的，NamedContextFactory销毁时，会关闭所有的子容器，子容器发布ContextClosedEvent事件时，还会给父容器也发布事件，从而需要触发SqlSessionFactoryBean的监听器执行，引起了这个错误**。

要修复这个问题，就是不要注册SqlSessionFactoryBean，而是注册SqlSessionFactory即可，Mybatis的自动配置类就只是注册的SqlSessionFactory。

```java
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>,
    InitializingBean, ApplicationListener<ApplicationEvent> {

    
}
```

可以看到它监听的是ApplicationEvent，所以无论发布什么事件都会触发它执行。在mybatis-spring-boot-starter的2.3.2版本中(适配Spring Boot 2的最后一个版本)修改了实现，对应的讨论修改动机([https://github.com/mybatis/spring/pull/797](https://github.com/mybatis/spring/pull/797))

```java
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>,
    InitializingBean, ApplicationListener<ContextRefreshedEvent> {

    
}
```

这个版本只监听ContextRefreshedEvent事件，当发布ContextClosedEvent事件时不会触发SqlSessionFactoryBean执行，因此就不会触发那个错误了。

但是这个问题产生的最根本原因是**Spring对于FactoryBean作为ApplicationListener时，支持的不够友好，因为在ApplicationListener销毁时，是会从监听器列表中移除的，但是呢对于FactoryBean有一个bug，注册时获取的beanName是带有&前缀的，也就是说完整的beanName是&sqlSessionFactoryBean，但是在disposableBeans这个销毁Map时又是sqlSessionFactoryBean，导致销毁时没有删除掉。**

**注册监听器时代码:**

AbstractApplicationContext.java

```java
protected void registerListeners() {
    // Register statically specified listeners first.
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    // 对于FactoryBean，会自动拼接一个&前缀, beanDefinition map中没有&前缀
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }
}
```

题外话，监听器注册时会注册两次，一次是beanName，另外一次是bean本身，是通过ApplicationListenerDetector来实现的，这是一个BeanPostProcessor，在postProcessAfterInitialization钩子中注册的。监听器执行时会去重。

**注册DisposableBean代码:**

AbstractAutowireCapableBeanFactory.java

doCreateBean -> registerDisposableBeanIfNecessary

这里的beanName是不带&的，因为bean定义里是没有&的

**销毁bean移除监听器代码:**

DisposableBeanAdapter.java

```java
public void destroy() {
    if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
        // 其中有一个ApplicationListenerDetector
        for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
            processor.postProcessBeforeDestruction(this.bean, this.beanName);
        }
    }

    if (this.invokeDisposableBean) {
        if (logger.isTraceEnabled()) {
            logger.trace("Invoking destroy() on bean with name '" + this.beanName + "'");
        }
        try {
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((DisposableBean) this.bean).destroy();
                    return null;
                }, this.acc);
            }
            else {
                ((DisposableBean) this.bean).destroy();
            }
        }
        catch (Throwable ex) {
            String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";
            if (logger.isDebugEnabled()) {
                logger.warn(msg, ex);
            }
            else {
                logger.warn(msg + ": " + ex);
            }
        }
    }

    if (this.destroyMethod != null) {
        invokeCustomDestroyMethod(this.destroyMethod);
    }
    else if (this.destroyMethodName != null) {
        Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);
        if (methodToInvoke != null) {
            invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));
        }
    }
}
```

ApplicationListenerDetector.java

```java
public void postProcessBeforeDestruction(Object bean, String beanName) {
    if (bean instanceof ApplicationListener) {
        try {
            ApplicationEventMulticaster multicaster = this.applicationContext.getApplicationEventMulticaster();
            // 删除成功
            multicaster.removeApplicationListener((ApplicationListener<?>) bean);
            // 删除失败, 因为内部使用的&前缀注册的, 而beanName没有&前缀
            multicaster.removeApplicationListenerBean(beanName);
        }
        catch (IllegalStateException ex) {
            // ApplicationEventMulticaster not initialized yet - no need to remove a listener
        }
    }
}
```

