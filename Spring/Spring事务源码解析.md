### 1. 背景

因为最近在看MyBatis-Spring的源码，但是里面涉及了颇多Spring事务底层API的知识，看完后有点似懂非懂的样子，于是便有了这篇文章。下面的源码分析仅针对于DataSourceTransactionManager这一个具体的事务管理器。当你直接使用JDBC编程或者使用MyBatis时使用的事务管理器就是DataSourceTransactionManager。在看这篇文章前可以先过下前文对事务相关接口以及类的介绍，让心中有一个数，两篇文章搭配来看效果会更好。

### 2. Spring事务API的使用

在分析源码前，先搞一个小例子跑起来，顺带简单介绍下如何使用Spring事务的底层API来管理事务。这里简单放下关键配置以及运行代码。

XML配置

```xml
<!-- 数据库连接池配置(Druid) -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <!-- 数据库驱动 -->
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <!-- 数据库地址 -->
    <property name="url" value="jdbc:mysql://127.0.0.1:3306/test" />
    <!-- 数据库用户名 -->
    <property name="username" value="root"/>
    <!-- 数据库密码 -->
    <property name="password" value="123456"/>
</bean>

<!-- 事务管理器，事务就靠它啦 -->
<bean 
    id="transactionManager"        	  
    class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>

<!-- JdbcTemplate，用来操作数据 -->
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <constructor-arg name="dataSource" ref="dataSource" />
</bean>

<!-- 事务模板，内部封装了事务开启以及提交回滚的操作，只需关心业务代码 -->
<bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
    <property name="transactionManager" ref="transactionManager" />
</bean>
```

测试代码

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(locations = {"classpath:spring-tx.xml"})
public class TransactionTest {
    @Autowired
    private DataSourceTransactionManager transactionManager;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private TransactionTemplate transactionTemplate;
    
    /** 随便一个操作数据库的方法吧 **/
    private Integer findCount(Integer id) {
        final String sql = "SELECT COUNT(*) FROM student";
        return jdbcTemplate.queryForObject(sql, Integer.class);
    }
    
    // 直接使用API来管理事务
    @Test
    public void transactionByAPI() {
        // 使用默认的隔离级别，默认的传播行为(PROPAGATION_REQUIRED)开启一个事务
        // 此方法会从给定的数据源中获取一个连接, 设置自动提交模式为false, 并且将连接绑定到当前线程。
        // JdbcTemplate内部会调用DataSourceUtils.getConnection(database)从线程中获取绑定的连接
        TransactionStatus transactionStatus = transactionManager.getTransaction(
            new DefaultTransactionDefinition());
        try {
            findCount();
            transactionManager.commit(transactionStatus);
        } catch (RuntimeException e) {
            transactionManager.rollback(transactionStatus);
            throw e;
        }
    }
    
