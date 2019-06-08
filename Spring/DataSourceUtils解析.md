### 1. 作用

看名字就能知道这个类是对DataSource的一个封装，这个类提供了一系列操作数据库连接的工具方法。这个类在Spring事务中非常重要，最主要的作用就是提供了能够从当前线程获取开启事务时绑定的连接。其中Spring Jdbc里的`JdbcTemplate`类就是采用`DataSourceUtils.getConnection()`方法获取连接的。

### 2. TransactionSynchronization

在分析`DataSourceUtils`的方法前，先简单介绍下这个接口吧。其实对于这个接口应该会很眼熟，应为在分析事务源码的时候，这个接口出现的频率很高。在事务的suspend、resume、rollback、commit方法中会触发这个接口提供的钩子。下面看下这个接口的定义

```java
public interface TransactionSynchronization extends Flushable {

	/** Completion status in case of proper commit */
	int STATUS_COMMITTED = 0;

	/** Completion status in case of proper rollback */
	int STATUS_ROLLED_BACK = 1;

	/** Completion status in case of heuristic mixed completion or system errors */
	int STATUS_UNKNOWN = 2;

	/**
	 * 此钩子在事务中的suspend方法触发。
	 * 这个钩子意味者如果自己在事务运行期间在当前线程绑定过资源，可能需要移除掉。
	 * 想象下事务里挂起的时候，会移除掉开启事务时绑定的connectionHolder.
	 */
	void suspend();

	/**
	 * 与suspend相反，在事务中的resume方法触发
	 */
	void resume();

	/**
	 * 此钩子在事务提交前执行
	 * 在调用commit方法时触发，之后再走commit逻辑。我们知道事务实际的结果还是需要根据
	 * commit方法来决定，因为即使调用了commit方法也仍然可能会执行回滚。
	 * 如果按照正常流程走提交的话，我们通过这个钩子方法由机会在事务真正提交前还能对数据库做一些操作。
	 */
	void beforeCommit(boolean readOnly);

	/**
	 * 此钩子在事务完成前执行(发生在beforeCommit之后)
	 * 调用commit或者rollback都能触发，此钩子在真正提交或者真正回滚前执行。
	 */
	void beforeCompletion();

	/**
	 * 事务真正提交后执行，这个钩子执行意味着事务真正提交了。
	 */
	void afterCommit();

	/**
	 * 事务完成后执行
	 * 可能是真正提交了，也有可能是回滚
	 */
	void afterCompletion(int status);

}

```

### 3. TransactionSynchronizationManager

这个类也简单提下吧，因为方法里的代码都很简单，这个类的作用就是绑定资源到当前线程、注册TransactionSynchronization接口、绑定事务的各个属性。

主要注意的点就是`isSynchronizationActive`方法以及`isActualTransactionActive`方法。前者代表事务同步是否激活，如果激活了才可以注册TransactionSynchronization接口事件。后者代表当前线程是否存在真实的事务。有前面事务分析知道，Spring开启一个空事务时，`isSynchronizationActive() = true`，而`isActualTransactionActive = false`，因为不存在真实的事务。

### 4. 方法解析

#### 4.1 获取连接(getConnection)

先简单介绍这个方法的作用，此方法的目的就是获取一个连接。连接的处理行为有两种

* **此连接由Spring事务管理，无论是空事务还是真实事务，只要此方法是在`getTransaction()`之后，在`commit`/`rollback`方法之前调用的。如果此事务是一个空事务，那么此方法会获取一个连接，并且会绑定到当前线程，同时会注册一个`ConnectionSynchronization`。当事务运行期间调用`commit` /`rollback`时便会触发`ConnectionSynchronization`的`beforeCompletion`或`afterCompletion`钩子来解绑线程中绑定的连接，然后释放掉。如果此事务是一个真实事务，那么此方法拿到的是开启事务时按照`TransactionDefinition`定义好的带事务连接，这个连接会绑定到当前线程，然后这个方法从当前线程获取。这个带事务的连接由事务管理器在事务真正回滚或提交后自动解绑释放。**
* **完全由调用者管理，与Spring事务无关，需要手动调用releaseConnection方法释放。**

注: 以上结论由以下代码以及事务管理器代码综合得出.

