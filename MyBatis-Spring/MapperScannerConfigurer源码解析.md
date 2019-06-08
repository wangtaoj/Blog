> **声明：源码基于mybatis-spring 1.3.2**

### 前文

首先在阅读本文前需要明白整合后的使用方式以及熟悉MyBatis本身的工作原理，再者如果对于本文相关知识点不熟悉的可以参考下述文章。

* [MyBatis与Spring整合](https://www.cnblogs.com/wt20/p/10938422.html)

* [SqlSessionTemplate源码解析](https://www.cnblogs.com/wt20/p/10963071.html)

* [Spring包扫描机制详解](https://www.cnblogs.com/wt20/p/10990697.html)

### 前言

一般在项目中使用MyBatis时，都会和Spring整合一起使用，通过注入一个Mapper接口来操纵数据库。其中的原理就是使用了MyBatis-Spring的`MapperScannerConfigurer`类，此类会将指定包下的接口生成代理类注册到Spring容器中。配置方式大致如下

```xml
<!-- 数据库连接池配置 -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <!-- 数据库驱动 -->
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <!-- 数据库地址 -->
    <property name="url" value="jdbc:mysql://127.0.0.1:3306/test"/>
    <!-- 数据库用户名 -->
    <property name="username" value="root"/>
    <!-- 数据库密码 -->
    <property name="password" value="123456"/>
</bean>
<!-- SqlSessionFactory配置 -->
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
<!-- 包扫描 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.wangtao.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
</bean>

```

经过以上配置MyBatis-Spring便会自动将com.wangtao.mapper包下接口生成代理类注册到Spring容器中，应用层代码只要注入接口就可以非常简单的执行数据库操作了。接下来分析下Mybatis-Spring包扫描的原理。

### 源码分析

#### MapperFactoryBean

在分析`MapperScannerConfigurer`之前有必要先讲下`MapperFactoryBean`这个及其重要的类，因为生成代理类的代码就在这个类中。当然熟悉MyBatis的人肯定知道，所谓的代理肯定就是MyBatis的接口绑定。对应的接口代理就是通过调用`sqlSession.getMapper()`方法生成的。

概貌，主要需要关注`checkDaoConfig`以及`getObject`两个方法。

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    /**
     * 需要代理的接口
     */
    private Class<T> mapperInterface;

    /**
     * 此字段的作用: 如果为true，指定的接口还没有被MyBatis解析过，也就是说MyBatis
     * 还没有对这个接口做绑定时，那么会去解析这个接口，并绑定该接口。
     * 这个接口可以采用MyBatis注解的方式书写SQL，也可以采用XML的经典方式。当然
     * 这个XML映射文件必须和Mapper接口同名并且在同一个包中。这点由addMapper方法决定的
     *
     * 对于SqlSessionFactoryBean配置的映射文件或者主配置文件中的<Mappers>节点配置的映射文件
     * 会在构造SqlSessioinFactory时就会根据映射文件的名称空间绑定好。
     */
  	private boolean addToConfig = true;
    
    public MapperFactoryBean(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }
    
    /**
     * 此方法在afterPropertiesSet调用
     * SqlSessionDaoSupport继承了DaoSupport，而DaoSupport实现了InitializingBean接口
     */
    @Override
    protected void checkDaoConfig() {
        // 父类SqlSessionDaoSupport会检查SqlSession是不是null
        super.checkDaoConfig();
		// 检查接口
        notNull(this.mapperInterface, "Property 'mapperInterface' is required");

        Configuration configuration = getSqlSession().getConfiguration();
        // 如果这个接口还没有被MyBatis解析过, 那么解析这个接口
        // addMapper会解析MyBatis注解方式，同时还会搜寻同包同名的XML映射文件完成绑定
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                configuration.addMapper(this.mapperInterface);
            } catch (Exception e) {
                logger.error("Error while adding the mapper '" + 
                             this.mapperInterface + "' to configuration.", e);
                throw new IllegalArgumentException(e);
            } finally {
                ErrorContext.instance().reset();
            }
        }
    }

    /**
     * 返回一个接口的代理实现
     */
    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }

    @Override
    public Class<T> getObjectType() {
        return this.mapperInterface;
    }
    
    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

此类继承了`SqlSessionDaoSupport`类并且实现了`FactoryBean`接口，我们可以知道`MapperFactoryBean`其实就是一个创建Mapper接口代理的工厂。其中`SqlSessionDaoSupport`需要传一个`SqlSessionFactory`实例来创建`SqlSession`或者直接从外部传入一个`SqlSessionTemplate`。

