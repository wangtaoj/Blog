### 开启测试

#### junit5

Spring Boot 2.7.x版本默认使用junit5

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

测试类

```java
@SpringBootTest
public class ApplicationTest {

    @Test
    public void contextLoad() {
        
    }
}
```

只要增加`@SpringBootTest`即可。

#### junit4

需要单独增加junit4的执行引擎

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>org.junit.vintage</groupId>
  <artifactId>junit-vintage-engine</artifactId>
  <scope>test</scope>
  <exclusions>
    <exclusion>
      <groupId>org.hamcrest</groupId>
      <artifactId>hamcrest-core</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTest {

    @Test
    public void contextLoad() {
        
    }
}
```

需要增加`@SpringBootTest`以及`@RunWith(SpringRunner.class)`

这样子既可以执行junit5也可以执行junit4

### 主配置类的寻找机制

首先主配置类是可以有多个的，默认情况下就是被`@SpringBootApplication`注解的类。

* (1)默认规则：**从测试类所在包，一层层往上找**，寻找带 `@SpringBootApplication` / `@SpringBootConfiguration` 的类，作为启动配置类。
* (2)也可以手动指定：`@SpringBootTest(classes = xxx.class)`
* (3)测试类中静态内部类加上`@Configuration`，也会被当做主配置类，可以和(2)同时生效。
* (4)测试类中静态内部内加上@TestConfiguration提供额外的配置，可以和(1)、(2)、(3)同时生效。

**注意: 默认的规则(1)与(2)和(3)是互质的，如果有了(2)或者(3)，则不会再加载默认规则了**

### @SpringBootApplication中的包扫描注意点

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    
}
```

可以看到`@SpringBootApplication` 里面的 `@ComponentScan` 默认自带两个排除过滤器

1. 排除自动配置类，防止提前加载出问题。两种情况: 

   被`@AutoConfiguration`注解的类，无需在spring.factories文件指定

   被`@Configuration`注解的类，需在spring.factories文件指定

2. 排除 `@TestComponent`、`@TestConfiguration`

3. 排除带测试注解（`@ExtendWith`、`@Test`）的类 → **也就是测试类本身不会被 Spring 扫描注册成 Bean**

4. 测试类里面的**静态内部类，就算写了 @Component，也不会被扫描**

可翻阅`TestTypeExcludeFilter`以及`AutoConfigurationExcludeFilter`源码看细节。

### 关于事务

普通业务代码没有这个效果，**只有单元测试才有**

- 如果 `@Transactional` 写在**测试类 / 测试方法上**：测试跑完之后无论是否发生异常，**事务都会回滚**，想要正常执行完事务是提交，加 `@Commit`
- 如果 `@Transactional` 加在 Service 等业务类上： 和平时一样，**正常提交，不会自动回滚**