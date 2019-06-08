### 简介

`SqlSessionTemplate`是mybatis-spring中最核心的一个类，我们知道MyBatis暴露出的最外层接口是`SqlSession`，所有的操作都是借助`SqlSession`接口的方法来完成的。MyBatis本身有一个默认实现类，也是我们在单独使用MyBatis时最常见的一个实现类`DefalutSqlSession`。而当我们将MyBatis与Spring整合时，便不再使用这个默认实现了，取而代之的是`SqlSessionTemplate`。与默认实现相比主要有如下区别:

* `SqlSessionTemplate`是线程安全的，可以被多个DAO共享，而`DefaultSqlSession`线程不安全。至于线程不安全的原因显而易见，因为一个`DefaultSqlSession`实际可以代表一个`Connection`，如果在多线程中使用时，一个线程在执行数据库操作，另一个线程执行别的操作时直接将事务提交了，岂不是就乱套了。因此MyBatis官方文档建议`DefaultSqlSession`的最佳作用域是方法作用域。
* `SqlSessionTemplate`是不支持事务以及关闭方法的，也就是`commit`、`rollback`以及`close`。如果显示调用这几个方法，会抛出一个异常。事务的提交回滚以及`SqlSession`的关闭全都由自己自动管理，不需要外部程序参与。

### 前置知识

想要看懂这篇文章，首先需要熟悉MyBatis本身的工作流程。其次，因为这个类涉及了很多与Spring事务相关的知识点，因此还需要熟悉Spring的事务机制与原理。

对于Spring事务的事务机制，可以看以下两篇文章。

