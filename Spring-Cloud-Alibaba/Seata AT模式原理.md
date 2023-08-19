> Seata 1.6.1

### 可参考文档
[官网Seata AT模式](http://seata.io/zh-cn/docs/dev/mode/at-mode.html)
[官网Seata AT实现博客](https://seata.io/zh-cn/blog/seata-at-lock.html)

### 基本术语
TC (Transaction Coordinator) - 事务协调者
维护全局和分支事务的状态，驱动全局事务提交或回滚。就是Seata Server。

TM (Transaction Manager) - 事务管理器
定义全局事务的范围：开始全局事务、提交或回滚全局事务。可以理解为`GlobalTransactional`标注的地方

RM (Resource Manager) - 资源管理器
管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

### 基本原理

AT模式主要分为2个阶段，也是一个2阶段提交模式
**一阶段**
* 本地事务执行数据库SQL，此时可能会获取本地行锁，看具体执行的SQL(MySQL自己支持)，并解析语句执行之前的数据镜像(before image)，以及执行之后的数据镜像(after image)保存到undo_log表中。before image用于全局事务回滚时的恢复，after image用户回滚时的比对，如果回滚时发现当前数据库的数据与after image不一致，说明中间有别的操作修改，此时便无法直接回滚了。(**undo_log日志与业务SQL在同一个本地事务执行**)
* 本地事务提交之前，需要拿到全局锁，才能进行提交。在提交之前，向TC(事务协调器)注册分支事务，获取改动数据的全局锁。获取全局锁的尝试被限制在一定范围内，超出范围将放弃，并回滚本地事务，释放本地锁。

**二阶段提交**
- 收到 TC 的分支提交请求，把请求放入一个异步任务的队列中，马上返回提交成功的结果给 TC。
- 异步任务阶段的分支提交请求将异步和批量地删除相应 UNDO LOG 记录。

**二阶段回滚**
* 收到 TC 的分支回滚请求，开启一个本地事务，执行如下操作。
* 通过 XID(全局事务ID) 和 Branch ID(分支事务ID)查找到相应的 UNDO LOG 记录。
* 数据校验：拿 UNDO LOG 中的after image与当前数据进行比较，如果有不同，说明数据被当前全局事务之外的动作做了修改。这种情况，需要根据配置策略来做处理。
* 根据 UNDO LOG 中的before image和业务 SQL 的相关信息生成并执行回滚的语句。
* 提交本地事务。并把本地事务的执行结果（即分支事务回滚的结果）上报给 TC。

### 写隔离

结合官网例子，记录自己的思考，**假设事务A和事务B都是全局事务的一个分支事务**，如果没有全局锁的存在，由于本地事务A在一阶段就已经提交了，那么本地事务A持有的行锁也会释放掉，那么别的事务B就能随意修改数据进行提交。此时事务A所在的全局事务通知本地事务A回滚，这个回滚操作就无法执行了，因为数据发生了变更，**而事务B也相当于发生了脏写，因为操作的数据是在事务A执行后的结果。** 如余额初始值为100，事务A扣减10，变成了90，事务B扣减10，变成了80，但是事务A是要回滚的，正常应该是事务B从100扣减10，最终结果为90。

如果有全局事务的存在，本地事务B修改了数据进行提交(**持有本地行锁**)，但是事务B它需要获取全局锁才能提交，而本地事务A所在的全局事务二阶段还没结束，所以本地事务B是申请不到全局锁的，这个时候事务A所在的全局事务通知本地事务A回滚，而回滚操作肯定是要先获取数据的行锁的，此时锁的持有情况，**事务A持有全局锁，想要申请数据的本地行锁，事务B持有数据的本地行锁，想要申请全局锁。** 即发生死锁，而申请全局锁的的尝试时间要短的多，**超过时间后将放弃申请全局锁，并回滚本地事务，释放本地行锁，这时事务A便能拿到本地行锁进行回滚，然后释放全局锁。**

**根据上述分析，必须是两个Seata全局事务去操作，才能避免脏写，如果另外一个事物不是Seata管理的事务，因为没有全局锁，会存在脏写问题。**

### @GlobalLock
与`@GlobalTransactional`注解不同，它不会开启一个全局事务，不会注册分支事务到TC中，但是有该注解时，之后的SQL也会被Seata进行代理，它的作用是本地事务提交前会检查下当前是否有全局锁的存在，如果存在全局锁则会抛异常，让本地事务回滚。**因此使用该注解也能解决上面的脏写问题，即一个Seata管理的全局事务和一个@GlobalLock注解的其它事务**。

相关逻辑可参见Seata的`ConnectionProxy`类，重写了commit方法
```java
@Override
public void commit() throws SQLException {
    try {
        lockRetryPolicy.execute(() -> {
            doCommit();
            return null;
        });
    } catch (SQLException e) {
        if (targetConnection != null && !getAutoCommit() && !getContext().isAutoCommitChanged()) {
            rollback();
        }
        throw e;
    } catch (Exception e) {
        throw new SQLException(e);
    }
}

private void doCommit() throws SQLException {
    // 在全局事务中, 会注册分支事务到TC中
    if (context.inGlobalTransaction()) {
        processGlobalTransactionCommit();
    } else if (context.isGlobalLockRequire()) {
        // 会检查是否有全局锁, 若存在默认抛异常
        processLocalCommitWithGlobalLocks();
    } else {
        // 执行原本逻辑
        targetConnection.commit();
    }
}
```

### SELECT FOR UPDATE

在数据库本地事务隔离级别 **读已提交（Read Committed）** 或以上的基础上，Seata（AT 模式）的默认全局隔离级别是 **读未提交（Read Uncommitted）** 。因为全局事务没有提交时，本地事务已经提交了，所以别的事务是可以看到数据的修改的。

如果应用在特定场景下，必需要求全局的 **读已提交** ，目前 Seata 的方式是通过 `SELECT FOR UPDATE` 语句的代理。

在全局事务或者@GlobalLock注解的方法中，Seata 会对`SELECT FOR UPDATE`语句进行代理，注意普通的select语句不会被代理。`SELECT FOR UPDATE` 语句的执行会申请 **全局锁** ，如果 **全局锁** 被其他事务持有，则释放本地锁（回滚 `SELECT FOR UPDATE` 语句的本地执行）并重试，再次执行该语句。
**同时回滚该`SELECT FOR UPDATE`语句时，采用了保存点的方式，避免把`SELECT FOR UPDATE`语句之前的操作也回滚了。**

SelectForUpdateExecutor.java
```java
@Override
public T doExecute(Object... args) throws Throwable {
    Connection conn = statementProxy.getConnection();
    DatabaseMetaData dbmd = conn.getMetaData();
    T rs;
    Savepoint sp = null;
    boolean originalAutoCommit = conn.getAutoCommit();
    try {
        if (originalAutoCommit) {
            conn.setAutoCommit(false);
        } else if (dbmd.supportsSavepoints()) {
            // 设置保存点
            sp = conn.setSavepoint();
        } else {
            throw new SQLException("not support savepoint. please check your db version");
        }

        LockRetryController lockRetryController = new LockRetryController();
        ArrayList<List<Object>> paramAppenderList = new ArrayList<>();
        String selectPKSQL = buildSelectSQL(paramAppenderList);
        // 死循环
        while (true) {
            try {
                // 执行目标select for update语句
                rs = statementCallback.execute(statementProxy.getTargetStatement(), args);

                // Try to get global lock of those rows selected
                TableRecords selectPKRows = buildTableRecords(getTableMeta(), selectPKSQL, paramAppenderList);
                String lockKeys = buildLockKey(selectPKRows);
                if (StringUtils.isNullOrEmpty(lockKeys)) {
                    break;
                }

                if (RootContext.inGlobalTransaction() || RootContext.requireGlobalLock()) {
                    // @GlobalTransactional or @GlobalLock, 
                    // 检查全局锁是否存在, 存在抛LockConflictException
                    statementProxy.getConnectionProxy().checkLock(lockKeys);
                } else {
                    throw new RuntimeException("Unknown situation!");
                }
                break;
            } catch (LockConflictException lce) {
                // 存在全局锁, 回滚到之前的保存点
                if (sp != null) {
                    conn.rollback(sp);
                } else {
                    conn.rollback();
                }
                // 休眠之后下一轮循环重试
                lockRetryController.sleep(lce);
            }
        }
    } finally {
        if (sp != null) {
            try {
                if (!JdbcConstants.ORACLE.equalsIgnoreCase(getDbType())) {
                    conn.releaseSavepoint(sp);
                }
            } catch (SQLException e) {
                LOGGER.error("{} release save point error.", getDbType(), e);
            }
        }
        if (originalAutoCommit) {
            conn.setAutoCommit(true);
        }
    }
    return rs;
}
```

### 全局事务ID管理

```java
// 获取全局事务ID
RootContext.getXID();

// 绑定全局事务
RootContext.bind(String xid);
```

这里有一个问题，不同的分支事务可能压根不在同一个应用中，别的分支事务如何知道自己处于一个全局事务中呢？

```java
/**
 * 开启全局事务, 主分支事务A
 */
@GlobalTransactional
@Override
public void purchase(PurchaseVO purchaseVO, boolean throwExp) {
    // 分支事务B(http通信)
    storageFeignClient.deduce();
    // 分支事务C(http通信)
    orderFeignClient.create();
}
```

Seata是通过拦截HTTP请求，将全局事务id设置到请求头中，别的处理器(SpringMVC)处理请求时从请求头中获取全局事务ID，然后绑定到`RootContext`中。
别的RPC请求方式同理。

Http请求拦截实现如下
AbstractHttpExecutor.java
```java
private <K> K wrapHttpExecute(Class<K> returnType, CloseableHttpClient httpClient, HttpUriRequest httpUriRequest,
            Map<String, String> headers) throws IOException {
    CloseableHttpResponse response;
    String xid = RootContext.getXID();
    if (xid != null) {
        // 发送请求前设置请求头
        headers.put(RootContext.KEY_XID, xid);
    }
    if (!headers.isEmpty()) {
        headers.forEach(httpUriRequest::addHeader);
    }
    response = httpClient.execute(httpUriRequest);
    int statusCode = response.getStatusLine().getStatusCode();
    /** 2xx is success. */
    if (statusCode < HttpStatus.SC_OK || statusCode > HttpStatus.SC_MULTI_STATUS) {
        throw new RuntimeException("Failed to invoke the http method "
                + httpUriRequest.getURI() + " in the service "
                + ". return status by: " + response.getStatusLine().getStatusCode());
    }

    return convertResult(response, returnType);
}
```

TransactionPropagationInterceptor.java

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    // 处理请求前, 绑定全局事务id
    String rpcXid = request.getHeader(RootContext.KEY_XID);
    return this.bindXid(rpcXid);
}

@Override
public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
    if (RootContext.inGlobalTransaction()) {
        // 处理请求后, 清理全局事务id
        String rpcXid = request.getHeader(RootContext.KEY_XID);
        this.cleanXid(rpcXid);
    }
}
```