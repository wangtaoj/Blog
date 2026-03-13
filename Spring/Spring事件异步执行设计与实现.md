### 背景

默认情况下，Spring的事件监听是同步执行的，也就是`publishEvent`方法来触发`ApplicationListener`执行的。当然了也可以配置成异步去执行，但是这是全局的，所有的事件都会变成异步执行了(需要自己配置一个`ApplicationEventMulticaster`并给它设置一个线程池)。

### 设计与实现

AsyncEvent.java

```java
/**
 * 所有异步事件的基类
 */
public abstract class AsyncEvent<T> extends ApplicationEvent {
    @Serial
    private static final long serialVersionUID = -3876435908556202255L;

    private final T payload;

    public AsyncEvent(T payload, Object source) {
        super(source);
        this.payload = payload;
    }

    public T getPayload() {
        return payload;
    }
}
```

AsyncEventListener.java

```java
/**
 * 监听器, 将事件分发到线程池中执行
 */
package com.wangtao.springboot3.event;

import com.wangtao.springboot3.event.handler.AsyncEventHandler;
import com.wangtao.springboot3.executor.CatchExceptionThreadFactory;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.util.LambdaSafe;
import org.springframework.context.event.EventListener;
import org.springframework.core.annotation.AnnotationAwareOrderComparator;
import org.springframework.core.task.TaskRejectedException;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * @author wangtao
 * Created at 2026-03-13
 */
@Component
public class AsyncEventListener implements DisposableBean {

    private final ThreadPoolTaskExecutor taskExecutor;

    private final List<AsyncEventHandler<?>> asyncEventHandlerList;

    private boolean useDefaultTaskExecutor;

    public AsyncEventListener(@Qualifier("asyncEventTaskExecutor") ObjectProvider<ThreadPoolTaskExecutor> taskExecutorProvider,
                              List<AsyncEventHandler<?>> asyncEventHandlerList) {
        this.taskExecutor = taskExecutorProvider.getIfAvailable(() -> {
            this.useDefaultTaskExecutor = true;
            return this.defaultTaskExecutor();
        });
        this.asyncEventHandlerList = asyncEventHandlerList;
        // 根据@Order或者Ordered接口排序
        AnnotationAwareOrderComparator.sort(this.asyncEventHandlerList);
    }

    @EventListener
    public void onAsyncEvent(AsyncEvent<?> asyncEvent) {
        try {
            this.taskExecutor.execute(() -> dispatchAsyncEvent(asyncEvent));
        } catch (TaskRejectedException e) {
            // 任务提交失败, 做补偿
        }
    }

    @SuppressWarnings("unchecked")
    private void dispatchAsyncEvent(AsyncEvent<?> asyncEvent) {
        // 会根据实际的asyncEvent实例来判断交给哪一个handler执行
        LambdaSafe.callbacks(AsyncEventHandler.class, asyncEventHandlerList, asyncEvent)
            .invoke(asyncEventHandler -> asyncEventHandler.handle(asyncEvent));
    }

    @Override
    public void destroy()  {
        if (this.useDefaultTaskExecutor) {
            this.taskExecutor.destroy();
        }
    }

    private ThreadPoolTaskExecutor defaultTaskExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(4);
        taskExecutor.setMaxPoolSize(8);
        taskExecutor.setThreadFactory(new CatchExceptionThreadFactory("async-event"));
        taskExecutor.setQueueCapacity(1000);
        taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        taskExecutor.setAwaitTerminationSeconds(20);
        taskExecutor.initialize();
        return taskExecutor;
    }
}

```

AsyncEventHandler.java

```java
/**
 * 真正执行事件的接口
 */
public interface AsyncEventHandler<T extends AsyncEvent<?>> {

    void handle(T asyncEvent);
}
```

### 使用与测试

```java
public class LikeAsyncEvent extends AsyncEvent<Integer> {

    @Serial
    private static final long serialVersionUID = 1761508846850771470L;

    public LikeAsyncEvent(Integer payload, Object source) {
        super(payload, source);
    }
}
```

```java
public class CommentAsyncEvent extends AsyncEvent<String> {

    @Serial
    private static final long serialVersionUID = 1761508846850771470L;

    public CommentAsyncEvent(String payload, Object source) {
        super(payload, source);
    }
}
```

```java
@Slf4j
@Component
public class LikeAsyncEventHandler implements AsyncEventHandler<LikeAsyncEvent> {

    @Override
    public void handle(LikeAsyncEvent asyncEvent) {
        log.info("======handle likeAsyncEvent, data: {}", asyncEvent.getPayload());
    }
}
```

```java
@Slf4j
@Component
public class CommentAsyncEventHandler implements AsyncEventHandler<CommentAsyncEvent> {

    @Override
    public void handle(CommentAsyncEvent asyncEvent) {
        log.info("======handle commentAsyncEvent, data: {}", asyncEvent.getPayload());
    }
}
```

```java
@Test
public void asyncEventTest() {
    applicationContext.publishEvent(new LikeAsyncEvent(1, this));
}
```