* [Spring事务源码分析](https://www.cnblogs.com/wt20/p/10957371.html)
* [DataSourceUtils源码分析](https://www.cnblogs.com/wt20/p/10958238.html)

### SpringManagedTransaction

在分析`SqlSessionFactoryBean`时，对这个新的事务对象以及这个对象的工厂类只是一笔带过，在分析`SqlSessionTemplate`之前，有必要先说明下这个类。

MyBatis本身内部有提供事务相关的API，但是与Spring整合后，需要将事务交给Spring来管理，以前的`JdbcTransaction`是不能与Spring一起工作的。而`SpringManagedTransaction`就是为了与Spring整合而设计的一个新的事务(Transaction接口)实现类。

先看`SpringManagedTransaction`的创建，一般都是使用`SpringManagedTransactionFactory`这个工厂类来创建。

```java
public class SpringManagedTransactionFactory implements TransactionFactory {

    /**
     * 会忽略隔离级别以及自动提交这两个参数
     */
    @Override
    public Transaction newTransaction(DataSource dataSource, 
                                      TransactionIsolationLevel level, 
                                      boolean autoCommit) {
        return new SpringManagedTransaction(dataSource);
    }
	// 其余方法略
}


public class SpringManagedTransaction implements Transaction {

    private static final Logger LOGGER = LoggerFactory.getLogger(
        SpringManagedTransaction.class);

    private final DataSource dataSource;

    private Connection connection;

    private boolean isConnectionTransactional;

    private boolean autoCommit;

    /** 只是简单的赋值 **/
    public SpringManagedTransaction(DataSource dataSource) {
        notNull(dataSource, "No DataSource specified");
        this.dataSource = dataSource;
    }

  	/**
  	 * 获取一个连接，通过DataSourceUtils.getConnection实现
  	 * 如果有事务(Spring管理),则会返回当前线程绑定的连接
  	 * 否则从数据源中拿到一个新连接
  	 */
    @Override
    public Connection getConnection() throws SQLException {
        if (this.connection == null) {
            openConnection();
        }
        return this.connection;
    }
    // 获取连接, 全都是DataSourceUtils里的方法，因此需要先弄懂这个类
    private void openConnection() throws SQLException {
        this.connection = DataSourceUtils.getConnection(this.dataSource);
        this.autoCommit = this.connection.getAutoCommit();
        this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(
            this.connection, this.dataSource);
    }

   /**
    * 提交操作，只有不是Spring事务管理的连接，并且这个连接从数据源中取出来就需要手动提交
    * 时才提交
    */
    @Override
    public void commit() throws SQLException {
        if (this.connection != null && 
            !this.isConnectionTransactional && !this.autoCommit) {
            LOGGER.debug(() -> "Committing JDBC Connection [" + this.connection + "]");
            this.connection.commit();
        }
    }
    /**
     * 同提交
     */
    @Override
    public void rollback() throws SQLException {
        if (this.connection != null && 
            !this.isConnectionTransactional && !this.autoCommit) {
            LOGGER.debug(() -> "Rolling back JDBC Connection [" + this.connection + "]");
            this.connection.rollback();
        }
    }
	/**
	 * 将连接释放
	 */
    @Override
    public void close() {
        DataSourceUtils.releaseConnection(this.connection, this.dataSource);
    }
}


```

可以看到这个类非常简单，前提时必须要把提到的前置知识搞明白。

### 源码分析

接下来就要从源码层面上揭开上述所说两点的秘密了。

先看构造方法，类中提供了三个重载构造方法，我们看参数最多的那个即可。

```java
/**
 * sqlSessionFactory: 用于创建SqlSesion的工厂类，可以使用SqlSessionFactoryBean来构建，
 *                    SqlSessionFactoryBean前文已介绍过。
 * executorType: 指定SqlSession中Executor的类型，默认是SimpleExecutor，有以下几个选项
 *               ExecutorType.SIMPLE、ExecutorType.REUSE、 ExecutorType.BATCH
 * exceptionTranslator: 异常转换器，将MyBatis中的异常转换成Spring中的DataAccessException异常
 */
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
                          PersistenceExceptionTranslator exceptionTranslator) {
	//对必要参数的非空判断
    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    // 使用JDK动态代理创建一个SqlSession代理，以后所有的数据库操作就委托给它啦
    // 因此这一步是关键
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
}
```

代理拦截

```java
private class SqlSessionInterceptor implements InvocationHandler {
    // 拦截方法，每当调用sqlSession中的方法时，都会先进入到这里。
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 为什么说SqlSessionTemplate是线程安全的，就是下面这段代码了
        // 每次调用方法，都会去获取一个SqlSession(其实就是DefaultSqlSession)
        // 这个方法非常重要，下面会详细说。现在我们只要知道，使用这个方法不会每次都新建一个SqlSession
        // 主要是以下两种情况：
        // 1. 在一个事务期间(特指是在Spring事务管理)内，拿的这个SqlSession是同一个，会使用		           // ThreadLocal将SqlSession绑定到当前线程，因此在一个事务方法内多次调用insert、update、   
        // select等方法使用的是同一个SqlSession，而且线程之间是隔离的。对于这种情况，事务的回滚和
        // 提交全部由Spring事务管理设施自动操作。
        // 2. 在Spring管理的事务之外，每次拿到的SqlSession都是一个新的。
        //    2.1 从数据源拿到的连接(conn.getAutoCommit() == false)是手动提交的，那么每执行一次
        //        都会自动提交或者回滚。即一个操作就是一个事务。
        //    2.2 从数据源拿到的连接是自动提交的，这种情况就不用说了。
        SqlSession sqlSession = getSqlSession(
            SqlSessionTemplate.this.sqlSessionFactory,
            SqlSessionTemplate.this.executorType,
            SqlSessionTemplate.this.exceptionTranslator);
        try {
            // 执行拦截方法
            Object result = method.invoke(sqlSession, args);
            // 非事务，即不是运行在Spring事务方法getTransaction，rollback/commit之间
            if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.
                                           sqlSessionFactory)) {
                // force commit even on non-dirty sessions because some databases require
                // a commit/rollback before calling close()
                // 对应第2种情况，commit会判断conn.getAutoCommit()的值是否需要调用    		
                // conn.commmit();
                sqlSession.commit(true);
            }
            return result;
        } catch (Throwable t) {
            // 转换异常，略
            Throwable unwrapped = unwrapThrowable(t);
            if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
                closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                sqlSession = null;
                Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
                if (translated != null) {
                    unwrapped = translated;
                }
            }
            throw unwrapped;
        } finally {
            if (sqlSession != null) {
                // 关闭sqlSession
                // 如果是在Spring管理的事务中，只是将此次事务中获取SqlSession的次数减一
                // 真正的关闭动作是在钩子(beforeCompletion或afterCompletion)回调时调用
                // 如果不是在Spring管理的事务，这直接掉用sqlSessiion.close方法
                // 其实我们看到上面出现异常其实没有调用sqlSession.rollback方法，这会在close
                // 时智能的判断需不需要回滚。
                closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            }
        }
    }
}
```

下面来分析`getSqlSession`、`isSqlSessionTransactional`以及`closeSqlSession`这三个关键方法，它们都是`SqlSessionUtils`里的工具方法，负责管理`SqlSession`的生命周期。

先看`getSqlSession`

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
	// 检查参数
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);
	// 从当前线程获取绑定的SqlSessionHolder
    // 如果不为null，说明是在Spring事务管理的环境下运行，直接返回里面的sqlSession
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager
        .getResource(sessionFactory);

    // 获取SqlSession，看下面方法解释
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
        return session;
    }

    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Creating a new SqlSession");
    }
	// 走到这里说明SqlSession为null
    // 1. Spring事务管理之外的情况，每次都会获取一个新的SqlSession(DefaultSqlSession)
    // 2. Spring事务管理内，那么是事务内首次获取SqlSession，接下来会将这个SqlSession
    //    绑定到当前线程。
    session = sessionFactory.openSession(executorType);

    // 如果是上述说的第2中情况，则准备绑定SqlSession到当前线程，并且注册SqlSessionSynchronization
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
}

