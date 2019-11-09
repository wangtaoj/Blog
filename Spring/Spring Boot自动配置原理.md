### 声明

> 源码基于Spring Boot 2.0.4

### 前文

这两篇文章对理解这篇文章的知识会很有帮助。

* [Spring 注解配置原理](https://www.cnblogs.com/wt20/p/11823783.html)

* [神奇的条件注解-Spring Boot自动配置的基石](https://www.cnblogs.com/wt20/p/11825423.html)

### 自动配置介绍

在Spring Boot中开启自动配置只需要在配置类上加上`@EnableAutoConfiguration`注解即可。Spring Boot程序都会在启动类添加`@SpringBootApplication`注解，`@SpringBootApplication`注解其实是是一个组合注解，相当于`@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan`这几个注解一起使用。因此Spring Boot程序默认开启自动配置。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    /**
     * 自动配置的开关，当我们程序使用了@EnableAutoConfiguration
     * 如果想要关掉自动配置，只需在application.properties文件加上
     * spring.boot.enableautoconfiguration = false 或者
     * spring.boot.enable-auto-configuration = false
     */
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    /**
     * 想要排除哪些自动配置类
     */
	Class<?>[] exclude() default {};

    /**
     * 同上，只不过使用类的完全限定名
     */ 
	String[] excludeName() default {};
}
```

默认情况下，Spring会去寻找类路径下META-INF/spring.factories文件，然后加载这个文件指定的自动配置类。具体自动配置行为全都是依赖这些自动配置类完成的。

```properties
# spring.factories 部分内容
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
等
```

### 源码分析

所有的`@EnableXXX`注解行为都是由`@Import`注解来完成的，`@EnableAutoConfiguration`注解同样也被`Import`注解注解了。重点关注`AutoConfigurationImportSelector`即可。这个类实现了`DeferredImportSelector`接口。也就是在主配置类全部解析完成后再执行`selectImports`方法选择哪些类继续进行解析。也就是说主配置类定义的bean优先注册，然后再注册`selectImports`选择的类，保证了用户的配置优先。

注: 主配置定义的bean包括主配置本身以及通过`@import`注解引入的bean以及注解扫描注册的bean。

```java
public class AutoConfigurationImportSelector
    implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
    BeanFactoryAware, EnvironmentAware, Ordered {
	
    /**
     * 将选择的类继续交给Spring解析
     * 解析类为ConfigurationClassPostProcessor
     * 感兴趣可参考这篇文章<<Spring 注解配置原理>>
     * 链接: https://www.cnblogs.com/wt20/p/11823783.html
     */
    @Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		// 判断spring.boot.enable-auto-configuration属性值
        if (!isEnabled(annotationMetadata)) {
            // 返回一个空数组，不选择任何一个自动配置类，即关闭自动配置
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata =     		   	                           AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		// 获取META-INF/spring.factories文件中
        // org.springframework.boot.autoconfigure.EnableAutoConfiguration属性值
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
        // 移除重复自动配置类
		configurations = removeDuplicates(configurations);
        // 获取exclude属性以及excludeName属性指定的类，将这些自动配置类跳过
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        // 校验这些排除类
		checkExcludedClasses(configurations, exclusions);
        // 从自动配置类集合中移除排除的类
		configurations.removeAll(exclusions);
        // 对剩下的自动配置类做一个过滤，具体不展开了
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return StringUtils.toStringArray(configurations);
	}
}

protected boolean isEnabled(AnnotationMetadata metadata) {
    if (getClass() == AutoConfigurationImportSelector.class) {
        // 判断spring.boot.enable-auto-configuration属性值, 默认为true
        return getEnvironment().getProperty(
            EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
            true);
    }
    return true;
}

// 检查有没有无效的排除类
private void checkExcludedClasses(List<String> configurations,
                                  Set<String> exclusions) {
    List<String> invalidExcludes = new ArrayList<>(exclusions.size());
    /*
     * 如果排除类在类路径存在并且不在自动配置类集合中就是一个无效的排除类
     */
    for (String exclusion : exclusions) {
        if (ClassUtils.isPresent(exclusion, getClass().getClassLoader())
            && !configurations.contains(exclusion)) {
            invalidExcludes.add(exclusion);
        }
    }
    // 检测到指定的排除类包含无效的类，抛出异常
    if (!invalidExcludes.isEmpty()) {
        handleInvalidExcludes(invalidExcludes);
    }
}
```

### 案列分析-数据源的自动配置

数据源的自动配置类为`DataSourceAutoConfiguration`，我们可以在spring-boot-autoconfigure.jar中的META-INF/spring.factories文件中找到这个类的声明。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
略,\
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
略,\
```

从类路径中寻找具体数据源实现的顺序为 HikariCP  ->  Tomcat -JDBC -> Commons-DBCP2 -> 其它数据源

可以参考官方文档[Working with SQL Databases - 29.1.2]( https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/html/boot-features-sql.html )。

也可以在applicaiton.properties文件中直接指定 `spring.datasource.type` 属性来指定使用哪个数据源，从而绕过这个寻找机制。

```java
/*
 * 只截取关键代码
 */
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
		DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {
    
    /**
     * 从这里就可以看出寻找顺序了
     * 当容器中没有DataSource 或者 XADataSource这两个类的bean才去创建
     */
    @Configuration
	@Conditional(PooledDataSourceCondition.class)
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
			DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
			DataSourceJmxConfiguration.class })
	protected static class PooledDataSourceConfiguration {

	}
}
```

```java
/**
 * 这里只挑Tomcat-JDBC做下说明
 * HikariCP、Commons-DBCP2和Tomcat-JDBC是一样的
 */
abstract class DataSourceConfiguration {

	@SuppressWarnings("unchecked")
	protected static <T> T createDataSource(DataSourceProperties properties,
			Class<? extends DataSource> type) {
		return (T) properties.initializeDataSourceBuilder().type(type).build();
	}

	/**
	 * 需要满足下面三个条件
	 * 1. tomcat-jdbc数据源在类路径上
	 * 2. 容器中不存在DataSource的bean
	 * 3. 如果没有指定spring.datasource.type属性，默认通过(matchIfMissing = true)
	 *    明确指定时spring.datasource.type值必须为org.apache.tomcat.jdbc.pool.DataSource
	 * Tomcat Pool DataSource configuration.
	 */
	@ConditionalOnClass(org.apache.tomcat.jdbc.pool.DataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.tomcat.jdbc.pool.DataSource", matchIfMissing = true)
	static class Tomcat {
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.tomcat")
		public org.apache.tomcat.jdbc.pool.DataSource dataSource(
				DataSourceProperties properties) {
			org.apache.tomcat.jdbc.pool.DataSource dataSource = createDataSource(
					properties, org.apache.tomcat.jdbc.pool.DataSource.class);
			DatabaseDriver databaseDriver = DatabaseDriver
					.fromJdbcUrl(properties.determineUrl());
			String validationQuery = databaseDriver.getValidationQuery();
			if (validationQuery != null) {
				dataSource.setTestOnBorrow(true);
				dataSource.setValidationQuery(validationQuery);
			}
			return dataSource;
		}
	}

	/**
	 * 通用的数据源配置，如果使用的数据源实现不是以上三种的一个，将会通过这个类来处理
	 * 当然依然要满足条件才会配置
	 * 1. 容器中不存在dataSource bean
	 * 2. 指定了spring.datasource.type属性，只要该属性不为false就会通过
	 *    因为@ConditionalOnProperty注解的havingValue属性没有指定
	 * Generic DataSource configuration.
	 */
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type")
	static class Generic {
		@Bean
		public DataSource dataSource(DataSourceProperties properties) {
            // 初始化数据源
			return properties.initializeDataSourceBuilder().build();
		}
	}
}
```

```java
// DataSourceProperties类
public DataSourceBuilder<?> initializeDataSourceBuilder() {
    // 初始化属性
    // 通常而言驱动类都不用明确指定，Spring会跟据url做推断
    // url, username, passowrd如果没有指定走内嵌数据库
    return DataSourceBuilder.create(getClassLoader()).type(getType())
        .driverClassName(determineDriverClassName()).url(determineUrl())
        .username(determineUsername()).password(determinePassword());
}

// 推断driverClassName
public String determineDriverClassName() {
    // 明确指定, 检查下驱动是否可以被加载，可以直接返回
    if (StringUtils.hasText(this.driverClassName)) {
        Assert.state(driverClassIsLoadable(),
                     () -> "Cannot load driver class: " + this.driverClassName);
        return this.driverClassName;
    }
    // 从url中推断
    String driverClassName = null;
    if (StringUtils.hasText(this.url)) {
        driverClassName = DatabaseDriver.fromJdbcUrl(this.url).getDriverClassName();
    }
    // url也为空，走内嵌数据库，前提是类路径需要存在内嵌数据库依赖
    if (!StringUtils.hasText(driverClassName)) {
        driverClassName = this.embeddedDatabaseConnection.getDriverClassName();
    }
    if (!StringUtils.hasText(driverClassName)) {
        throw new DataSourceBeanCreationException(
            "Failed to determine a suitable driver class", this,
            this.embeddedDatabaseConnection);
    }
    return driverClassName;
}
```