    // 使用TransactionTemplate操作事务, execute方法内部封装了事务样板代码, 只需要专注业务.
    @Test
    public void transactionByTemplate() {
        transactionTemplate.execute(new TransactionCallback<Student>() {
            @Override
            public Integer doInTransaction(TransactionStatus status) {
                return findCount();
            }
        });
    }
}
```

### 3. 源码分析

在分析源码前先做一个约定，以免引起误解，Spring为了解决方法调用时每个方法中的事务该如何运行，引进了事务传播行为这一属性。我们有以下几种说法，如果熟悉传播行为的话，下面这些特别容易理解。

* 当外部不存在真实事务，指定的传播行为不新建真实事务时(PROPAGATION_SUPPORTS、PROPAGATION_NOT_SUPPORTED、PROPAGATION_NEVER)，我们说Spring开启了一个**新的空事务**。
* 当外部不存在真实事务，指定的传播行为新建一个真实事务时(PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED)，我们说Spring开启了一个**新的真实事务**。
* 当外部存在一个真实事务，指定的传播行为(PROPAGATION_REQUIRED、PROPAGATION_SUPPORTS，PROPAGATION_MANDATORY)是参与到外部事务时，我们说这是一个**参与事务**。
* 当外部存在一个真实事务，指定的传播行为是PROPAGATION_NESTED，我们说这是一个**嵌套事务**。**嵌套事务是利用JDBC标准的保存点来实现的，依然依赖于外部事务。**
* 当外部存在一个真实事务，指定的传播行为是PROPAGATION_REQUIRES_NEW时，此时会挂起外部事务，开启一个**新的真实事务**，相当于一个独立的真实事务。只有一点影响，**如果这个新的真实事务向外抛出了异常，而被外部事务捕捉，可能会引起外部事务回滚(具体看外部事务如何处理这个异常)。这个新的真实事务完全不受外部事务影响。**
* 当外部存在一个真实事务，指定的传播行为是PROPAGATION_NOT_SUPPORTED时，此时会挂起外部事务，开启一个**新的空事务**，行为与上一条一样，只是一个以事务方式运行，一个以非事务方式运行。

注:

* 因此说开启事务，并不是说就开启了一个真实事务，可能开启了一个空事务，也有可能只是参与到外部事务中运行。**空事务是以非事务方式运行**。
* 注意说明带的**新**字，因为这个事务是新建的才需要释放开启事务时获取的资源(如果有的话，空事务不会获取连接)。**新说明事务是独立的，如果有外部事物的话，不会依赖于外部事务。**
* **只有这个事务是新的真实事务时，事务管理器在commit/rollback后才会清理线程中绑定的连接资源，其实是一个ConnectionHolder对象，并且重置从数据源中拿到的连接，然后归还。当然也只有这个事务是新的真实事务，开启事务时才会从数据源中拿连接，并且绑定到当前线程。**

#### 3.1 开启事务(getTransaction)

```java
// 位于 DataSourceTransactionManager.java
@Override
protected Object doGetTransaction() {
    // 代表一个事务对象
    DataSourceTransactionObject txObject = new DataSourceTransactionObject();
    // nestedTransactionAllowed属性在事务管理器构造方法就初始化为true了
    // 这个值代表支不支持嵌套事务
    txObject.setSavepointAllowed(isNestedTransactionAllowed());
    // 从当前线程获取绑定的资源，如果Spring要开启一个新的真实事务时，该值为null
    // 比方说这次开启的是一个参与事务，那么拿到的就是外部事务绑定的资源了
    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.
        getResource(this.dataSource);
    // 设置值，第二个参数代表是不是一个新的connectionHolder
    // 如果conHolder=null, 第二个参数会在后续被修改成true, 继续往下看吧。
    // 这里也可以得知参与事务与外部事务的事务对象中共享一个connectionHolder
    // 只是会标识出不是一个新的connectionHolder
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}

// 位于 AbstractPlatformTransactionManager.java
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    // 获取一个事务对象，由子类实现(看上面解析)
    Object transaction = doGetTransaction();

    // Cache debug flag to avoid repeated checks.
    boolean debugEnabled = logger.isDebugEnabled();

    // 如果传进来的definition为空，则使用一个默认定义
    if (definition == null) {
        definition = new DefaultTransactionDefinition();
    }

    // 在开启事务前判断下当前有没有真实事务存在来决定走向，下面再说.
    if (isExistingTransaction(transaction)) {
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // 现在外部不存在真实事务，如果传播行为是PROPAGATION_MANDATORY，则抛出异常
    // PROPAGATION_MANDATORY: 强制外部存在一个真实事务，加入到外部事务。否则抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
            "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    // 当外部不存在真实事务时，以下三个传播行为是一样的，都是新建一个新的真实事务
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
             definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
             definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
        }
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException ex) {
            resume(null, suspendedResources);
            throw ex;
        }
        catch (Error err) {
            resume(null, suspendedResources);
            throw err;
        }
    }
    else {
        // 走到这里说明指定的传播行为为PROPAGATION_SUPPORTS、PROPAGATION_NOT_SUPPORTED、
        // PROPAGATION_NEVER，新建一个空事务
        // 尽管是一个空事务，也会根据属性来决定需不需要绑定事务的各个属性到当前线程。
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                        "isolation level will effectively be ignored: " + definition);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
    }
}
```

可以看到getTransaction最终由三个分支走向，我们先看最简单的第三个分支吧(创建一个空事务)

```java
// 判断下隔离级别，如果有指定隔离级别，但是不是在真实的事务环境中运行，会被忽略掉
if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
    logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                "isolation level will effectively be ignored: " + definition);
}
// 需不需要同步此空事务的特征，默认都会同步，即便是空事务
boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
// 返回一个DefaultTransactionStatus
// 可以去看下前文对DefaultTransactionStatus的介绍，对各个属性有详细介绍
// 第二个参数为null, 说明是一个空事务
// 第三个参数为true, 说明这是一个新开的事务
// 第四个参数在默认情况下是true, 但是在下文可能依旧会被修改。
// 第五个参数没啥好说的
// 第六个参数是被挂起的资源，如果在开启事务时有挂起外部事务时，需要将外部事务涉及的属性和连接保存到这里
// 当此事务调用commit或者rollback方法时会将挂起的资源恢复，开启一个空事务，当然是null,不需要恢复。
return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
}

