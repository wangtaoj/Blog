### 基本概念

内层事务发生异常时, 外部事务即便try catch了内层事务，也会使得外层事务回滚，同时会抛出`UnexpectedRollbackException`。

因为在外层事务角度看，它本来是要正常提交事务的，但是由于参与的内层事务出现异常，而它们又同属于一个真实的物理事务，Spring默认采取将事务回滚，同时抛出一个不期望的回滚异常。

这个行为通过`globalRollbackOnParticipationFailure`参数控制，默认为true。即参于事务失败时，则全局回滚。

### 执行流程

代码如下

```java
@RequiredArgsConstructor
public class TxCase2 {

    private final TxCase3 txCase3 ;

    @Transactional
    public void execute() {
        try {
            txCase3.execute();
        } catch (RuntimeException e) {
            log.error(e.getMessage(), e);
        }
        log.info("====txCase2 execute====");
    }
}


public class TxCase3 {

    @Transactional
    public void execute() {
        log.info("====txCase3 execute====");
        throw new RuntimeException("模拟异常");
    }
}
```

执行结果

```tcl
====txCase3 execute====	
java.lang.RuntimeException: 模拟异常
	at com.wangtao.springboottest.TransactionTest$TxCase3.execute(TransactionTest.java:119)
====txCase2 execute====
org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only
```

执行流程

```tex
txCase2 begin
txCase3 begin(只是参与到txCase2的事务)
txCase3 执行异常
txCase3 rollback(发现不是一个新开的事务, 根据globalRollbackOnParticipationFailure来决定是否标记全局回滚)
txCase2 执行结束(因为try catch了，整个方法并没有发生异常)
txCase2 commit(发现标记了全局回滚, 走回滚动作)
txCase2 rollback(unexpected=true) （此时unexpected为true）
如果unexpected=true，抛出UnexpectedRollbackException

```

### failEarlyOnGlobalRollbackOnly作用

此参数的含义是，如果是全局回滚导致进入的rollback(unexpected参数=true)，要不要将upexpected参数重置为false，不重置则会提前抛出`UnexpectedRollbackException`，只有在3层事务时才有效果，默认为false，当此参数为false时，会将upexpected参数重置为false，就不会在中间的参与事务中抛出`UnexpectedRollbackException`了。

比如事务3参与事务2，事务2参与事务1，事务2中的方法捕获了事务3方法中的异常。

执行流程如下

事务3方法发生异常 -> 事务3正常发生回滚(unexpected=false) -> 发现自己不是新事务 -> 标记全局回滚 -> unexpected=false不会抛出`UnexpectedRollbackException`

事务2由于try catch了 -> 事务2方法正常执行完毕 -> 事务2commit -> 发现全局回滚标记 -> 转而触发rollback(unexpected=true) -> 发现自己不是新事务 -> 重新标记全局回滚 -> 根据`failEarlyOnGlobalRollbackOnly`是否将unexpected重置为false -> 若不重置，则抛出`UnexpectedRollbackException` -> 若重置则rollback正常结束

若事务2抛出`UnexpectedRollbackException` -> 则事务1方法执行时遭遇异常结束 -> 发生回滚(unexpected=false)  -> 发现自己是新事务 -> 回滚整个事务

若事务2不抛出`UnexpectedRollbackException` -> 则事务1方法正常执行结束 -> 事务1commit -> 发现全局回滚标记 -> 转而触发rollback(unexpected=true) -> 发现自己是新事务 -> 回滚整个事务 -> unexpected=true(抛出`UnexpectedRollbackException`)

```java
@Slf4j
@SpringBootTest
public class TransactionTest {

    @Autowired
    private TxCase1 txCase1;

    @Test
    public void testGlobalRollback() {
        txCase1.execute();
    }

    @TestConfiguration
    public static class TransactionConfig {

        @Bean
        public MyPlatformTransactionManagerCustomizer myPlatformTransactionManagerCustomizer() {
            return new MyPlatformTransactionManagerCustomizer();
        }

        @Bean
        public TxCase3 txCase3() {
            return new TxCase3();
        }

        @Bean
        public TxCase2 txCase2(TxCase3 txCase3) {
            return new TxCase2(txCase3);
        }

        @Bean
        public TxCase1 txCase1(TxCase2 txCase2) {
            return new TxCase1(txCase2);
        }
    }

    /**
     * 配置事务管理器属性
     */
    public static class MyPlatformTransactionManagerCustomizer implements PlatformTransactionManagerCustomizer<AbstractPlatformTransactionManager> {

        @Override
        public void customize(AbstractPlatformTransactionManager transactionManager) {
            transactionManager.setFailEarlyOnGlobalRollbackOnly(true);
        }
    }

    @RequiredArgsConstructor
    public static class TxCase1 {

        private final TxCase2 txCase2;

        @Transactional
        public void execute() {
            txCase2.execute();
            log.info("====txCase1 execute====");
        }
    }

    @RequiredArgsConstructor
    public static class TxCase2 {

        private final TxCase3 txCase3 ;

        @Transactional
        public void execute() {
            try {
                txCase3.execute();
            } catch (RuntimeException e) {
                log.error(e.getMessage(), e);
            }
            log.info("====txCase2 execute====");
        }
    }

    public static class TxCase3 {

        @Transactional
        public void execute() {
            log.info("====txCase3 execute====");
            throw new RuntimeException("模拟异常");
        }
    }
}
```

`failEarlyOnGlobalRollbackOnly`设置为false的执行结果(默认情况)

```tex
====txCase3 execute====
java.lang.RuntimeException: 模拟异常
====txCase2 execute====
====txCase1 execute====
```



`failEarlyOnGlobalRollbackOnly`设置为true的执行结果

```tex
 ====txCase3 execute====
 java.lang.RuntimeException: 模拟异常
 ====txCase2 execute====
 org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only
```

可以看到由于`UnexpectedRollbackException`被提前抛出，TxCase1中的execute方法在调用`txCase2.execute()`时会碰到`UnexpectedRollbackException`异常，导致后续代码不会执行。

此时**TxCase3会因为异常而直接进入rollback(不再是先进入commit，然后发现是全局回滚再触发rollback动作)，此时的unexpected参数为false，不会再重复抛出UnexpectedRollbackException异常了。**