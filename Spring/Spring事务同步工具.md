Spring的事务提供了一套事务同步的机制，暴露一些钩子给用户来执行自己的逻辑。基于此封装了事务钩子工具类。有时候可能我们想当前方法事务提交之后执行一些逻辑，比如发送消息到MQ中，那么可以很优雅的使用该工具类来实现这个目的，而不用将发生消息到MQ这段逻辑放到事物方法外面。

```java
public final class TransactionUtils {

    private TransactionUtils() {}

    /**
     * 事务提交之后执行
     * @param action 动作
     */
    public static void executeAfterCommit(Runnable action) {
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    action.run();
                }
            });
        }
    }

    /**
     * 事务回滚之后执行
     * @param action 动作
     */
    public static void executeAfterRollback(Runnable action) {
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCompletion(int status) {
                    if (TransactionSynchronization.STATUS_ROLLED_BACK == status) {
                        action.run();
                    }
                }
            });
        }
    }

    /**
     * 事务完成后执行, 回滚或者提交都有可能
     * 根据status来判断
     * @param action 动作
     */
    public static void executeAfterCompletion(Consumer<Integer> action) {
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCompletion(int status) {
                    action.accept(status);
                }
            });
        }
    }
}
```

不使用该工具类写法

```java
@Service
public class BizService {
    
    @Transactional
    public void execute() {
        // doSomething
    }
}

@Service
public class AnotherService {
    
    @Autowired
    private BizService bizService;

    public void doAction() {
        // 事务方法
        bizService.execute();
        // 发送消息
        sendMq();
    }
}
```

使用工具类后

```java
@Service
public class BizService {

    @Transactional
    public void execute() {
        // doSomething
        // 事务提交后执行逻辑
        TransactionUtils.executeAfterCommit(() -> {
            // 发送消息
            sendMq();
        });
    }
}
```