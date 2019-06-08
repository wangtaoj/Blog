> **声明：源码基于4.3.18**

### 目标

此篇文章会主要介绍Spring中两个非常重要的关于包扫描的基础类，由于Spring代码太庞大，因此本文不会细致地说明每一行代码地作用，只会讲清楚关键的地方有什么作用，以及一些子类可以重写的方法，用来覆盖默认扫描行为。最后会基于Spring提供的包扫描设施来写一个简单的例子来模仿MyBatis-Spring扫描Mapper接口，生成代理注册到容器中。我们主要关注`ClassPathScanningCandidateComponentProvider`以及`ClassPathBeanDefinitionScanner`这两个类，讲清楚这两个类的作用以及开发者需要关注的方法。

### ClassPathScanningCandidateComponentProvider

此类是Spring中包扫描机制最底层的类，用于扫描指定包下面的类文件，并且会根据用户提供的`includeFilters`以及`excludeFilters`来过滤掉不想注册的类，最后生成一个基本的`BeanDefinition`。

先看下两个比较重要的属性吧

```java
/**
 * 包含集合，如果类文件匹配includeFilters集合中任意一个TypeFilter条件，那么就通过筛选。
 * 其中最常见的TypeFilter有AnnotationTypeFilter、AssignableTypeFilter
 * AnnotationTypeFilter: 代表类是否被指定注解标注
 * AssignableTypeFilter: 代表类是否继承(实现)自指定的超类
 */
private final List<TypeFilter> includeFilters = new LinkedList<TypeFilter>();

/**
 * 排除集合, 如果类文件匹配excludeFilters集合中任何一个TypeFilter条件，那么就不会通过筛选
 * 并且excludeFilters优先级高于includeFilters
 */
private final List<TypeFilter> excludeFilters = new LinkedList<TypeFilter>();
```

初始化，只看参数最长的那个构造方法

```java
public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters, 
                                                   Environment environment) {
    // 主要关注useDefaultFilters这个参数, 如果为true, 会注册一个默认的
    // includeFilter
    if (useDefaultFilters) {
        registerDefaultFilters();
    }
    setEnvironment(environment);
    setResourceLoader(null);
}

protected void registerDefaultFilters() {
    // 注册一个注解的TypeFilter，意思是如果类定义时有被@Component注解标注
    // 那么就会通过筛选，需要注意的是衍生注解也是会通过筛选的
    // 比如@Service、@Controller、@Repository，它们有一个共同点，那就是这三个
    // 注解本身就是被@Component注解标注的
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));
    // 下面是注册JAVA EE里的一些注解，一般开发也不用, 就省略了
}
```

接下来就是最重要的方法了，也就是扫描包文件，并且将符合筛选条件的类生成`BeanDefinition`。下面代码将日志打印剔除了。

```java
/**
 * 此方法会扫描指定包以及子包
 * @param basePackage 需要扫描的包，形如com.wangtao.dao
 * @return 返回一个符合筛选条件后的BeanDefinition集合
 */
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
    try {
        // 将包名解析成路径
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        // 获取此包以及子包下的所有.class文件资源
        Resource[] resources = this.resourcePatternResolver.
            getResources(packageSearchPath);
        for (Resource resource : resources) {
            if (resource.isReadable()) {
                try {
                    // 得到类文件的元数据，是基于ASM字节码技术实现的，此时还没有加载类文件
                    // 包括类名、注解信息、父类、接口等等一系列信息
                    MetadataReader metadataReader = this.metadataReaderFactory.
                        getMetadataReader(resource);
                    // 匹配筛选条件，也就是上述includeFilters、excludeFilters这两个集合
                    if (isCandidateComponent(metadataReader)) {
                        // 创建一个BeanDefinition
                        // 只是简单的设置了beanClassName属性为类的完全限定名
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBean
                            Definition(metadataReader);
                        sbd.setResource(resource);
                        sbd.setSource(resource);
                        // 这里会再有一个筛选条件，一般是根据类文件的元数据筛选
                        // 比如是不是具体类，是不是顶层类，是不是抽象类等
                        // 默认情况下只添加顶层的具体类，顶层的意思是可以独立实例化而不会依赖外部类
                        // 成员内部类需要外部类对象才能实例化，就不会通过。
                        if (isCandidateComponent(sbd)) {
                            candidates.add(sbd);
                        }
                    }
                }
                catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                        "Failed to read candidate component class: " + resource, ex);
                }
            }
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath 
                                               scanning", ex);
    }
    return candidates;
}
```

