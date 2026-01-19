### Spring官方文档参考

https://docs.spring.io/spring-boot/reference/features/logging.html

### Logback变量默认值

如果变量未定义，在`<pattern>`标签中使用时情况说明

${}取值，从`<property>`标签定义的变量获取 -> jvm系统参数->环境变量 (优先级从左往右)

%X{}取值，从MDC中获取变量值

```xml
<!-- 日志中会输出APP_NAME_IS_UNDEFINED -->
<pattern>${APP_NAME}</pattern>

<!-- 日志中会输出空字符串 -->
<pattern>%X{APP_NAME}</pattern>
```

默认值写法

使用:-语法，注意需要一个横杠(-)

```xml
<!-- 这样子就会输出unknow了 -->
<pattern>${APP_NAME:-unknow}</pattern>
<!-- 注意这种写法也不会输出空字符串, 而是输出APP_NAME_IS_UNDEFINED -->
<pattern>${APP_NAME:-}</pattern>
<!-- 这样子就会输出unknow了 -->
<pattern>%X{APP_NAME:-unknow}</pattern>
```

### 日志实现切换

SpringBoot默认的日志实现为logback，如果想要切换成log4j2，只需排除logback依赖，添加log4j2的即可

```xml
<!-- 添加log4j2依赖 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>

<!-- spring-boot-starter是基础依赖，自带了logback日志依赖, 排除即可 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-logging</artifactId>
    </exclusion>
  </exclusions>
</dependency>

```

### 默认的日志配置

在没有指定配置文件时，Spring Boot打印日志行为可参考spring-boot.jar包中base.xml。

路径为org.springframework.boot.logging.logback包下。

log4j2同理。

### 日志配置文件

以logback为例

类路径下存在

* logback-spring.xml
* logback.xml

同时存在时Spring Boot优先使用logback-spring.xml。

那么logback-spring.xml与logback.xml有什么区别呢？

logback.xml是logback日志框架默认的配置文件，由logback自己加载，Spring Boot不会参与解析和增强它，而logback-spring.xml由Spring Boot来加载，会被Spring Boot增强，并且解析时机在Spring的环境初始化之后。因此在logback-spring.xml中可以读取Spring环境中的变量。

```xml
<configuration>
  <!-- 从Spring环境中读取配置 -->
  <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="myapp"/>

  <!-- 根据spring激活的profile来生效配置 -->
  <springProfile name="dev">
    <logger name="com.waston.app" level="debug">
  </springProfile>

  <springProfile name="dev">
    <Logger name="com.waston.app" level="info">
  </springProfile>
</configuration>
```

注: SpringBoot 2.x版本扩展语法对log4j2不支持，需要引入log4j2扩展语法的依赖才行。Spring Boot 3.x版本官方支持，无需引入额外依赖。

```xml
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-spring-boot</artifactId>
  <scope>runtime</scope>
</dependency>
```

```xml
<Properties>
  <!-- 从Spring环境中读取配置 -->
  <Property name="applicationName">${spring:spring.application.name:-default}</Property>
</Properties>

<SpringProfile name="dev">
  <Logger name="com.waston.app" level="debug">
</SpringProfile>
```

### JVM系统属性

Spring Boot会把环境中一些日志属性添加到JVM系统属性中，具体可参考`LoggingSystemProperties`

| `logging.file.name` | `LOG_FILE`          |
| ------------------- | ------------------- |
| `logging.file.path` | `LOG_PATH`          |
| 无                  | PID(从操作系统获取) |

注：如果环境中的值存在${}占位符，也会一并替换掉的。