protected final DefaultTransactionStatus prepareTransactionStatus(
    TransactionDefinition definition, Object transaction, boolean newTransaction,
    boolean newSynchronization, boolean debug, Object suspendedResources) {
	// 创建一个DefaultTransactionStatus对象
    DefaultTransactionStatus status = newTransactionStatus(
        definition, transaction, newTransaction, newSynchronization, debug, suspendedResources);
    prepareSynchronization(status, definition);
    return status;
}

protected DefaultTransactionStatus newTransactionStatus(
    TransactionDefinition definition, Object transaction, boolean newTransaction,
    boolean newSynchronization, boolean debug, Object suspendedResources) {
    // 判断当前线程是否已经同步，如果同步过便不再需要同步
    boolean actualNewSynchronization = newSynchronization &&
        !TransactionSynchronizationManager.isSynchronizationActive();
    return new DefaultTransactionStatus(
        transaction, newTransaction, actualNewSynchronization,
        definition.isReadOnly(), debug, suspendedResources);
}

protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
    // 需要同步的话
    if (status.isNewSynchronization()) {
        // 此事务是一个真实事务，设置为true, 否则为null
        TransactionSynchronizationManager.setActualTransactionActive(
         status.hasTransaction());
        // 设置此事务的隔离级别
        TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
            definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT ?
            definition.getIsolationLevel() : null);
        // 设置事务只读
        TransactionSynchronizationManager.setCurrentTransactionReadOnly(
            definition.isReadOnly());
        // 设置此事务的名字，默认为null
        TransactionSynchronizationManager.setCurrentTransactionName(
            definition.getName());
        // 初始化线程里的事务同步，管理的是一组TransactionSynchronization
        // TransactionSynchronization接口定义了一组钩子，在事务各个期间触发
        // 事务管理器内部没有向里面注册过TransactionSynchronization
        // 初始化只是初始化一个空集合
        TransactionSynchronizationManager.initSynchronization();
    }
}
```

现在看第二个分支(外部不存在真实事务，创建一个新的真实事务)

```java
// 挂起资源(看下面具体解析)
// suspend方法的参数是一个事务对象，代表要挂起该事务，并且情况线程绑定的资源以及事务状态
// 这里外部不存在真实事务，为什么也要调用呢？
// 这是因为外部可能存在一个空事务，需要将这个外部空事务曾经向当前线程绑定的事务状态清空
// 而空事务的事务对象就是null, 这也是为什么参数传null的原因。
SuspendedResourcesHolder suspendedResources = suspend(null);
if (debugEnabled) {
    logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
}
try {
    // 默认为true
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    DefaultTransactionStatus status = newTransactionStatus(
        definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
    // 重点来了，这里便会从数据源中拿到一个连接，对这个连接设置一系列参数(自动提交，隔离级别，事务只读)
    // 同时将连接绑定到当前线程，具体往下看
    doBegin(transaction, definition);
    // 此方法已经说过，不再展开
    prepareSynchronization(status, definition);
    return status;
}
catch (RuntimeException ex) {
    // 出现异常，恢复挂起的资源
    resume(null, suspendedResources);
    throw ex;
}
catch (Error err) {
    resume(null, suspendedResources);
    throw err;
}

// =====================================分割线=====================================

protected final SuspendedResourcesHolder suspend(Object transaction) throws TransactionException {
	// 如果为true，说明存在外部事务(空事务或者真实事务)同步过，我们需要将同步过的状态清空并
    // 保存到当前事务的TransactionStatus对象中，当前事务结束后会根据保存的信息将外部事务状态
    // 恢复，重新绑定到当前线程
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        // 如果有注册过TransactionSynchronization，则触发每一个TransactionSynchronization
        // 接口的suspend钩子，并且将同步状态清除掉。
        // 也就是说再调用TransactionSynchronizationManager.isSynchronizationActive()会返回
        // false. 同时返回TransactionSynchronization列表，用以将来的恢复。
        List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchro
            nization();
        try {
            Object suspendedResources = null;
            // 挂起事务对象，也就是将事务持有的连接对象置为null
            // 同时将当前线程的绑定的连接清空
            if (transaction != null) {
                suspendedResources = doSuspend(transaction);
            }
            // 一系列清空操作
            String name = TransactionSynchronizationManager.getCurrentTransactionName();
            TransactionSynchronizationManager.setCurrentTransactionName(null);
            boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
            TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
            Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
            boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
            TransactionSynchronizationManager.setActualTransactionActive(false);
            return new SuspendedResourcesHolder(
                suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
        }
        catch (RuntimeException ex) {
            // doSuspend failed - original transaction is still active...
            doResumeSynchronization(suspendedSynchronizations);
            throw ex;
        }
        catch (Error err) {
            // doSuspend failed - original transaction is still active...
            doResumeSynchronization(suspendedSynchronizations);
            throw err;
        }
    }
    else if (transaction != null) {
        // Transaction active but no synchronization active.
        Object suspendedResources = doSuspend(transaction);
        return new SuspendedResourcesHolder(suspendedResources);
    }
    else {
        // Neither transaction nor synchronization active.
        return null;
    }
}

// =====================================分割线=====================================
// 既然说了suspend，那么作为对称，把resume也放这说了吧
protected final void resume(Object transaction, SuspendedResourcesHolder resourcesHolder)
			throws TransactionException {

    if (resourcesHolder != null) {
        // 获取被挂起的连接
        Object suspendedResources = resourcesHolder.suspendedResources;
        if (suspendedResources != null) {
            // 重新将连接绑定到当前线程
            doResume(transaction, suspendedResources);
            // doResume 调的是以下代码
            // TransactionSynchronizationManager.bindResource(
            //                       this.dataSource, suspendedResources);
        }
        // 恢复被挂起事务的各种属性
        List<TransactionSynchronization> suspendedSynchronizations = resourcesHol
             der.suspendedSynhronizations;
        if (suspendedSynchronizations != null) {
            TransactionSynchronizationManager.setActualTransactionActive(
                resourcesHolder.wasActive);
            TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
                resourcesHolder.isolationLevel);
            TransactionSynchronizationManager.setCurrentTransactionReadOnly(
                resourcesHolder.readOnly);
            TransactionSynchronizationManager.setCurrentTransactionName(
                resourcesHolder.name);
            doResumeSynchronization(suspendedSynchronizations);
        }
	}
}