#### MapperScannerConfigurer

当明白了`MapperFactoryBean`的作用后，我们知道扫描接口时实际上需要注册的是一个个`MapperFactoryBean`的`BeanDefinition`，我们从容器中拿到的就是`MapperFactoryBean`中`getObject`方法返回的接口代理实现。

先看类中的字段

```java
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
    
    /** 需要扫描的包，多个包名可以使用逗号分割 **/
    private String basePackage;
	/** 同MapperFactoryBean **/
    private boolean addToConfig = true;
	/** 
	 * 被sqlSessionFactoryBeanName取代
	 * 被取代的原因放在总结里，如果好奇可以看下
	 */
    private SqlSessionFactory sqlSessionFactory;
	/** 被sqlSessionTemplateBeanName取代 **/
    private SqlSessionTemplate sqlSessionTemplate;
	/** SqlSessionFactory的名字 **/
    private String sqlSessionFactoryBeanName;
	/**
	 * sqlSessionTemplate的名字，创建MapperFactoryBean时需要依赖
     * 一个SqlSessionFactory或者sqlSessionTemplate实例。
	 */
    private String sqlSessionTemplateBeanName;

    /** 扫描条件，只有接口被此注解标注才会被注册，默认为null **/
    private Class<? extends Annotation> annotationClass;
	/** 扫描条件，只有接口继承了指定的这个接口才会被注册，默认为null **/
    private Class<?> markerInterface;

    private ApplicationContext applicationContext;

    /** 代表MapperScannerConfigurer这个bean的名字 **/
    private String beanName;

    /** 
     * 如果为true, 代表MapperScannerConfigurer配置使用了${}表达式
     * 需要先解析${}表达式
     * 因此如果配置MapperScannerConfigurer时有使用${}表达式，需要将这个值设置成true
     */
    private boolean processPropertyPlaceHolders;

    /**
     * 注册接口时使用这个字段来生成bean的名字
     */
    private BeanNameGenerator nameGenerator;
}
```

`MapperScannerConfigurer`实现了`BeanDefinitionRegistryPostProcessor`了接口，当我们想要注册bean时便可以实现这个接口，然后实现`void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)`方法来注册即可。如果对于`BeanDefinitionRegistryPostProcessor`不熟的话，恐怕需要学习下Spring容器初始化的流程了，这里我简单提下`ApplicaitonContext`容器大致关键的步骤。

1. 读取配置文件，将<bean>标签解析成一个个的`BeanDefinition`

2. 执行`BeanDefinitionRegistryPostProcessor`接口的`postProcessBeanDefinitionRegistry`方法。

3. 执行`BeanFactoryPostProcessor`接口的`postProcessBeanFactory`方法，在这里可以扩展修改第一步的`BeanDefinition`。

4. 初始化bean实例(调用构造方法)。

5. 调用setter方法设置属性。

6. 执行`BeanPostProcessor`接口的`postProcessBeforeInitialization`方法。

7. 执行`InitializingBean`接口的`afterPropertiesSet`方法。

8. 执行bean的init-method。

9. 执行`BeanPostProcessor`接口的`postProcessAfterInitialization`方法。

从上述容器的初始化流程中，可知Spring解析完配置文件生成一个个`BeanDefinition`后，便会调用所有实现了`BeanDefinitionRegistryPostProcessor`接口的bean中的`postProcessBeanDefinitionRegistry`方法。因此`MapperScannerConfigurer`中的`postProcessBeanDefinitionRegistry`将会得到执行。值得注意的一个点就是对于实现了`BeanDefinitionRegistryPostProcessor`或者`BeanFactoryPostProcessor`的bean来说，要向执行这两个接口的方法，需要先有bean的实例。因此会先创建实例，这样就会导致4-9这些步骤提早触发。因此`MapperScannerConfigurer`的执行流程为

* 构造方法实例化
* 调用setter方法设置属性
* 调用`setBeanName`以及`setApplicationContext`初始化`beanName`以及`applicationContext`属性，Aware接口里的方法是由`ApplicationContextAwareProcessor`回调的，`ApplicationContextAwareProcessor`是一个`BeanPostProcessor`，回调逻辑是在`postProcessBeforeInitialization`方法中
* 执行`afterPropertiesSet`方法
* 执行`postProcessBeanDefinitionRegistry`方法
* 执行`postProcessBeanFactory`方法