再看看两个`isCandidateComponent`方法的默认实现，一般来说我们可能需要重写这两个方法来改变默认的筛选条件。

```java
// 根据excludeFilters、excludeFilters初步筛选
// 一目了然，基本不用再需要解释
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
    for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, this.metadataReaderFactory)) {
            return false;
        }
    }
    for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, this.metadataReaderFactory)) {
            return isConditionMatch(metadataReader);
        }
    }
    return false;
}

// 根据类文件元数据筛选
// 只扫描顶层具体类或者虽然是抽象类但是存在@Lookup标记的方法
// 后面那个是用于方法注入，我从来没用过。
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
		AnnotationMetadata metadata = beanDefinition.getMetadata();
		return (metadata.isIndependent() && (metadata.isConcrete() ||
				(metadata.isAbstract() && metadata.hasAnnotatedMethods(
                    Lookup.class.getName()))));
}
```

**总结：此类在默认情况下会将指定包以及子包下中被`@Component`(以及衍生注解)标记的顶层类创建一个`BeanDefinition`。**

说到这插下Spring开启注解扫描的配置，有时我们可能只想在SpringMVC的配置文件中扫描@Controller标记的类，其它层扫描@Service、@Component、@Repository标记的类，就可以像下面这样分层配置。

springmvc.xml

```xml
<context:component-scan base-package="com.wangtao.controller" 
                        use-default-filters="false">
    <context:include-filter 
        type="annotation" 
        expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

applicationContext.xml

```xml
<context:component-scan base-package="com.wangtao.service,com.wangtao.dao">
    <context:exclude-filter 
        type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

也就是说SpringMVC中禁掉默认的`includeFilters`，添加了一个使用`@Controller`标记的条件，而applicationContext.xml使用默认的`includeFilters`，但是排除对`@Controller`标记的类。

### ClassPathBeanDefinitionScanner

此类继承自`ClassPathScanningCandidateComponentProvider`，除了拥有父类扫描包的功能外，还会对扫描后的`BeanDefinition`加工并注册到Spring容器中，所谓的加工就是指会设置一些类文件中用注解标记的一些属性值，如`@Lazy`、`@Scope`、`@Primary`等。

先看一些重要属性

```java
/** 用于注册bean **/
private final BeanDefinitionRegistry registry;

/** 
 * 此类存储了BeanDefinition一些默认属性值
 * lazyInit: false
 * autowireMode： AbstractBeanDefinition.AUTOWIRE_NO
 * initMethodName： null
 * destroyMethodName: null
**/
private BeanDefinitionDefaults beanDefinitionDefaults = new BeanDefinitionDefaults();

/** bean name 生成器 **/
private BeanNameGenerator beanNameGenerator = new AnnotationBeanNameGenerator();

/**
 * 默认值：true
 * 相当于开启<context:annotation-config>
 * 意味着我们可以使用@Autowired、@Resource、@PostConstruct、@PreDestroy注解
 * 会自动帮我们注册解析这几个注解的BeanPostProcessor
 */
private boolean includeAnnotationConfig = true;
```

构造方法

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, 
                                      boolean useDefaultFilters,
                                      Environment environment, 
                                      ResourceLoader resourceLoader) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    this.registry = registry;
    // 同ClassPathScanningCandidateComponentProvider
    if (useDefaultFilters) {
        registerDefaultFilters();
    }
    setEnvironment(environment);
    setResourceLoader(resourceLoader);
}
```

接下来看最重要的扫描方法

```java
/**
 * 返回扫描真正注册bean的数量
 */
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    doScan(basePackages);
	// 注册几个BeanPostProcessor用来解析@Autowired、@Reource等几个注解
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