private void doResumeSynchronization(List<TransactionSynchronization> suspendedSynchronizations) {
    // 重新初始化同步状态，并将这些TransactionSynchronization恢复
    TransactionSynchronizationManager.initSynchronization();
    for (TransactionSynchronization synchronization : suspendedSynchronizations) {
        // 触发resume钩子
        synchronization.resume();
        TransactionSynchronizationManager.registerSynchronization(synchronization);
    }
}

@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        // 事务对象中connectionHolder属性为null
        // 还记得doGetTransaction()方法吗，此方法会获取线程中绑定的连接，如果没有绑定就是null
        if (!txObject.hasConnectionHolder() ||
            txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            // 从数据源中获取连接
            Connection newCon = this.dataSource.getConnection();
            if (logger.isDebugEnabled()) {
                logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
            }
            // 重新设置值，第二个参数为true, 代表这是一个新的connectionHolder
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        // 这里会根据传进来的definition设置连接的隔离级别以及事务是否只读
        // 并且返回从数据库获取时的隔离级别，以便将来将连接归还时恢复
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);

       // 将自动提交模式修改为false, 开启一个真实的事务
        if (con.getAutoCommit()) {
            // 设置这个值的目的是归还连接时需不需要恢复自动提交模式的值
            txObject.setMustRestoreAutoCommit(true);
            if (logger.isDebugEnabled()) {
                logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }
            con.setAutoCommit(false);
        }

        // 这里默认不会执行的，其实不太清除这里的作用，如果readOnly设置成true, 已经对连接设置了值
        // 为什么要对数据库立即执行下 SET TRANSACTION READ ONLY
        prepareTransactionalConnection(con, definition);
        // 设置值为true, 代表存在一个真实的事务
        txObject.getConnectionHolder().setTransactionActive(true);

        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // 如果是新的connectionHolder，会绑定到当前线程
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
        }
    }

    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, this.dataSource);
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}

