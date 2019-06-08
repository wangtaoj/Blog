### 目标

本文主要目标是介绍MyBatis如何与Spring整合，MyBatis在Spring中使用的几种方式，为后面分析整合的源码奠定基础。

### 方式一. 使用SqlSessionTemplate

SqlSessionTemplate也是一个SqlSession实例，与sqlSessionFactory.openSession()方法返回的DefaultSqlSession实例，主要区别有两点

* SqlSessionTemplate是线程安全的，可以供多个DAO类共享，而DefaultSqlSession线程不安全。
* SqlSessionTemplate不支持事务API，调用rollbakc，commit方法会抛出异常，同时也不支持close方法，这些方法作用全都由Spring管理，会基于Spring 的事务配置来自动提交、回滚、关闭 session。

为了获取SqlSessionTemplate实例，我们需要一个SqlSessionFactory。我们可以借助mybatis-spring包提供的SqlSessionFactoryBean来获取一个SqlSessionFactory，MyBatis中的配置全都可以由这个类来配置。SqlSessionFactoryBean实现了FactoryBean，这是Spring中一种特殊的bean，当我们在XML配置中配置一个FactoryBean时，使用容器的getBean方法获取的对象实际是FactoryBean接口中getObject()返回的对象。

XML关键配置如下

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
    <property name="mapperLocations" value="classpath:mappers/AddressMapper.xml"/>
</bean>

<!-- SqlSessionTemplate配置 -->
<bean id="sessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg index="0" name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

测试代码

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(locations = "classpath:sqlsession/by-sqlsession.xml")
public class SqlSessionFactoryTest {

    /** 注入SqlSessionTemplate **/
    @Autowired
    private SqlSessionTemplate sqlSessionTemplate;

    @Test
    public void executeByTemplate() {
        // 注意不能显示调用commit、rollback、close方法
        AddressMapper addressMapper = sqlSessionTemplate.getMapper(AddressMapper.class);
        addressMapper.selectByPrimaryKey(1);
    }
}

```

可以看到此种方式需要手动获取SqlSession以及绑定的接口实例来执行数据库操作，比较麻烦。

### 方式二. 手动生成接口代理类

为了简化MyBatis的使用，mybatis-spring可以为我们创建一个接口的代理实现类交给Spring容器管理，这样我们便不再需要和SqlSession打交道，不再需要知道MyBatis的底层API。这时便需要MapperFactoryBean这个类出场了，看名字就知道它是一个创建Mapper接口的一个FactoryBean。

XML关键配置如下

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
    <property name="mapperLocations" value="classpath:mappers/AddressMapper.xml"/>
</bean>

<!-- 为指定接口生成代理实现类 -->
<bean id="addressMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
    <property name="mapperInterface" value="com.wangtao.spring.manual.AddressMapper"/>
    <property name="sqlSessionFactory" ref="sqlSessionFactory"/>
</bean>
```

测试代码

```java
@RunWith(SpringRunner.class)
@ContextConfiguration("classpath:manual/spring-manual.xml")
public class AddressMapperTest {

    /** 直接注入接口实例 **/
    @Autowired
    private AddressMapper addressMapper;

    @Test
    public void selectByPrimarykey() {
        addressMapper.selectByPrimaryKey(1);
    }
}
```

可以看到现在使用MyBatis操作数据库已经非常方便了，只需要一个接口以及映射文件就行了，但是如果Mapper接口非常多，配置生成接口代理就得一个一个去指定，显得很麻烦。

### 方式三. 自动扫描，生成接口代理类。

使用方式二，需要自己手动一个一个的去配置接口，但是我们可以指定一个包名，让MyBatis自动去扫描这个包下的接口，并生成代理实例放到Spring容器中。

XML关键配置如下

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <!-- 注入连接池 -->
    <property name="dataSource" ref="dataSource"/>
    <!--主配置 -->
    <property name="configuration">
        <bean class="org.apache.ibatis.session.Configuration">
            <!--下划线转驼峰-->
            <property name="mapUnderscoreToCamelCase" value="true"/>
        </bean>
    </property>
    <!-- 加载mybatis映射文件 -->
    <property name="mapperLocations" value="classpath:mappers/*.xml"/>
</bean>

<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!-- 指定需要扫描的包名 -->
    <property name="basePackage" value="com.wangtao.spring.autoscan"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
</bean>

```

测试代码

```java
@RunWith(SpringRunner.class)
@ContextConfiguration("classpath:autoscan/spring-scan.xml")
public class UserMapperTest {

    @Autowired
    private AddressMapper addressMapper;

    @Test
    public void selectByPrimarykey() {
        addressMapper.selectByPrimaryKey(1);
    }
}
```

### 总结

本文介绍了MyBatis与Spring的整合时使用的几种方式，可以看到与Spring整合后可以非常方便地管理事务，事务全都交给Spring管理，而且通过接口绑定的方式时完全不用知道MyBatis的API，关注业务代码即可。