```java
@Override
public void afterPropertiesSet() throws Exception {
    // 检查basePackage参数，这个参数是必须的。
    notNull(this.basePackage, "Property 'basePackage' is required");
}

@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
        // 处理占位符，processPropertyPlaceHolders默认为false
        // 因为MapperScannerConfigurer先于Spring中处理占位符的BeanFactoryPostProcessor执行
        processPropertyPlaceHolders();
    }

    // 创建一个扫描器，这个类下面细讲
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    // 一系列设置方法，作用在讲字段的时候说了
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    // 注册TypeFilter，也就是筛选条件
    scanner.registerFilters();
    // 扫描包，多个包名可以使用逗号分割
    scanner.scan(StringUtils.tokenizeToStringArray(
        this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}

private void processPropertyPlaceHolders() {
    // 获取容器中所有的PropertyResourceConfigurer实例，用以解析MapperScannerConfigurer配置
    // 的${}表达式，因为MyBatis-Spring并不知道具体是哪一个专门来解析MapperScannerConfigurer配置
    // 的${}表达式。
    Map<String, PropertyResourceConfigurer> prcs = applicationContext
        .getBeansOfType(PropertyResourceConfigurer.class);
    if (!prcs.isEmpty() && 
        applicationContext instanceof ConfigurableApplicationContext) {
        // 获取MapperScannerConfigurer对应的BeanDefinition
        BeanDefinition mapperScannerBean = ((ConfigurableApplicationContext) 
            applicationContext).getBeanFactory().getBeanDefinition(beanName);

        // 创建一个新的DefaultListableBeanFactory，作用是用来作为
        // postProcessBeanFactory方法的参数。
        DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
        // 将MapperScannerConfigurer对应的BeanDefinition注册到factory中
        factory.registerBeanDefinition(beanName, mapperScannerBean);
		// 上面这两步的作用是为接下来解析${}表达式做准备
        // 处理${}表达式
        for (PropertyResourceConfigurer prc : prcs.values()) {
            prc.postProcessBeanFactory(factory);
        }
		// 经过上述操作beanDefinition里的属性将会被更新
        PropertyValues values = mapperScannerBean.getPropertyValues();
		// 根据PropertyValues更新basePackage、sqlSessionFactoryBeanName
        // sqlSessionTemplateBeanName的值。为什么只需要更新这三个?
        // 是因为创建MapperFactoryBean实例时需要这三个值。
        // basePackage用来扫描接口
        // sqlSessionFactoryBeanName与sqlSessionTemplateBeanName的值二选一，用来创建
        // SqlSession。
        this.basePackage = updatePropertyValue("basePackage", values);
        this.sqlSessionFactoryBeanName = updatePropertyValue(
            "sqlSessionFactoryBeanName", values);
        this.sqlSessionTemplateBeanName = updatePropertyValue(
            "sqlSessionTemplateBeanName", values);
    }
}
```

接下来便要讲非常重要的扫描类，`MapperScannerConfigurer`将扫描以及注册的功能全都委托给这个类了。

先看定义，如果对于`ClassPathBeanDefinitionScanner`不熟，可以看下前文小节中的**Spring包扫描机制详解**

```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
    private boolean addToConfig = true;

    private SqlSessionFactory sqlSessionFactory;

    private SqlSessionTemplate sqlSessionTemplate;

    private String sqlSessionTemplateBeanName;

    private String sqlSessionFactoryBeanName;

    private Class<? extends Annotation> annotationClass;

    private Class<?> markerInterface;

    private MapperFactoryBean<?> mapperFactoryBean = new MapperFactoryBean<Object>();
}
```

