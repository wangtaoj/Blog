### PlatformTransactionManager

```java
/**
 * Spring事务抽象的顶级接口
 * 以下所说的具体行为以DataSourceTransactionManager这个实现类为准
 */
public interface PlatformTransactionManager {
    
    /**
     * 开启一个事务, 即从给定的数据源中获取一个连接，关闭自动提交模式，并且将连接绑定到当前线程
     * 并且使用给定的参数设置连接的一些行为(隔离级别, 事务只读特性)
     * 如果指定的隔离级别不需要事务，那么开启一个空事务，也不会从数据源中获取连接，绑定连接到线程
     * @Param definition 定义了事务的一些特征，如传播行为，隔离级别等。
     */
    TransactionStatus getTransaction(TransactionDefinition definition);
    
    /**
     * 提交一个事务
     */
    void commit(TransactionStatus status)；
    
    /**
     * 回滚一个事务
     */
    void rollback(TransactionStatus status)；
    
}
```

### TransactionDefinition

此接口主要用来定义事务的一些属性(传播行为、隔离级别、事务超时时间、事务只读)，接口中定义了一组get方法来获取这些属性，并且定义了7个传播行为。

* **PROPAGATION_REQUIRED**：如果当前存在事务，则加入该事务，如果当前不存在事务，则创建一个新的事务。
* **PROPAGATION_SUPPORTS**：如果当前存在事务，则加入事务，如果当前不存在事务，则以非事务运行。
* **PROPAGATION_MANDATORY**：强制当前必须存在一个事务，在该事务中运行，否则抛出异常。
* **PROPAGATION_REQUIRES_NEW**：创建一个新的事务，如果当前存在一个事务，挂起当前事务，这两个事务不相干。
* **PROPAGATION_NOT_SUPPORTED**：不支持事务，总是以非事务运行，如果当前存在事务，挂起当前事务，获取一个新的连接来执行。
* **PROPAGATION_NEVER**：不支持事务，如果存在事务则抛出异常。
* **PROPAGATION_NESTED**：嵌套事务，如果当前不存在事务，则行为与PROPAGATION_REQUIRED一致。采用JDBC的保存点(Savepoint)实现。

**DefaultTransactionDefinition**是一个默认的实现类，传播行为默认为PROPAGATION_REQUIRED，隔离级别默认采用数据库设置的隔离级别，只读默认为false。

### TransactionStatus

```java
/**
 * 此接口定义事务的状态，此接口获取的值会影响commit和rollback的行为。
 */
public interface TransactionStatus extends SavepointManager, Flushable {

	/**
	 * 判断此事务是不是一个新的事务，因为受传播行为影响，会参与到一个已经存在的事务
	 * 那么结果就不是一个新事务。
	 */
	boolean isNewTransaction();

	/**
	 * 判断是否有保存点（嵌套事务使用）
	 */
	boolean hasSavepoint();

	/**
	 * 设置该事务的结果只有回滚这一行为，当设置为true后，即便调用commit方法也会发生回滚。
	 */
	void setRollbackOnly();
	boolean isRollbackOnly();

	/**
	 * 事务是否完成，事务提交或者回滚后，事务就完成了。
	 */
	boolean isCompleted();

}

```

### DefaultTransactionStatus

```java
public abstract class AbstractTransactionStatus implements TransactionStatus {

    /**
     * 设置为true，那么事务只会有一种结果那就是回滚
     * 设置了此属性，即使调用commit也会执行回滚
     */
	private boolean rollbackOnly = false;

    /**
     * 事务状态
     */
	private boolean completed = false;

    /**
     * 保存点, 用于嵌套事务
     */
	private Object savepoint;
}

public class DefaultTransactionStatus extends AbstractTransactionStatus {

    /**
     * 代表一个事务对象
     * DataSourceTransactionManager事务管理器使用的是其内部类DataSourceTransactionObject
     */
	private final Object transaction;

    /**
     * 代表是不是一个新事务(空事务或真实存在的事务)
     * 注: 值为true时不代表开启一个真实存在的事务
     * 当指定事务传播行为以非事务方式运行的时候，Spring也会将这个值置为true，但是transaction为null
     * 以下几种情况newTransaction = true
     * 1. 开启一个真实存在的事务，必须是新的，参与外部事务的不算。 (transaction ！= null)
     * 2. 开启一个空事务，必须是新开的。 （transaction = null）
     *  2.1 不存在外部真实事务
     *  传播行为可能的值: PROPAGATION_NOT_SUPPORTED，PROPAGATION_SUPPORTS,PROPAGATION_NEVER
     *  2.2 存在外部真实事务
     *  传播行为的值: PROPAGATION_NOT_SUPPORTED(因为会将外部事务挂起，以非事务方式运行)
     *  ==========================================================================
     * 判断是否存在一个新的真实事务: newTransaction == true && transaction != null
     * 判断存在一个空事务(非事务执行方式): newTransaction == true && transaction != null
     * 判断此事务是否参与到外部事务: newTransaction == false && transaction != null
     * 判断存在一个事务(新的真实事务或者参与外部事务)：transaction != null
     */
	private final boolean newTransaction;

    /** 
     * 值为true，事务管理器需要为当前线程绑定当前事务的属性以及TransactionSynchronization回调接口
     * 这个值会受到AbstractPlatformTransactionManager类中定义的transactionSynchronization字段      * 影响。
     * transactionSynchronization可能的值为AbstractPlatformTransactionManager类定义的
     * 三个常量，默认为SYNCHRONIZATION_ALWAYS
     * 1. SYNCHRONIZATION_ALWAYS：总是激活事务同步(无论是空事务还是真实事务)
     * 2. SYNCHRONIZATION_ON_ACTUAL_TRANSACTION: 只有真实事务才激活事务同步
     * 3. SYNCHRONIZATION_NEVER：不激活事务同步
     * 这个字段的作用:
     * 如果为true，那么会将当前的事务属性(传播行为、隔离级别、事务名字、事务只读性、事务是否存活-用以区分      * 是空事务还是真实的事务)以及TransactionSynchronization回调接口
     * ==========================================================================
     * 在DataSourceTransactionManager事务管理器中并且transactionSynchronization为默认值时:
     * newSynchronization=true的情况
     * 1. 外部没有真实事务时
     *   1.1 调用事务接口的getTransaction方法开启一个事务(无论是空事务还是真实事务)
     * 2. 外部存在真实事务
     *   2.1 事务传播行为是PROPAGATION_NOT_SUPPORTED、PROPAGATION_REQUIRES_NEW时。
     *       因为这两个传播行为会挂起外部事务，而开一个新的事务(对应空事务，真实事务)
     */
	private final boolean newSynchronization;

    /**
     * 事务是否只读
     */
	private final boolean readOnly;
	/**
	 * 仅仅用来调试，打印日志，避免重复调用logger.isDebug()
	 */
	private final boolean debug;

    /**
     * 被挂起的资源，是一个SuspendedResourcesHolder对象
     */
	private final Object suspendedResources;
```

}

