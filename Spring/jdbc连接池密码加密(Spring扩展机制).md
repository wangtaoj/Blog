### 前言

如果想要在application.yaml文件中配置的密码是一个密文，并且数据库连接池在初始化时可以正常的拿到连接，那么我们便要在连接池初始化前将密文变成明文。下面将使用Spring提供的几个扩展机制来实现这件事

### 方案1: BeanFactoryPostProcessor

`BeanFactoryPostProcessor`可以让我们拿到`DataSource`的`BeanDefinition`，这样我们便可以修改属性了。

```java
@Component
public class DataSourceBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        String[] beanNames = beanFactory.getBeanNamesForType(DataSource.class);
        for (String beanName : beanNames) {
            BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
            if (beanDefinition.getPropertyValues().contains("password")) {
                // 获取密文
                Object encryptPassword = beanDefinition.getPropertyValues().get("password");
                if (encryptPassword != null) {
                    // 解密获取明文(这里方便起见使用的Base64编码)
                    String realPassword = Base64.decodeStr(encryptPassword.toString());
                    // 覆盖属性
                    beanDefinition.getPropertyValues().add("password", realPassword);
                }
            }
        }
    }
}
```

**该代码只适用于以前古老的使用xml配置DataSource的场景，即下面配置**

```xml
<bean id="dataSource" class="org.apache.tomcat.jdbc.pool.DataSource">
    <!-- 数据库驱动 -->
    <property name="driverClassName" value="${db.driverClass}"/>
    <!-- 数据库地址 -->
    <property name="url" value="${db.url}"/>
    <!-- 数据库用户名 -->
    <property name="username" value="${db.username}"/>
    <!-- 数据库密码 -->
    <property name="password" value="xxxxx"/>
  </bean>
```

**因为这么配置下面这段代码才能拿到密文**

```java
// 获取密文
Object encryptPassword = beanDefinition.getPropertyValues().get("password");
```

而目前一般都使用Spring Boot，自动装配使用的Java配置，是拿不到值的。因此稍微改写从`Environment`实例中去拿密文。

```java
@Component
public class DataSourceBeanFactoryPostProcessor implements BeanFactoryPostProcessor, EnvironmentAware {

    private Environment environment;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        String[] beanNames = beanFactory.getBeanNamesForType(DataSource.class);
        for (String beanName : beanNames) {
            BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
            String encryptPassword = environment.getProperty("spring.datasource.password");
            if (encryptPassword != null) {
                // 解密获取明文(这里方便起见使用的Base64编码)
                String realPassword = Base64.decodeStr(encryptPassword);
                // 添加属性(即会调用set方法赋值)
                beanDefinition.getPropertyValues().add("password", realPassword);
            }
        }
    }

    @Override
    public void setEnvironment(@NonNull Environment environment) {
        this.environment = environment;
    }
}
```

**使用该种方式有一个问题，如果Bean有使用`@ConfigurationProperties`来装配属性，那么将会被再次覆盖，因为`Environment`中的属性并没有被修改。而`@ConfigurationProperties`绑定属性是在bean通过set方法赋值完了之后再装配的，绑定时机是`BeanPostProcessor.postProcessBeforeInitialization`**。

幸好连接池虽然使用了`@ConfigurationProperties`来装配其他属性，但是不包括密码，所以不会被覆盖。

```java
static class Tomcat {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.tomcat")
    org.apache.tomcat.jdbc.pool.DataSource dataSource(DataSourceProperties properties) {
        org.apache.tomcat.jdbc.pool.DataSource dataSource = createDataSource(properties,
                org.apache.tomcat.jdbc.pool.DataSource.class);
        DatabaseDriver databaseDriver = DatabaseDriver.fromJdbcUrl(properties.determineUrl());
        String validationQuery = databaseDriver.getValidationQuery();
        if (validationQuery != null) {
            dataSource.setTestOnBorrow(true);
            dataSource.setValidationQuery(validationQuery);
        }
        return dataSource;
    }

}
```