private static SqlSession sessionHolder(ExecutorType executorType, 
                                        SqlSessionHolder holder) {
    // 如果不是Spring管理的事务中，那么holder = null
    // 直接返回null
    SqlSession session = null;
    if (holder != null && holder.isSynchronizedWithTransaction()) {
        // 检查下同一Spring管理的事务中再次获取SqlSession时的executorType
        // 如果与第一次不一样，抛出异常，第一次获取时会绑定到当前线程
        // 供在同一个事务中再次获取时使用。
        if (holder.getExecutorType() != executorType) {
            throw new TransientDataAccessResourceException("Cannot change the  	ExecutorType when there is an existing transaction");
        }
		// 获取次数加1
        holder.requested();
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Fetched SqlSession [" + holder.getSqlSession() + "] from current transaction");
        }
        session = holder.getSqlSession();
    }
    return session;
}

private static void registerSessionHolder(SqlSessionFactory sessionFactory, 
                                          ExecutorType executorType,
                                          PersistenceExceptionTranslator                 
                                          exceptionTranslator, 
                                          SqlSession session) {
    SqlSessionHolder holder;
    // 只有是在开启了Spring事务时，这个方法才返回true
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        Environment environment = sessionFactory.getConfiguration().getEnvironment();

        // 如果MyBatis中使用的事务工厂是SpringManagedTransactionFactory(默认)
        if (environment.getTransactionFactory() instanceof SpringManagedTransaction
            Factory) {
			// 创建一个SqlSessionHolder
            holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
            // 绑定到当前线程，供后续同一Spring事务中使用
            TransactionSynchronizationManager.bindResource(sessionFactory, holder);
            // 注册一个事务回调的接口，这里的方法依次会在Spring事务的某个阶段触发
            // 上面绑定的SqlSessionHolder就是在这里的钩子方法注销的，往下看吧
            TransactionSynchronizationManager.registerSynchronization(
                new SqlSessionSynchronization(holder, sessionFactory));
            // 设置值为true，sessionHolder方法内有用到
            holder.setSynchronizedWithTransaction(true);
            // 获取次数加1
            holder.requested();
        } else {
            // Spring事务使用的数据源与MyBatis中的数据源不一致，那么还是运行在Spring事务之外咯
            // 其实我觉得这一行应该要先判断，然后再判断事务工厂，再绑定sqlSession?
            // 这里可以讨论下?
            if (TransactionSynchronizationManager.getResource(
                environment.getDataSource()) == null) {
                LOGGER.debug(() -> "SqlSession [" + session
                             + "] was not registered for synchronization because DataSource is not transactional");
            } else {
                // Spring事务使用的数据源与MyBatis中的数据源一致，抛出异常.
                throw new TransientDataAccessResourceException(
                    "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
            }
        }
    } else {
        // 不是在Spring事务中运行
        LOGGER.debug(() -> "SqlSession [" + session
                     + "] was not registered for synchronization because synchronization is not active");
    }

}