```

最后看第一个分支(外部存在一个真实的事务)

```java
// 外部存在一个真实的事务
if (isExistingTransaction(transaction)) {	
	return handleExistingTransaction(definition, transaction, debugEnabled);
}

@Override
protected boolean isExistingTransaction(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    // transactionActive的值在doBegin中被设置成true
    return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().
            isTransactionActive());
}

private TransactionStatus handleExistingTransaction(
    TransactionDefinition definition, Object transaction, boolean debugEnabled)
    throws TransactionException {

    // PROPAGATION_NEVER: 以非事务方式运行，如果外部存在真实事务，则抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
            "Existing transaction found for transaction marked with propagation 'never'");
    }

    // PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果外部存在真实事务，则挂起外部事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction");
        }
        // 挂起外部事务，这将清空事务对象的connectionHolder以及线程绑定的各种事务信息
        // 相当于回到线程首次调用事务的getTransaction的状态
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        // 返回一个DefaultTransactionStatus, 此方法已经说过
        // 注意的是第三个参数为false
        return prepareTransactionStatus(
            definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }

    // PROPAGATION_REQUIRES_NEW: 外部不存在真实事务，则新建一个新的真实事务
    // 如果存在真实事务，则挂起外部事务，新建一个新的真实事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction, creating new transaction with name [" +
                         definition.getName() + "]");
        }
        // 挂起外部事物
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // 开启事务
            doBegin(transaction, definition);
            // 如果需要同步，则同步事务状态到当前线程
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException beginEx) {
            // 发生异常，恢复挂起的事务
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
        catch (Error beginErr) {
            resumeAfterBeginException(transaction, suspendedResources, beginErr);
            throw beginErr;
        }
    }

    // PROPAGATION_NESTED：外部不存在真实事务，创建一个新的真实事务，外部存在真实事务，则是嵌套事务
    // 使用JDBC的保存点实现
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        // 不允许嵌套事务，则抛出异常
        // 这个值在事务管理器构造函数就初始化为true了
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                "Transaction manager does not allow nested transactions by default - " +
                "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        if (debugEnabled) {
            logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
        }
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status =
                prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            // 创建一个保存点
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            // Nested transaction through nested begin and commit/rollback calls.
            // Usually only for JTA: Spring synchronization might get activated here
            // in case of a pre-existing JTA transaction.
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                definition, transaction, true, newSynchronization, debugEnabled, null);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
    }

    // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
    // 参与外部真实事务
    if (debugEnabled) {
        logger.debug("Participating in existing transaction");
    }
    // 这里校验下此事务的定义于外部事务是否有差别，不一致会抛出异常
    if (isValidateExistingTransaction()) {
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                Constants isoConstants = DefaultTransactionDefinition.constants;
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                                                           definition + "] specifies isolation level which is incompatible with existing transaction: " +
                                                           (currentIsolationLevel != null ?
                                                            isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                                                            "(unknown)"));
            }
        }
        if (!definition.isReadOnly()) {
            if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                                                           definition + "] is not marked as read-only but existing transaction is");
            }
        }
    }
    // 老代码了，应该已经不陌生了
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}

```

分析完getTransaction这个方法，对Spring开启事务便有一个全貌了。

#### 3.2 回滚(roolback)

```java
public final void rollback(TransactionStatus status) throws TransactionException {
    // 检查事务状态，每一个事务只能调用rollback或者commit一次.
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }
    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    processRollback(defStatus);
}

