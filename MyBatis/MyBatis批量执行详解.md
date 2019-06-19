> 源码基于MyBatis 3.4.6

### 如何使用

MyBatis内部提供了批量执行SQL的功能，当然这也只是对JDBC的一个包装。在介绍MyBatis中如何使用批量功能前，先来段原生的JDBC代码，看看如何执行一个批量SQL。大多数使用批量执行功能时，大多数都是对同一条SQL语句反复执行插入、更新、删除，只是传递的参数不一致。在接下来的代码中我将向MySQL中批量插入100000条数据。

```java
/**
 * 实体类
 */
@Getter
@Setter
public class Example {

    private Integer id;

    private String name;

    private Integer age;
}

/**
 * 测试类
 */
public class ExampleMapperTest {

	// 准备100000条数据
    private List<Example> prepareData() {
        List<Example> examples = new ArrayList<>();
        Random random = new Random();
        for(int i = 1; i <= 100000; i++) {
            Example example = new Example();
            example.setId(i);
            example.setName("example-" + i);
            example.setAge(random.nextInt(101));
            examples.add(example);
        }
        return examples;
    }

    // 执行批量插入
    @Test
    public void testJDBCBatch() throws SQLException {
        String url = "jdbc:mysql://127.0.0.1:3306/test";
        long start = System.currentTimeMillis();
        Connection conn = DriverManager.getConnection(url, "root", "123456");
        List<Example> examples = prepareData();
        try {
            // 开启事务
            conn.setAutoCommit(false);
            PreparedStatement ps = conn.prepareStatement("INSERT INTO example(id, name, 									age) VALUES (?, ?, ?)");
            for(Example example : examples) {
                // 设置参数
                ps.setInt(1, example.getId());
                ps.setString(2, example.getName());
                ps.setInt(3, example.getAge());
                // 添加到批量语句集合中
                ps.addBatch();
            }
            // 执行批量操作
            int[] updateCounts = ps.executeBatch();
            // 提交事务
            conn.commit();
            long end = System.currentTimeMillis();
            System.out.println("批量执行耗时: " + (end - start) / 1000.0 + "s");
            for(int updateCount : updateCounts) {
                Assert.assertEquals(1, updateCount);
            }
        } catch (SQLException e) {
            conn.rollback();
            throw e;
        } finally {
            conn.close();
        }
    }
}

```

我对上述批量插入操作进行了一个简单的测试，并且统计了执行时间，在MySQL5.6中，批量插入100000条数据大约需要10秒左右。

接下来看看如何在MyBatis中如何执行批量操作，上面用到的实体类就不贴了。

```java
/**
 * 获取SqlSession的工具类
 */
public class MyBatisUtils {

    private final static SqlSessionFactory sqlSessionFactory;

    static{
        try {
            Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        } catch (IOException e) {
            throw new RuntimeException("sqlSessionFactory init fail.", e);
        }
    }

    public static SqlSession getSqlSession(ExecutorType executorType) {
        return sqlSessionFactory.openSession(executorType);
    }
}

/**
 * Mapper接口，为了方便直接使用注解的方式。
 */
public interface ExampleMapper {

    @Insert(value = "INSERT INTO example(id, name, age) VALUES (#{id}, #{name}, #{age})")
    int insert(Example example);
}

/**
 * 测试类
 */
public class ExampleMapperTest {

    @Test
    public void testBatch() {
        // 准备数据，上面已经贴过了。
        List<Example> examples = prepareData();
        // 关键，如果使用批量功能，需要使用BatchExecutor而不是默认的SimpleExecutor
        try (SqlSession session = MyBatisUtils.getSqlSession(ExecutorType.BATCH)) {
            ExampleMapper exampleMapper = session.getMapper(ExampleMapper.class);
            for(Example example : examples) {
                // 因为使用的Executor是BatchExecutor, 并不会真的执行
                // 内部调用的是statement.addBatch()
                exampleMapper.insert(example);
            }
            // 执行批量操作, 内部调用的是statement.executeBatch()
            // 如果不需要返回值可以不用显示调用，commit方法内部会调用此方法
            List<BatchResult> results = session.flushStatements();
            session.commit();
            Assert.assertEquals(1, results.size());
            BatchResult result = results.get(0);
            for(int updateCount : result.getUpdateCounts()) {
                Assert.assertEquals(1, updateCount);
            }
        }
    }
}

```

这里可能需要说明的是`flushStatements`方法了，此方法定义在`SqlSession`接口中，签名如下

```JAVA
List<BatchResult> flushStatements();
```

此方法的作用就是将前面所有执行过的INSERT、UPDATE、DELETE语句真正刷新到数据库中。底层调用了JDBC的`statement.executeBatch`方法。这里有疑惑的肯定是这个方法的返回值了。通俗的说如果执行的是同一个方法并且执行的是同一条SQL，注意这里的SQL还没有设置参数，也就是说SQL里的占位符**'?'**还没有被处理成真正的参数，那么每次执行的结果共用一个`BatchResult`，真正的结果可以通过`BatchResult`中的`getUpdateCounts`方法获取。

按照上面例子，执行的是`ExampleMapper`中的`insert`方法，执行的SQL语句是`INSERT INTO example(id, name, age) VALUES (?, ?, ?)`。这100000万次执行的都是同一个方法同一条SQL语句。那么返回值中只有一个`BatchResult`，`getUpdateCounts`返回的数组大小是100000，代表着每一次执行的结果。