// 最后再看下这个事务的回调吧，这个类最终肯定要实现TransactionSynchronization接口的
// 其实可以发现这个类的逻辑和Spring中DataSourceUtils中的ConnectionSynchronization逻辑几乎完全一致
// 主要关注这几个钩子方法吧
private static final class SqlSessionSynchronization extends TransactionSynchronization
-Adapter {
    
    // 事务被挂起时，解绑绑定的sqlSessionHolder
    public void suspend() {
        if (this.holderActive) {
            LOGGER.debug(() -> "Transaction synchronization suspending SqlSession [" + this.holder.getSqlSession() + "]");
            TransactionSynchronizationManager.unbindResource(this.sessionFactory);
        }
    }
    // 与suspend对应
    public void resume() {
      if (this.holderActive) {
        LOGGER.debug(() -> "Transaction synchronization resuming SqlSession [" + this.holder.getSqlSession() + "]");
        TransactionSynchronizationManager.bindResource(this.sessionFactory, this.holder);
      }
    }

   
   // 这里被触发是因为调用了transactionManager.commit()方法，Spring事务马上可能就要提交了
   // 这里需要调用下sqlSession.commit()以完成MyBatis内部缓存的相关工作以及刷新批处理
   // 如果不调用的话，那么在sqlSession.close()时，内部就会走回滚逻辑了。
   // 当然了这里虽然调用sqlSession.commit()，但是对于数据库事务因为会调用MyBatis事务对象的commit
   // 而SpringManagedTransaction这个对象的commit有做过判断，因此不会影响Spring的事务提交
    @Override
    public void beforeCommit(boolean readOnly) {
      if (TransactionSynchronizationManager.isActualTransactionActive()) {
        try {
          LOGGER.debug(() -> "Transaction synchronization committing SqlSession [" + this.holder.getSqlSession() + "]");
          this.holder.getSqlSession().commit();
        } catch (PersistenceException p) {
          if (this.holder.getPersistenceExceptionTranslator() != null) {
            DataAccessException translated = this.holder.getPersistenceExceptionTranslator()
                .translateExceptionIfPossible(p);
            if (translated != null) {
              throw translated;
            }
          }
          throw p;
        }
      }
    }

   // 整个事务即将完成，需要清理线程中绑定的SqlSession，以及关闭
   // 这里曾经有一个bug，可以去看下Issue #18的有关讨论
   // 主要是JTA事务，其实对于DataSourceTransactionManager这个事务管理器是不存在这个Issue所说的问题
    @Override
    public void beforeCompletion() {
      // Issue #18 Close SqlSession and deregister it now
      // because afterCompletion may be called from a different thread
      if (!this.holder.isOpen()) {
        LOGGER
            .debug(() -> "Transaction synchronization deregistering SqlSession [" + this.holder.getSqlSession() + "]");
        TransactionSynchronizationManager.unbindResource(sessionFactory);
        this.holderActive = false;
        LOGGER.debug(() -> "Transaction synchronization closing SqlSession [" + this.holder.getSqlSession() + "]");
        this.holder.getSqlSession().close();
      }
    }
	// 同上
    @Override
    public void afterCompletion(int status) {
      if (this.holderActive) {
        // afterCompletion may have been called from a different thread
        // so avoid failing if there is nothing in this one
        LOGGER
            .debug(() -> "Transaction synchronization deregistering SqlSession [" + this.holder.getSqlSession() + "]");
        TransactionSynchronizationManager.unbindResourceIfPossible(sessionFactory);
        this.holderActive = false;
        LOGGER.debug(() -> "Transaction synchronization closing SqlSession [" + this.holder.getSqlSession() + "]");
        this.holder.getSqlSession().close();
      }
      this.holder.reset();
    }
  }

```

总结下`getSqlSession`方法:

`getSqlSession`主要的目的就是获取一个`SqlSession`对象，主要分为下面两大情况:

* 在Spring事务管理内，那么首次调用此方法，会绑定一个`SqlSession`对象到当前线程，供后续调用，而不用重新创建。随着Spring事务的提交方法被调用时会触发`beforeCommit`钩子而执行`sqlSession.commit()`以完成MyBatis的内部流程(缓存、批处理)，或者是Spring事务的回滚方法。无论是提交还是回滚最后都会触发`beforeCompletion`以及`afterCompletion`钩子来解绑绑定的`SqlSession`以及`close`。
* Spring事务管理之外，每次都会返回一个新的`SqlSession`。`SqlSession`的提交、回滚、释放都会在每一个方法执行后得到应有的调用，在上面动态代理拦截逻辑中体现。

再看`closeSqlSession`

代码非常简单，不用做过多的解释。

```java
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager
        .getResource(sessionFactory);
    // 是Spring事务管理的，将获取次数减一
    if ((holder != null) && (holder.getSqlSession() == session)) {
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Releasing transactional SqlSession [" + session + "]");
        }
        holder.released();
    } else {
        // 否则直接释放
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Closing non transactional SqlSession [" + session + "]");
        }
        session.close();
    }
}
```

最后是`isSqlSessionTransactional` 

此方法就使用来判断给定的`SqlSession`对象是不是运行在Spring事务中，如果看懂了`getSqlSession`方法的话，也不用做过多解释了。

```java
public static boolean isSqlSessionTransactional(SqlSession session, 
                                                SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager
        .getResource(sessionFactory);
    return (holder != null) && (holder.getSqlSession() == session);
}
```

至此`SqlSessionTemplate`中最核心的源码就全部分析完毕了，而类中其它方法都是通过这个代理`SqlSession`完成的。

### 总结

本文主要分析了`SqlSessionTemplate`这个类的实现原理，解释了这个类为什么是线程安全的，又是如何参与进Spring管理的事务中这两个最大的特征点。