private void processRollback(DefaultTransactionStatus status) {
    try {
        try {
            // 触发TransactionSynchronization接口的beforeCompletion回调方法
            triggerBeforeCompletion(status);
            // 存在保存点，说明是嵌套事务，回滚到该保存点
            // 即外部存在一个真实事务，并且该事务是一个嵌套事务(PROPAGATION_NESTED)
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Rolling back transaction to savepoint");
                }
                status.rollbackToHeldSavepoint();
            }
            // 是一个新的真实事务，直接回滚，调用conn.rollback()
            // 分为两种情况
            // 1. 外部存在真实事务
            // 该事务可能的传播行为: PROPAGATION_REQUIRES_NEW
            // 2. 外部不存在真实事务
            // 该事务可能的传播行为: PROPAGATION_REQUIRES_NEW, PROPAGATION_REQUIRED
            // 以及PROPAGATION_NESTED
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction rollback");
                }
                doRollback(status);
            }
            // 走到这里说明该事务不是一个新的事务，但是存在一个事务对象，说明该事务参与到外部事务
            // 即外部存在一个事务，该事务可能的传播行为为以下两个之一
            // PROPAGATION_REQUIRED，PROPAGATION_SUPPORTS
            else if (status.hasTransaction()) {
                // isGlobalRollbackOnParticipationFailure()默认true
                // 如果外部调用过status.setRollbackOnly()
                // 那么status.isLocalRollbackOnly()为true
                if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipation
                    Failure()) {
                    if (status.isDebug()) {
                        logger.debug("Participating transaction failed - marking existing 								transaction as rollback-only");
                    }
                    // 这里设置事务对象rollbackOnly为true, 其实是事务对象中的connectionHolder
                    // 因为内部事务是参与到外部事务，共享一个connectionHolder
                    // 这个操作会影响到外部事务结果，会使得外部事务即使调用commit也会发生回滚
                    // 具体commit方法再说.
                    doSetRollbackOnly(status);
                }
                else {
                    if (status.isDebug()) {
                        logger.debug("Participating transaction failed - letting        							transaction originator decide on rollback");
                    }
                }
            }
            // 走到这里说明是以非事务方式运行的(称这种情况为空事务),不需要回滚，什么都不做。
            // 分为两种情况
            // 1. 外部存在事务
            // 此空事务的传播行为：PROPAGATION_NOT_SUPPORTED
            // 2. 外部不存在事务
            // 此空事务可能的传播行为：PROPAGATION_NOT_SUPPORTED，PROPAGATION_SUPPORTS以及
            // PROPAGATION_NEVER
            else {
                logger.debug("Should roll back transaction but cannot - no transaction 							available");
            }
        }
        catch (RuntimeException ex) {
            // 触发TransactionSynchronization接口的afterCompletion回调方法
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw ex;
        }
        catch (Error err) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw err;
        }
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
    }
    finally {
        // 事务完成后，清理线程中绑定的事务相关属性以及资源，同时重置连接并释放
        cleanupAfterCompletion(status);
    }
}