```java
// 构造方法
public ClassPathMapperScanner(BeanDefinitionRegistry registry) {
    // 禁用默认的includeFilters
    super(registry, false);
}

// 注册TypeFilter
public void registerFilters() {
    boolean acceptAllInterfaces = true;
	// 添加一个AnnotationTypeFilter，需要接口被此注解标注
    if (this.annotationClass != null) {
        addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
        acceptAllInterfaces = false;
    }
    // 添加一个AssignableTypeFilter，需要接口继承markerInterface
    // AssignableTypeFilter会接受参数本身以及参数的派生类
    if (this.markerInterface != null) {
        addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
            // 这里是去掉markerInterface自己
            @Override
            protected boolean matchClassName(String className) {
                return false;
            }
        });
        acceptAllInterfaces = false;
    }
    
    // 如果没有指定TypeFilter筛选，注册一个接受报下所有接口的TypeFilter
    if (acceptAllInterfaces) {
        addIncludeFilter(new TypeFilter() {
            @Override
            public boolean match(MetadataReader metadataReader, 
                                 MetadataReaderFactory metadataReaderFactory) 
                throws IOException {
                return true;
            }
        });
    }
    // exclude package-info.java
    addExcludeFilter(new TypeFilter() {
        @Override
        public boolean match(MetadataReader metadataReader, 
                             MetadataReaderFactory metadataReaderFactory) 
            throws IOException {
            String className = metadataReader.getClassMetadata().getClassName();
            return className.endsWith("package-info");
        }
    });
}

/**
 * 重写此方法，只筛选指定包下的顶层接口。
 * super.doScan方法中的findCandidateComponents方法会用到
 */
@Override
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    return beanDefinition.getMetadata().isInterface() 
        && beanDefinition.getMetadata().isIndependent();
}

/**
 * 重写此方法，super.doScan方法会用到
 * 只是打印下日志
 */
@Override
protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) {
    if (super.checkCandidate(beanName, beanDefinition)) {
        return true;
    } else {
        logger.warn("Skipping MapperFactoryBean with name '" + beanName 
                    + "' and '" + beanDefinition.getBeanClassName() + "' mapperInterface"
                    + ". Bean already defined with the same name!");
        return false;
    }
}
```

接下来就是最重要的`scan`方法

```java
// 这个完全就是父类ClassPathBeanDefinitionScanner的方法，以前讲过，因此不再展开说了，
// 主要关注子类重写的doScan方法
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    doScan(basePackages);
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 调用父类ClassPathBeanDefinitionScanner的doScan方法
    // 获取所有接口的BeanDefinition，此时这些BeanDefinition已经被注册到容器中了
    // 但是还没有实例化，因此可以对这些BeanDefinition做些修改
    // 接下来就要对一个个的BeanDefinition加工了
    // 将这一个个的BeanDefinition转成代表BeanFactoryBean的BeanDefinition
    // BeanFactoryBean会为接口生成代理实现
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
    if (beanDefinitions.isEmpty()) {
        logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + 
                    "' package. Please check your configuration.");
    } else {
        processBeanDefinitions(beanDefinitions);
    }
    return beanDefinitions;
}

private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
        definition = (GenericBeanDefinition) holder.getBeanDefinition();
        if (logger.isDebugEnabled()) {
            logger.debug("Creating MapperFactoryBean with name '" + 
                         holder.getBeanName() + "' and '" + 
                         definition.getBeanClassName() + "' mapperInterface");
        }
        // 构造方法注入接口名字，此时definition.getBeanClassName()获取的是
        // 接口的完全限定名，如果看的仔细点知道MapperFactoryBean构造方法参数是
        // 接口Class对象，为什么传字符串就行了呢? 这是Spring在初始化实例时会将字符串
        // 根据PropertyEditor转化规则转成相对应的类型，将String -> Class
        // 的PropertyEditor实现ClassEditor.
        // 在初始化SqlSessionFactoryBean时指定映射文件，也用的是字符串，但实际类型是Resource
        // 将String -> Resource是ResourceEditor的功能
        definition.getConstructorArgumentValues()
            .addGenericArgumentValue(definition.getBeanClassName()); // issue #59
        // 修改beanClass为MapperFactoryBean.class
        definition.setBeanClass(this.mapperFactoryBean.getClass());
        // 属性注入addToConfig
        definition.getPropertyValues().add("addToConfig", this.addToConfig);

        boolean explicitFactoryUsed = false;
        // 属性注入sqlSessionFactory
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
            // 加入this.sqlSessionFactoryBeanName的值为sqlSessionFactory，那么
            // 相当于<property="sqlSessionFactory" ref="sqlSessionFactory" />
            definition.getPropertyValues().add("sqlSessionFactory", 
                   new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
            definition.getPropertyValues()
                .add("sqlSessionFactory", this.sqlSessionFactory);
            explicitFactoryUsed = true;
        }

        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
            // 同时使用sqlSessionTemplate、sqlSessionFactory时，根据sqlSessionFactory
            // 创建的sqlSession将被覆盖掉，这点可以去看MapperFactoryBean父类
            // SqlSessionDaoSupport的setSqlSessionFactory以及setSqlSessionTemplate方法
            if (explicitFactoryUsed) {
                logger.warn("Cannot use both: sqlSessionTemplate" + 
                   "and sqlSessionFactory together. sqlSessionFactory is ignored.");
            }
            definition.getPropertyValues().add("sqlSessionTemplate",
                                               new RuntimeBeanReference(
                                                   this.sqlSessionTemplateBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
            if (explicitFactoryUsed) {
               logger.warn("Cannot use both: sqlSessionTemplate" + 
                   "and sqlSessionFactory together. sqlSessionFactory is ignored.");
            }
            definition.getPropertyValues()
                .add("sqlSessionTemplate", this.sqlSessionTemplate);
            explicitFactoryUsed = true;
        }
		// SqlSessionFactory和sqlSessionTemplate都没有指定，开启自动根据类型注入。
        if (!explicitFactoryUsed) {
            if (logger.isDebugEnabled()) {
                logger.debug("Enabling autowire by type for MapperFactoryBean with"
                            + "name '" + holder.getBeanName() + "'.");
            }
            definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }
    }
}
```

