### 简要介绍

Lifecycle组件是为了扩展bean的生命周期功能而增加的，它还有一个直接子类为SmartLifecycle接口，该接口能够控制组件生命周期的执行顺序。

下面是接口定义

```java
public interface Lifecycle {

    
    void start();

    
    void stop();

    
    boolean isRunning();

}
```

```java
public interface SmartLifecycle extends Lifecycle, Phased {

   
    int DEFAULT_PHASE = Integer.MAX_VALUE;

   
    default boolean isAutoStartup() {
        return true;
    }

    default void stop(Runnable callback) {
        stop();
        callback.run();
    }

  
    @Override
    default int getPhase() {
        return DEFAULT_PHASE;
    }
}
```

SmartLifecycle和Lifecycle区别如下

* 实现SmartLifecycle接口的bean，在ApplicationContext初始化完成后(所有单例bean都已初始化)，如果isAutoStartup方法返回true，则会自动调用start方法。而Lifecycle接口则需要手动调用ApplicationContext的start方法来触发。
* SmartLifecycle接口可以控制执行顺序，它继承了Phased接口，getPhase方法返回的值越小，start()则越先调用，stop方法越晚执行。
* SmartLifecycle增加了一个带有回调接口的stop，这样组件可以异步停止，回调callback是spring自己传递，就是一个CountDownLatch执行countDown方法，而主线程会执行CountDownLatch的await方法，等待组件stop方法执行完毕，当然这个等待时间由spring.lifecycle.timeout-per-shutdown-phase控制的，默认是30s。该参数不是等待所有的Lifecycle，执行时会根据phase来分组，是指每一个组的Lifecycle最多执行spring.lifecycle.timeout-per-shutdown-phase这么久，超了这个时间就执行下一个组了。
* 另外ApplicationContext执行close方法时，不论是Lifecycle还是SmartLifecycle都会执行，前提是isRunning方法返回true。此时Lifecycle的phase会给0来参与分组排序。

因此只需要使用SmartLifecycle即可。

另外一个细节就是isRunning方法了，spring在调用start方法之前会先判断isRunning方法是不是返回false，调用stop方法之前会判断isRunning方法是不是返回true

实现模板

```java
public class SimpleLifecycle implements SmartLifecycle {

    /**
     * 如果stop方法有异步线程, 则需要使用volatile修饰
     */
    private volatile boolean running;

    @Override
    public void start() {
        running = true;
        // doSomething
    }

    @Override
    public void stop() {
        running = false;
        // doSomething
    }

    @Override
    public boolean isRunning() {
        return running;
    }

    /**
     * 根据需要指定顺序
     */
    @Override
    public int getPhase() {
        return 0;
    }
}
```

### 执行时机源码

#### start生命周期

AbstractApplicationContext类

refresh -> finishRefresh -> getLifecycleProcessor().onRefresh()

触发SmartLifecycle执行的类为LifecycleProcessor

#### stop生命周期

AbstractApplicationContext类

close -> doClose -> lifecycleProcessor.onClose()

### 应用

tomcat容器的启动和关闭就是通过SmartLifecycle来实现的

* WebServerStartStopLifecycle

* WebServerGracefulShutdownLifecycle(开启优雅停机属性才有作用, server.shutdown=graceful)，这样tomcat就会停止处理新的请求，会等待现有请求执行完毕。等待最大时间为spring.lifecycle.timeout-per-shutdown-phase，超过这个时间还没处理完成，WebServerStartStopLifecycle的stop方法就会执行了，会修改WebServerGracefulShutdownLifecycle循环的标记而退出。