可以看到装配的属性前缀是`spring.datasource.tomcat`，而我们配置属性时是这样。

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1/test
    username: root
    password: MTIzNDU2
    tomcat:
      initial-size: 5
      max-active: 100
      min-idle: 5
```

所以环境中并没有`spring.datasource.tomcat.password`属性，因此就不会被覆盖了。

### 方案二: 直接修改Environment

可参考[Spring Boot环境扩展机制](https://www.cnblogs.com/wt20/p/17476002.html)

Spring Boot在启动时提供了扩展机制，用于定制`Environment`，并且执行时机是在上下文创建之前，因此该方案基本能应对所有情况。

```java
public class DataSourceEnvPostProcessor implements EnvironmentPostProcessor, Ordered {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        String encryptPassword = environment.getProperty("spring.datasource.password");
        // 解密获取明文(这里方便起见使用的Base64编码)
        String realPassword = Base64.decodeStr(encryptPassword);
        Map<String, Object> map = new HashMap<>();
        map.put("spring.datasource.password", realPassword);
        PropertySource<?> dbPasswordPropertySource = new MapPropertySource("dbPassword", map);
        // 优先级最高, 起到属性覆盖作用
        environment.getPropertySources().addFirst(dbPasswordPropertySource);
    }

    /**
     * 最低优先级执行, 低于别的EnvironmentPostProcessor实例执行, 以便可以拿到spring.datasource.password属性
     * 加载application.yml文件的EnvironmentPostProcessor是ConfigFileApplicationListener
     */
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}

```

MEAT-INF/spring.factories
```properties
org.springframework.boot.env.EnvironmentPostProcessor=\  
com.wangtao.springboottest.env.DataSourceEnvPostProcessor
```
指定自定义的`EnvironmentPostProcessor`，使得Spring Boot去加载它。

**该方案注意点就是自定义的`EnvironmentPostProcessor`执行顺序要低于加载`application.yml`文件的`EnvironmentPostProcessor`，以便可以拿到`application.yml`文件配置的属性，然后将自己定义的`PropertySource`放到列表最前面，起到属性覆盖作用。**

### 方案三: ApplicationContextInitializer扩展

`ApplicationContextInitializer`执行时机是`ApplicationContext`创建后执行(此时refresh方法还未调用)，此时`Environment`实例已经全部初始化完了(所有的`EnvironmentPostProcessor`全部执行完毕)。

**本来方案二已经很完美了，但是微服务项目一般使用配置中心，如果把密码放到配置中心中，方案二将失效，因为配置中心的加载时机是在`ApplicationContextInitializer`执行的，晚于`EnvironmentPostProcessor`，因此我们将拿不到`spring.datasource.password`的属性值，导致方案二失效。**

于是使用`ApplicationContextInitializer`方案便出来了。

```java
public class DataSourceApplicationContextInitializer implements Ordered,
        ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(@NonNull ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        String encryptPassword = environment.getProperty("spring.datasource.password");
        // 解密获取明文(这里方便起见使用的Base64编码)
        String realPassword = Base64.decodeStr(encryptPassword);
        Map<String, Object> map = new HashMap<>();
        map.put("spring.datasource.password", realPassword);
        PropertySource<?> dbPasswordPropertySource = new MapPropertySource("dbPassword", map);
        // 优先级最高, 起到属性覆盖作用
        environment.getPropertySources().addFirst(dbPasswordPropertySource);
    }

    /**
     * 最低优先级, 低于加载配置中心的ApplicationContextInitializer即可
     * PropertySourceBootstrapConfiguration
     */
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}

```

MEAT-INF/spring.factories
```properties
org.springframework.context.ApplicationContextInitializer=\  
com.wangtao.springboottest.config.DataSourceApplicationContextInitializer
```
指定自定义的`ApplicationContextInitializer`，使得Spring Boot去加载它。

该方案本质上与`EnvironmentPostProcessor`没有什么区别，只是执行时机稍微晚了一点点。