### 源码分析

`SqlSession`这个接口操作数据库的功能都是`Executor`接口实现的，而批量功能正是由上面提到过的`BatchExecutor`实现。`SqlSession`接口中的`update`、`insert`、`delete`最终都会调用`Executor`的`update`方法。

```java
// 位于父类BaseExecutor中，BatchExecutor继承了BaseExecutor
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    // 清空本地缓存
    clearLocalCache();
    // 对数据库做INSERT、UPDATE、DELETE操作，由子类实现
    return doUpdate(ms, parameter);
}
```

接下来看`BatchExecutor`

```java
public class BatchExecutor extends BaseExecutor {

  public static final int BATCH_UPDATE_RETURN_VALUE = Integer.MIN_VALUE + 1002;
  /**
   * Statement集合，如果执行的方法不一样或者SQL语句不同
   * 都会创建一个Statement
   */
  private final List<Statement> statementList = new ArrayList<Statement>();
  /**
   * 与上面一一对应，用来保存每一个statement的执行结果。
   */
  private final List<BatchResult> batchResultList = new ArrayList<BatchResult>();
  // 当前的SQL语句
  private String currentSql;
  // XML中的每一个<insert><update><delete><select>标签最终被解析成
  // 一个个的MappedStatement对象，该对象的id就是名称空间+标签id
  // 也就是所说的接口完全限定名 + "." + 方法名字
  private MappedStatement currentStatement;

  public BatchExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
  }

  @Override
  public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    // 获取需要执行的SQL，这里的SQL中参数都已经被解析成?占位符了
    // 就是#{id, jdbcType=INTEGER}  ->  ?
    final String sql = boundSql.getSql();
    final Statement stmt;
    // 如果SQL相同并且是同一个<update> | <insert> | <delete>标签
    // 为什么还需要判断<update> | <insert> | <delete>，因为这些标签配置的属性
    // 会影响具体的执行行为。
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
      int last = statementList.size() - 1;
      stmt = statementList.get(last);
      applyTransactionTimeout(stmt);
      // 设置参数
      handler.parameterize(stmt);//fix Issues 322
      BatchResult batchResult = batchResultList.get(last);
      batchResult.addParameterObject(parameterObject);
    } else {
      // 初始化Statement
      Connection connection = getConnection(ms.getStatementLog());
      stmt = handler.prepare(connection, transaction.getTimeout());
      handler.parameterize(stmt);    //fix Issues 322
      currentSql = sql;
      currentStatement = ms;
      statementList.add(stmt);
      batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    // 调用addBatch方法
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
  }
}
```

下面来看看`flushStatements`方法

```java
// 位于父类BaseExecutor中，BatchExecutor继承了BaseExecutor
@Override
public List<BatchResult> flushStatements() throws SQLException {
    return flushStatements(false);
}

/**
 * isRollBack: 如果事务发生回滚，此参数的值设置成true
 */
public List<BatchResult> flushStatements(boolean isRollBack) throws SQLException {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    return doFlushStatements(isRollBack);
}

@Override
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
        List<BatchResult> results = new ArrayList<BatchResult>();
        // 直接返回
        if (isRollback) {
            return Collections.emptyList();
        }
        for (int i = 0, n = statementList.size(); i < n; i++) {
            Statement stmt = statementList.get(i);
            applyTransactionTimeout(stmt);
            BatchResult batchResult = batchResultList.get(i);
            try {
                // 执行批量操作，并将结果保存到batchResult中
                batchResult.setUpdateCounts(stmt.executeBatch());
                MappedStatement ms = batchResult.getMappedStatement();
                List<Object> parameterObjects = batchResult.getParameterObjects();
                // 以下仅针对insert语句，用于返回自增主键
                KeyGenerator keyGenerator = ms.getKeyGenerator();
                if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
                    Jdbc3KeyGenerator jdbc3KeyGenerator = 
                        (Jdbc3KeyGenerator) keyGenerator;
                    jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
                } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { 
                    for (Object parameter : parameterObjects) {
                        keyGenerator.processAfter(this, ms, stmt, parameter);
                    }
                }
                closeStatement(stmt);
            } catch (BatchUpdateException e) {
                // 省略异常处理
            }
            results.add(batchResult);
        }
        return results;
    } finally {
        for (Statement stmt : statementList) {
            closeStatement(stmt);
        }
        // 清空
        currentSql = null;
        statementList.clear();
        batchResultList.clear();
    }
}

```

### 总结

本文主要讲解了如何在MyBatis中使用批量操作的功能，并且对源码的关键位置进行了说明。下面主要总结下MyBatis中批量处理的行为。

* 如果执行了SELECT操作，那么会将先前的UPDATE、INSERT、DELETE语句刷新到数据库中。这一点去看`BatchExecutor`中的`doQuery`方法即可。
* MyBatis会在`commit`或`rollback`方法中调用`flushStatements`刷新语句。其中`commit`会调用`flushStatements()`，而`rollback`会调用`flushStatements(true)`，也就是如果回滚那就不再执行批量操作了。因此即使没有显示调用`flushStatements`方法，MyBatis也会保证批量操作正常执行。