到这里基本上将`MapperScannerConfigurer`源码分析完毕了。

### 总结

只要将源码跟下，基本就可以非常清楚MyBatis-Spring接口扫描的工作原理了。这里还是简单总结下思路吧。1. 1. **1. `MapperScannerConfigurer`是一个包扫描的配置类，其中`basePackage`属性是必需的，多个包时可以使用逗号隔开。`sqlSessionFactoryBeanName`、`sqlSessionTemplateBeanName`二选一即可。如果都选了那么根据`sqlSessionFactoryBeanName`引用的`SqlSessionFactory`创建的`SqlSession`会被覆盖掉。如果两个都没配置，将会根据类型自动装配，这种方式适合应用程序中只有一个`SqlSessionFactory`或者`SqlSessionTemplat`的bean存在。其它配置就不总结了。**

**2.  具体实现扫描以及注册接口的功能被委托给`ClassPathMapperScanner`类了。具体做法就是将Spring扫描接口后生成的`BeanDefinition`修改成一个表示`MapperFactoryBean`的`BeanDefinition`，而`MapperFactoryBean`是一个实现`FactoryBean`的特殊bean，相当于返回Mapper接口实例的工厂。具体返回接口实例的原理是MyBatis本身的接口绑定功能，底层其实是动态代理。**

**3. `MapperScannerConfigurer`中`sqlSessionFactory`以及`sqlSessionTemplate`字段被废除的原因?**

`MapperScannerConfigurer`实现了`BeanDefinitionRegistryPostProcessor`接口，而此接口的`postProcessBeanDefinitionRegistry`方法会先于处理${}占位符的bean运行。通常而言处理${}表达式的bean是`PropertyPlaceholderConfigurer`或者`PropertySourcesPlaceholderConfigurer`这两个`BeanFactoryPostProcessor`。通过前面分析我们知道`MapperScannerConfigurer`运行流程如下所示:

- 构造方法实例化
- 调用setter方法设置属性
- 调用`setBeanName`以及`setApplicationContext`初始化`beanName`以及`applicationContext`属性，Aware接口里的方法是由`ApplicationContextAwareProcessor`回调的，`ApplicationContextAwareProcessor`是一个`BeanPostProcessor`，回调逻辑是在`postProcessBeforeInitialization`方法中
- 执行`afterPropertiesSet`方法
- 执行`postProcessBeanDefinitionRegistry`方法
- 执行`postProcessBeanFactory`方法

如果实例化`MapperScannerConfigurer`时需要依赖`SqlSessionFactory`或者`SqlSessionTemplate`时，那么需要调用容器的`getBean`方法提前初始化`SqlSessionFactory`或者`SqlSessionTemplate`，这时如果这两个bean配置时使用了${}表达式，那么${}表达式还没有解析成真正的值，就会出现错误。`SqlSessionFactory`或者`SqlSessionTemplate`的创建需要依赖于数据库连接池，一般连接池的配置都会采用${}表达式来配置的。而如果采用`sqlSessionFactoryBeanName`或者`sqlSessionTemplateBeanName`的配置就不会提前触发实例化`SqlSessionFactory`或者`SqlSessionTemplate`，也就不会出现上述问题了。

