### 前言

Spring Boot 2.4.0对于环境属性加载进行了重写，废弃了`ConfigFileApplicationListener`类，而使用了新的`ConfigDataEnvironmentPostProcessor`来加载属性配置。而且还引入了`spring.config.import`属性用于导入外部配置，因此Spring Cloud也默认不再创建bootstrap上下文，配置中心可以使用`srping.config.import`机制来加载属性。

### 使用

#### 老的方式

如果仍要用bootstrap.yml方式来使用配置中心，需要引入以下依赖

```xml
<!-- 开启bootstrap上下文 -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

BootstrapApplicationListener.java

```java
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    ConfigurableEnvironment environment = event.getEnvironment();
    // 开关没有开启并且也没有使用老的方式(ConfigFileApplicationListener)来加载配置
    if (!bootstrapEnabled(environment) && !useLegacyProcessing(environment)) {
        return;
    }
    // 创建bootstrap上下文
}

public static boolean bootstrapEnabled(Environment environment) {
		return environment.getProperty(BOOTSTRAP_ENABLED_PROPERTY, Boolean.class, false) || MARKER_CLASS_EXISTS;
}
```

引入上面依赖实则就是导入一个标记类，这个标记类存在则满足条件，或者在JVM启动参数设置开关为true

`spring.cloud.bootstrap.enabled=true`，注意不能写到application.yml文件中，因为此时还没加载呀。

#### 新的方式

application.yml

```yaml
spring:
  application:
    name: nacos-config
  config:
    import: nacos:/nacos-config.yml
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yml
```

上述就是导入namespace=public，group=DEFAULT_GROUP，dataId=nacos-config.yml的配置。

导入url还可以携带参数，如nacos:/nacos-config.yml?group=xxx&refreshEnabled=false。

参数可以覆盖全局指定的值。

`spring.config.import`可以导入多个配置，如下所示

```yaml
spring:
  config:
    import:
     - nacos:/nacos-config.yml
     - nacos:/test
```

全局参数

* namespace: spring.cloud.nacos.config.namespace指定，默认为空，即public
* group: spring.cloud.nacos.config.group指定，默认DEFAULT_GROUP，可以被导入url中group参数覆盖
* fileExtension: spring.cloud.nacos.config.fileExtension，默认为properties，如果dataId携带后缀，则使用dataId中的后缀
* refresh: spring.cloud.nacos.config.refreshEnabled指定，默认为true，可以被导入url中refreshEnabled参数覆盖

注意的点

* 与老的方式比，默认不会自动去加载dataId为服务名称的配置，必须使用spring.config.import明确指定。
* nacos目前不会自动去拼接profile信息，即如果当前激活的profile是dev，不会去加载nacos-config-dev.yml。
* 顺序性，import进来的配置仅仅位于application.yml之前，也就是优先级高于application.yml，而老的方式会把配置挪到最前面。

### 简要原理

引入以下几个组件

* ConfigDataLocation，代表配置的位置，一个字符串表示
* ConfigDataResource，代表具体配置
* ConfigDataLocationResolver，用于把ConfigDataLocation指定的配置解析成对应的ConfigDataResource
* ConfigDataLoader，用于把ConfigDataResource加载成具体的PropertySource

因此路径写法可以参考ConfigDataLocationResolver类中的解析。