/**
 * 将BeanDefinition集合返回，子类若有必要可以继续对BeanDefinition做修改
 */
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new 
        LinkedHashSet<BeanDefinitionHolder>();
    for (String basePackage : basePackages) {
        // 得到所有符合扫描条件的BeanDefinition集合，接下来会对这些BeanDefinition加工
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            // 主要包括bean的scope属性(默认单例)以及代理策略(不需要代理，JDK动态代理、CGLIB)
            // 来适配AOP
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver
                .resolveScopeMetadata(candidate);
            // 设置scope属性
            candidate.setScope(scopeMetadata.getScopeName());
            // 生成bean name
            String beanName = this.beanNameGenerator
                .generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                // 根据beanDefinitionDefaults设置一些默认值
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            // 如果是注解定义的Bean， findCandidateComponents默认实现返回的BeanDefinition
            // 是一个ScannedGenericBeanDefinition，其实现了AnnotatedBeanDefinition接口
            if (candidate instanceof AnnotatedBeanDefinition) {
                // 解析@Scope、@Primary、@Lazy等属性并设置到BeanDefinition中
                AnnotationConfigUtils.processCommonDefinitionAnnotations(
                    (AnnotatedBeanDefinition) candidate);
            }
            // 检查BeanDefinition
            // 主要检查这个容器中是否已经存在此BeanDefinition
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = 
                    new BeanDefinitionHolder(candidate, beanName);
                // 设置代理策略
                definitionHolder =AnnotationConfigUtils.applyScopedProxyMode(
                    scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                // 注册到Spring容器中
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```

因此如果我们需要自定义扫描实现bean的注册，基本上就是要继承`ClassPathBeanDefinitionScanner`并且重写`doScan`方法了。大致框架就是

```java
public class MyScanner extends ClassPathBeanDefinitionScanner {
    
    public MyScanner(BeanDefinitionRegistry registry) {
        super(registry)
    }
    
    @Overide
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        // 父类已经将这些bean注册了
        Set<BeanDefinitionHolder> holders = super.doScan(basePackages);
        for(BeanDefinitionHolder holder : holders) {
            // 在这里修改BeanDefinition引用的对象即可
        }
        return holders;
    }
}
```

### 例子

本小节将会实现一个将指定包下的带有@Mapper标记的接口生成代理注册到Spring容器中，命名将与MyBatis-Spring一致。

#### MapperFactoryBean

此类作用用来生成接口的代理实现

```java
public class MapperFactoryBean<T> implements FactoryBean<T>, InitializingBean {

    /**
     * 代理的接口
     */
    private Class<T> mapperInterface;

    /**
     * 代理对象
     */
    private T mapperObject;

    public MapperFactoryBean(Class<T> mapperInterface) {
        if(mapperInterface == null || !mapperInterface.isInterface()) {
            throw new IllegalArgumentException(mapperInterface + " 
                                               must be a interface.");
        }
        this.mapperInterface = mapperInterface;
    }

    @SuppressWarnings("unchecked")
    private T createProxy() {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(),
                                          new Class<?>[]{mapperInterface}, 
                                          new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) {
                System.out.println(method.getName() + " is called.");
                // 注意代理方法的返回值不能是基本类型, 否则返回null会发生空指针异常.
                return null;
            }                                     
        });
    }
    /**
     * 此方法会在Spring创建对象后, 并且调用了所有配置文件中配置属性的setter方法后执行
     * 在这里我们检查下参数以及创建接口代理对象
     */
    @Override
    public void afterPropertiesSet() {
        // 检查参数
        if(mapperInterface == null || !mapperInterface.isInterface()) {
            throw new IllegalArgumentException(mapperInterface + " 
                                               must be a interface.");
        }
        if(mapperObject == null) {
            mapperObject = createProxy();
        }
    }

    @Override
    public T getObject() {
        if(mapperObject == null) {
            mapperObject = createProxy();
        }
        return mapperObject;
    }

    @Override
    public Class<?> getObjectType() {
        return mapperInterface;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

#### @Mapper

```java
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Mapper {
    String value() default "";
}
```

#### MapperAnnotationBeanNameGenerator

此类用来生成bean的名字

```java
/**
 * bean name生成器
 * 继承自AnnotationBeanNameGenerator
 * AnnotationBeanNameGenerator会解析Componnet注解或者其衍生注解(@Service、@Repository等)的
 * value属性
 * 获取名字, 如果没有显示指定value属性, 那么会生成一个默认名字, 即类名首字母小写后的名字
 * 现在想要实现解析指定注解的value属性而不是Component注解
 * 如果value属性没有指定, 那么采用默认策略
 */
public class MapperAnnotationBeanNameGenerator extends AnnotationBeanNameGenerator {

    /**
     * 以这个注解指定的名字来生成bean name
     */
    private Class<? extends Annotation> annotation;

    public MapperAnnotationBeanNameGenerator(Class<? extends Annotation> annotation) {
        this.annotation = annotation;
    }

    /*
     * 所有注解信息都是通过.class文件解析出来的, 基于ASM实现, 也意味着这个类还被有被加载
     */
    @Override
    public String generateBeanName(BeanDefinition definition, 
                                   BeanDefinitionRegistry registry) {
        return super.generateBeanName(definition, registry);
    }

    /**
     * 重写此方法, 使得适配@Mapper注解指定的bean name.
     * 决定generateBeanName方法是否解析此annotationType代表的注解
     * 举例:
     * 若 annotationType = org.springframework.stereotype.Service(@Service)
     * 那么 metaAnnotationTypes = [@Target, @Retention, @Documented, @Component]
     * 即标记annotationType这个注解的注解集合
     * 那么 attributes = {value: ''}
     * @param annotationType 想要解析的注解
     * @param metaAnnotationTypes 注解annotationType的注解集合
     * @param attributes annotationType注解的属性集合
     */
    @Override
    protected boolean isStereotypeWithNameValue(String annotationType, 
                                                Set<String> metaAnnotationTypes, 
                                                Map<String, Object> attributes) {
        boolean isStereotype = (Objects.equals(annotationType, annotation.getName()))
                || (metaAnnotationTypes != null 
                    && metaAnnotationTypes.contains(annotation.getName()));
        return isStereotype && attributes.containsKey("value");
    }
}
```

#### MapperScannerConfigurer

提供可配置的扫描参数

```java
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean {

    private static final Logger LOG = LoggerFactory.getLogger(
        MapperScannerConfigurer.class);

    private String basePackage;

    /**
     * 默认为Mapper注解
     */
    private Class<? extends Annotation> markedAnnotation = Mapper.class;

    /**
     * bean name生成器
     */
    private BeanNameGenerator beanNameGenerator;

    public void setBasePackage(String basePackage) {
        this.basePackage = basePackage;
    }

    public void setMarkedAnnotation(Class<? extends Annotation> markedAnnotation) {
        this.markedAnnotation = markedAnnotation;
    }

    public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
        this.beanNameGenerator = beanNameGenerator;
    }

    @Override
    public void afterPropertiesSet() {
        if(basePackage == null) {
            throw new IllegalArgumentException("the property 'basePackage' 
                                               can not be null!");
        }
    }

    /**
     * 扫描指定包, 并且将满足扫描条件的的接口生成代理注册
     */
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 
                                               throws BeansException {
        MapperScanner mapperScanner = new MapperScanner(registry);
        // 添加扫描条件, 默认只扫描@Mapper标记的类文件
        mapperScanner.addIncludeFilter(new AnnotationTypeFilter(markedAnnotation));
        if(beanNameGenerator == null) {
            beanNameGenerator = new MapperAnnotationBeanNameGenerator(markedAnnotation);
        }
        mapperScanner.setBeanNameGenerator(beanNameGenerator);
        // 开始扫描, 并注册
        int beanCount = mapperScanner.scan(StringUtils.tokenizeToStringArray(
            basePackage, ",");
        LOG.debug("The count of registered bean is " + beanCount);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
                                           throws BeansException {
        // just do nothing.
    }
}

```

#### MapperScanner

```java
public class MapperScanner extends ClassPathBeanDefinitionScanner {

    public MapperScanner(BeanDefinitionRegistry registry) {
        // 禁用掉默认的扫描选择条件
        // 默认只扫描被@Component标记的类,
        // 当然了衍生注解(@Controller、@Service、@Repository)也会被扫描
        super(registry, false);
    }

    /**
     * 默认情况下只有顶层具体类才会通过
     * 只返回是接口的beanDefinition
     * @param beanDefinition bean
     * @return true / false
     */
    @Override
    protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
        return beanDefinition.getMetadata().isInterface() 
            && beanDefinition.getMetadata().isIndependent();
    }

    /**
     * 注册扫描的类
     * @param basePackages 包数组, 形如{"com.wangtao"}
     * @return 返回实际注册的bean数量
     */
    @Override
    public int scan(String... basePackages) {
        return super.scan(basePackages);
    }

    @Override
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        // 父类方法已经向Spring容器中注册过这些BeanDefinition了, 只需修改此引用
        // 不需要再注册
        Set<BeanDefinitionHolder> holders = super.doScan(basePackages);
        for (BeanDefinitionHolder holder : holders) {
            convertToMapperFactoryBean(holder.getBeanDefinition());
        }
        return holders;
    }

    /**
     * 修改原有Mapper接口的beanDefinition, 将其转化为MapperFactoryBean的beanDefinition
     * @param beanDefinition Mapper接口的beanDefinition
     */
    private void convertToMapperFactoryBean(BeanDefinition beanDefinition) {
        // 强转
        GenericBeanDefinition mapperFactoryDefinition = 
            (GenericBeanDefinition) beanDefinition;
        // 只能拿到接口名, 不能拿到Class对象, 因为此时还没有被类加载器加载
        String mapperInterfaceName = beanDefinition.getBeanClassName();
        mapperFactoryDefinition.setBeanClass(MapperFactoryBean.class);

        // 使用构造函数注入
        // 这里给的只是接口的完全限定名, 而不是Class对象, 因为Spring初始化的时候
        // 会自动将字符串转化成对应的类型, 而对应这里将会使用的是ClassEditor转化功能.
        // 之所以不用Class, 是因为对应Class文件此时还没有被类加载器加载
        mapperFactoryDefinition.getConstructorArgumentValues()
            .addIndexedArgumentValue(0, mapperInterfaceName);
    }
}
```

#### 测试

applicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="mapperScannerConfigurer"      class="com.wangtao.spring.scan.MapperScannerConfigurer">
    <property name="basePackage" value="com.wangtao.spring.scan.mapper" />
  </bean>
</beans>
```

com.wangtao.spring.scan.mapper包下的mapper接口

```java
/** 顶层具体类 **/
public class AddressMapper {
}

/** 顶层具体类，使用@Mapper标记 **/
@Mapper
public class StudentMapper {
}

/** 接口，将会被扫描，bean name = userMapper **/
@Mapper
public interface UserMapper {
    void insert();
}

/** 接口，将会被扫描，bean name = orderMapperProxy **/
@Mapper("orderMapperProxy")
public interface OrderMapper {
    void update();
}
```

```java
public class ScannerTest {
    @Test
    public void scan() {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(
            "classpath:spring-scan.xml");
        String[] beanNames = applicationContext.getBeanDefinitionNames();
        UserMapper userMapper = applicationContext
            .getBean("userMapper", UserMapper.class);
        Assert.assertNotNull(userMapper);
        userMapper.insert();
        OrderMapper orderMapper = applicationContext
            .getBean("orderMapperProxy", OrderMapper.class);
        Assert.assertNotNull(orderMapper);
        orderMapper.update();
    }
}
```

### 总结

文本主要介绍了Spring中两个非常重要的有关包扫描的类，最后给出一个可运行的例子来讲解如何使用这两个类来实现自定义的扫描器。从这个例子其实已经道出了MyBatis-Spring扫描Mapper接口的工作原理。