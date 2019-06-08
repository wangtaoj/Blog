### 目的

此文的主旨在于梳理SqlSessionFactoryBean的初始流程，不拘泥于实现细节。

### 使用

`SqlSessionFactoryBean`的主要作用便是用来创建`SqlSessionFactory`，在它的构建过程中会解析`MyBatis`所需要的配置文件，将所有配置都封装到`Configuration`类中，最后返回`SqlSessionFactory实例`。`SqlSessionFactoryBean`实现了`Spring`中两个用于初始化Bean的接口`FactoryBean`、`InitializingBean`.

InitializingBean定义如下

```java
public interface InitializingBean {
	/**
	 * 此方法在BeanFactory设置了Bean配置里的所有属性后执行。
	 */
	void afterPropertiesSet() throws Exception;
}
```

我们一般在XML中都会有这么一段配置，如下:

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <!-- 注入连接池 -->
  <property name="dataSource" ref="dataSource"/>
  <!--主配置 -->
  <property name="configuration">
    <bean class="org.apache.ibatis.session.Configuration">
      <property name="mapUnderscoreToCamelCase" value="true"/>
    </bean>
  </property>
  <!-- 加载mybatis映射文件 -->
  <property name="mapperLocations" value="classpath:mappers/*.xml"/>
</bean>
```

上述配置的作用就是让容器创建一个ID为`sqlSessionFactory`的`SqlSessionFactoryBean`实例，并且为实例指定了`datasource`属性，手动指定了主配置对象，开启`MyBatis`下划线转驼峰的属性配置，并且指定映射文件所在的位置。

**注:**
**虽然指定的是`SqlSessionFactoryBean`实例，但是因为继承了`FactoryBean`接口，因此我们从容器拿到的对象实际上是`getObject`方法返回的对象，即`SqlSessionFactory`实例。**

### 创建流程

首先自不用说，Spring会创建`SqlSessionFactoryBean`实例，设置配置的所有属性，这是第一步，接下来便会走`afterPropertiesSet()`方法。

```java
@Override
public void afterPropertiesSet() throws Exception {
    // 检查数据源
    notNull(dataSource, "Property 'dataSource' is required");
    // 检查SqlSessionFactoryBuilder, 已经在构造方法里初始化
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    // configuration实例与configLocation只能有一个
    state((configuration == null && configLocation == null) || !(configuration != null &&          configLocation != null),
        "Property 'configuration' and 'configLocation' can not specified with together");
    // 解析配置文件，创建SqlSessionFactory。这属于MyBatis本身的内容，不再展开说明。
    this.sqlSessionFactory = buildSqlSessionFactory();
}
```

创建`SqlSessionFactory`的过程与`MyBatis`不同的点主要在与`transactionFactory`，在创建事务工厂时不再使用`MyBatis`自带的`JdbcTransaction`类，而是使用`SpringManagedTransactionFactory`这个新的事务工厂类，这个事务获取到的连接全部来自于Spring管理，这样把事务全权交给Spring管理，而`MyBatis`本身不再涉及事务管理。如果在使用Spring时没有配置(使用)事务，那么获取到的连接取决于`DataSource`，无论拿到的连接是自动提交还是手动提交，MyBatis每一次方法调用都会视情况提交或回滚。当然对于自动提交而言，MyBatis的`commit`或者`rollbakc`是不会调用`conn.commit()`或`conn.rollback`。

在经过`afterPropertiesSet`方法后，`SqlSessionFactory`实例便创建完毕。需要使用`SqlSessionFactory`实例便会调用`getObject`方法。

```java
@Override
public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
        afterPropertiesSet();
    }
    return this.sqlSessionFactory;
}
```