private void cleanupAfterCompletion(DefaultTransactionStatus status) {
    // 设置事务状态已完成
    status.setCompleted();
    // 是一个新的事务同步，清理线程中绑定的事务相关属性以及资源
    if (status.isNewSynchronization()) {
        TransactionSynchronizationManager.clear();
    }
    // 是一个新的真实事务
    if (status.isNewTransaction()) {
        // 清空绑定的连接，并且将连接重置为初始状态，然后释放
        doCleanupAfterCompletion(status.getTransaction());
    }
    // 恢复挂起的资源
    if (status.getSuspendedResources() != null) {
        if (status.isDebug()) {
            logger.debug("Resuming suspended transaction after completion of inner 						transaction");
        }
        resume(status.getTransaction(), (SuspendedResourcesHolder) 						 status.getSuspendedResources());
    }
}
```

分析rollback的逻辑，我们可以看到rollback的逻辑分支是与getTransaction方法对应的。

#### 3.3 提交(commit)

```java
public final void commit(TransactionStatus status) throws TransactionException {
    // 与回滚一样，检查下事务状态是否完成
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    // 在提交前，调用者使用status.setRollbackOnly()方法，事务只能是回滚一种结果
    if (defStatus.isLocalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Transactional code has requested rollback");
        }
        processRollback(defStatus);
        return;
    }
    // shouldCommitOnGlobalRollbackOnly()默认返回false
    // defStatus.isGlobalRollbackOnly()定义如下
    // public boolean isGlobalRollbackOnly() {
	//	  return ((this.transaction instanceof SmartTransactionObject) &&
	//			((SmartTransactionObject) this.transaction).isRollbackOnly());
	// }
    // 注: 下文所说的回滚只是调用Spring事务的rollback API, 只有最外层事务才会发生真正的回滚.
    // 事务本身是不会修改事务对象的rollbackOnly的，还记得rollback方法里的调用的doSetRollbackOnly方     // 法吗? 当参与的内部事务发生了回滚，会修改事务对象rollbackOnly属性，因为参与的内部事务与外部事物
    // 的事务对象共享同一个connectionHolder。这样即使外部事物仍然显示调用commit方法也会进行回滚。
    // 同时最外层事务会抛出一个UnexpectedRollbackException
    // 抛出异常是因为内部参与事务是回滚的，而内部事务与外部事务其实是同一个真实的事务，本应该也是调用         // rollback方法进行回滚，但是却调用了commit，所以抛出一个异常警告。
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
        }
        processRollback(defStatus);
        // Throw UnexpectedRollbackException only at outermost transaction boundary
        // or if explicitly asked to.
        if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
            throw new UnexpectedRollbackException(
                "Transaction rolled back because it has been marked as rollback-only");
        }
        return;
    }
    // 执行提交
    processCommit(defStatus);
}

private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        // 一个标记，以防triggerBeforeCommit抛出异常
        boolean beforeCompletionInvoked = false;
        try {
            prepareForCommit(status);
            // 触发TransactionSynchronization接口的beforeCommit方法
            triggerBeforeCommit(status);
            // 触发TransactionSynchronization接口的beforeCompletion方法
            triggerBeforeCompletion(status);
            // 标记为true
            beforeCompletionInvoked = true;
            boolean globalRollbackOnly = false;
            if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
                globalRollbackOnly = status.isGlobalRollbackOnly();
            }
            // 嵌套事务，提交操作为释放事务创建的保存点
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Releasing transaction savepoint");
                }
                status.releaseHeldSavepoint();
            }
            // 是一个新的真实事务，执行真正的提交操作
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction commit");
                }
                doCommit(status);
            }
            // 与上面对应，如果有一个全局rollbackOnly标志，如果上面的操作没有回滚，而是执行
            // processCommit提交了，也抛出一个UnexpectedRollbackException。
            if (globalRollbackOnly) {
                throw new UnexpectedRollbackException(
                    "Transaction silently rolled back because it has been marked as rollback-only");
            }
        }
        catch (UnexpectedRollbackException ex) {
            // can only be caused by doCommit
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
            throw ex;
        }
        catch (TransactionException ex) {
            // can only be caused by doCommit
            if (isRollbackOnCommitFailure()) {
                doRollbackOnCommitException(status, ex);
            }
            else {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            }
            throw ex;
        }
        catch (RuntimeException ex) {
            // 走到这里是在执行triggerBeforeCompletion前发生了异常，重新触发
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, ex);
            throw ex;
        }
        catch (Error err) {
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, err);
            throw err;
        }

        // Trigger afterCommit callbacks, with an exception thrown there
        // propagated to callers but the transaction still considered as committed.
        try {
            // 触发TransactionSynchronization接口的afterCommit方法
            triggerAfterCommit(status);
        }
        finally {
            // 触发TransactionSynchronization接口的afterCompeltion方法
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    }
    finally {
        // 清理操作，同rollback.
        cleanupAfterCompletion(status);
    }
}
```

### 4. 总结

本文比较细致的介绍了DataSourceTransactionManager的原理，可以看到整个实现采用的是模板方法模式。整个骨架已经在AbstractPlatformTransactionManager定义好了，子类只需要实现特定的以do开头的模板方法即可。本文没有对TransactionSynchronizationManager这个工具类做非常细的介绍，后面会再写一片文章来介绍这一个类以及DataSourceUtils这个比较重要的类。