```java
public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {
    try {
        return doGetConnection(dataSource);
    }
    catch (SQLException ex) {
        throw new CannotGetJdbcConnectionException("Could not get JDBC Connection", ex);
    }
}
public static Connection doGetConnection(DataSource dataSource) throws SQLException {
    Assert.notNull(dataSource, "No DataSource specified");
    // 从当前线程获取绑定的ConnectionHolder
    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager
        .getResource(dataSource);
	// 如果不为空，并且内部持有连接，直接返回此连接
    // 首先开启事务绑定的conHolder是肯定持有连接的，那么什么时候conHolder.hasConnection() = false
    // 答案要继续往下找，才能明白。
    // 这里直接解释下: 此方法在一个空事务中会绑定一个connectionHolder，
    // 同时注册了一个ConnectionSynchronization事件，如果这个空事务被挂起，就会触发这个
    // 事件的suspend钩子，这个钩子除了会解绑这个connectionHolder外，还会判断
    // connectionHolder持有的连接是否还被外部程序使用，如果不再使用了，会提前释放掉。
    // 因此如果后面这个事务被恢复了，那么能从数据源中直接获取一个新的连接
    if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
        // 将引用计数加1，代表获取这个conHolder的次数
        conHolder.requested();
        if (!conHolder.hasConnection()) {
            logger.debug("Fetching resumed JDBC Connection from DataSource");
            conHolder.setConnection(dataSource.getConnection());
        }
        return conHolder.getConnection();
    }

    logger.debug("Fetching JDBC Connection from DataSource");
    // 直接从给定的数据源获取连接
    Connection con = dataSource.getConnection();
	// 如果事务同步是开启的，从前面事务源码分析，可以知道如果开启了一个空事务(非事务方式运行)，
    // 这个方法的值也会返回true.
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        logger.debug("Registering transaction synchronization for JDBC Connection");
        // 使用这个连接创建一个ConnectionHolder
        ConnectionHolder holderToUse = conHolder;
        if (holderToUse == null) {
            holderToUse = new ConnectionHolder(con);
        }
        else {
            holderToUse.setConnection(con);
        }
        holderToUse.requested();
        // 注册一个TransactionSynchronization
        // 这里出现一个新类ConnectionSynchronization，看下面解析。
        // 当事务运行整个期间会触发这个类实现的钩子方法
        TransactionSynchronizationManager.registerSynchronization(
            new ConnectionSynchronization(holderToUse, dataSource));
        holderToUse.setSynchronizedWithTransaction(true);
        if (holderToUse != conHolder) {
            // 将这个连接绑定到当前线程
            TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
        }
    }
    return con;
}
```

```java
// 主要关注这个实现的钩子方法都做了啥
// 首先明确一个观点，由getConnection方法可知，此类被注册前，拿到的连接都是数据源中新鲜的连接
// 也就意味者这个连接是非事务的。
private static class ConnectionSynchronization extends TransactionSynchronizationAdapter {

    private final ConnectionHolder connectionHolder;

    private final DataSource dataSource;

    private int order;

    private boolean holderActive = true;

    public ConnectionSynchronization(ConnectionHolder connectionHolder, 
                                     DataSource dataSource) {
        this.connectionHolder = connectionHolder;
        this.dataSource = dataSource;
        this.order = getConnectionSynchronizationOrder(dataSource);
    }

    @Override
    public int getOrder() {
        return this.order;
    }

    // 当存在一个空事务的时候，getConnection()方法会从数据源中拿到一个资源并绑定到当前线程。
    // 因此当这个空事务被挂起的时候，需要解绑
    // 当外部不存在真实事务时，新建一个新的真实事务，会挂起外部空事务
    // 具体可看AbstractPlatformTransactionManager中getTransaction方法的第二个大分支
    @Override
    public void suspend() {
        if (this.holderActive) {
            // 解绑绑定的连接
            TransactionSynchronizationManager.unbindResource(this.dataSource);
            // 如果这个连接没有再被引用了，也就是说不再使用了，将这个连接提前释放。
            // 这段代码也解决了getConnection方法提的疑问。
            if (this.connectionHolder.hasConnection() && 
                !this.connectionHolder.isOpen()) {
                // Release Connection on suspend if the application doesn't keep
                // a handle to it anymore. We will fetch a fresh Connection if the
                // application accesses the ConnectionHolder again after resume,
                // assuming that it will participate in the same transaction.
                releaseConnection(this.connectionHolder.
                                  getConnection(), this.dataSource);
                this.connectionHolder.setConnection(null);
            }
        }
    }

    @Override
    public void resume() {
        // 恢复，与suspend对应
        if (this.holderActive) {
            TransactionSynchronizationManager.bindResource(
                this.dataSource, this.connectionHolder);
        }
    }

    @Override
    public void beforeCompletion() {
        // 如果外部程序不再使用此连接，解绑connectionHolder，并且释放
        // 因为这个连接是非事务的，Spring在调用rollback或者commit不会有实际的事务操作
        // 因此如果不再使用可以提前释放掉
        if (!this.connectionHolder.isOpen()) {
            TransactionSynchronizationManager.unbindResource(this.dataSource);
            this.holderActive = false;
            if (this.connectionHolder.hasConnection()) {
                releaseConnection(this.connectionHolder
                                  .getConnection(), this.dataSource);
            }
        }
    }

    @Override
    public void afterCompletion(int status) {
        // 事务完成后，还没有在beforeCompletion钩子方法中解绑释放掉
        // 那么解绑释放掉
        if (this.holderActive) {
            // The thread-bound ConnectionHolder might not be available anymore,
            // since afterCompletion might get called from a different thread.
            // 这段注释没明白，会什么afterCompletion钩子会被多个线程调用
            // 而beforeCompletion却不会呢? 不应该都是在以一个线程内执行吗?
            // 现在知道的一点原因是在JTA事务中会发生多个线程调用的问题
            // DataSourceTransactionManager这个管理器不会发生上述情况
            TransactionSynchronizationManager.unbindResourceIfPossible(this.dataSource);
            this.holderActive = false;
            if (this.connectionHolder.hasConnection()) {
                releaseConnection(this.connectionHolder.
                                  getConnection(), this.dataSource);
                // Reset the ConnectionHolder: It might remain bound to the thread.
                this.connectionHolder.setConnection(null);
            }
        }
        this.connectionHolder.reset();
    }
}

```

#### 4.2 释放连接(releaseConnection)

```java
public static void releaseConnection(Connection con, DataSource dataSource) {
    try {
        doReleaseConnection(con, dataSource);
    }
    catch (SQLException ex) {
        logger.debug("Could not close JDBC Connection", ex);
    }
    catch (Throwable ex) {
        logger.debug("Unexpected exception on closing JDBC Connection", ex);
    }
}

public static void doReleaseConnection(Connection con, 
                                       DataSource dataSource) throws SQLException {
    if (con == null) {
        return;
    }
    if (dataSource != null) {
        ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationM
            anager.getResource(dataSource);
        // 此连接受Spring事务管理，可能是一个空事务，也可能是一个真实事务
        if (conHolder != null && connectionEquals(conHolder, con)) {
            // 将此连接引用数减一
            conHolder.released();
            return;
        }
    }
    logger.debug("Returning JDBC Connection to DataSource");
    // 不受Spring事务管理的连接，释放掉，归还给数据源。
    doCloseConnection(con, dataSource);
}
```

可以看到如果我们直接调用这个方法来释放连接时，如果这个连接由Spring事务管理，并不会真的将这个连接释放掉，只是将这个连接的引用次数减一，要想真的释放掉这个连接，需要先解绑这个拥有此连接的`connectinHolder`，然后再调用次方法才能释放掉。观察事务管理器的释放操作，也可以看到事务真正提交或者回滚后要释放掉这个连接时，会先解绑，然后再调用此方法释放连接。

#### 4.3 isConnectionTransactional

此方法用来判断给定的连接是不是由Spring事务管理器绑定在当前线程，包括真实事务以及空事务。

* 真实事务由事务管理的`getTransaction`方法绑定，准确点我们知道其实是`doBegin`方法。

* 空事务是由`DataSourceUtils.getConnection()`绑定。下面用一段伪代码描述

  ```java
  // 已非事务方式运行(开启一个空事务)
  TransactionDefinition definition = new DefaultTransactionDefinition(
  				TransactionDefinition.PROPAGATION_NOT_SUPPORTED);
  TransactionStatus status = transactionManager.getTransaction(definition);
  try {
      // 此种情况会绑定连接到当前线程
      Connection conn = DataSourceUtils.getConnection(dataSource);
      // use conn doSomething
      transactionManager.commit(status);
  } catch(RuntimeExecption e) {
       transactionManager.rollback(status);
  }
    
  ```

代码分析:

```java
/** 
 * 代码很简单，理解了上面两点，这段代码一看就懂
 * 如果要判断此连接是Spring事务管理，并且是真实事务
 * 可以这样简单判断
 * return conHolder != null 
 *        && conHolder.getConnection() == con 
 *        && conHolder.isTransactionActive();
 */
public static boolean isConnectionTransactional(Connection con, DataSource dataSource) {
    if (dataSource == null) {
        return false;
    }
    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    return (conHolder != null && connectionEquals(conHolder, con));
}
```

### 5. 总结

本文主要分析了工具类中最重要也是最常用的几个方法，其它方法非常简单，也就没啥好说的了。
