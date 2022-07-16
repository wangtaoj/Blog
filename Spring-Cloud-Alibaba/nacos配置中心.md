### 文档

* [官方文档](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-config)
* 本节代码见nacos-config工程

### 配置中心基本概念

nacos配置中心使用namespace，group，dataId来唯一定位一个配置，默认的namespace为public，group为 DEFAULT_GROUP。dataId为配置的名字，包含后缀。nacos支持text、json、yaml、properties等文件格式作为配置文件。

### Java项目基础使用

#### 基本使用

引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

在 **bootstrap.yml**文件中配置nacos server的地址

```yaml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
```

不像上一节服务发现的例子，配置中心相关配置必须写在**bootstrap.yml**，而不能写在**application.yml**中。对于此配置，项目中将读取namespace=public、group=DEFAULT_GROUP、dataId=nacos-config.properties的配置文件，dataId默认使用服务名，格式默认为properties。可以使用`spring.cloud.nacos.config.file-extension=yml`来指定格式。

```yml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yml
```

#### 动态刷新

默认情况下，即使用服务名作为dataId的配置文件时，是支持动态刷新的，当修改了配置的值，Java项目可以感知到。可以由` spring.cloud.nacos.config.refresh.enabled=false `来关闭

测试例子

```java
@RefreshScope
@RequestMapping("/api/config")
@RestController
public class ConfigController {

    @Value("${user.id}")
    private Integer id;

    @Value("${user.name}")
    private String name;

    @Value("${user.age}")
    private Integer age;

    @Autowired
    private Environment environment;

    @Autowired
    private UserProperties userProperties;

    /**
     * 动态刷新时对应的值不会更新
     * 需要借助@RefreshScope注解, 再次访问这个bean时会更新
     */
    @GetMapping("/byValue")
    public UserVO byValue() {
        return new UserVO(id, name, age);
    }

    /**
     * 动态刷新时对应的值也会更新
     */
    @GetMapping("/byConfigurationProperties")
    public void byConfigurationProperties() {
        Instant start = Instant.now();
        while (true) {
            Duration duration = Duration.between(start, Instant.now());
            if (duration.getSeconds() > 60) {
                break;
            }
            System.out.println(userProperties);
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    @GetMapping("/dynamicUpdate")
    public void dynamicUpdate() {
        Instant start = Instant.now();
        while (true) {
            Duration duration = Duration.between(start, Instant.now());
            if (duration.getSeconds() > 60) {
                break;
            }
            String name = environment.getProperty("user.name");
            String age = environment.getProperty("user.age");
            System.out.println("name: " + name + ", age: " + age);
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

在Spring Boot中使用配置，一般是这3种方式

* `environment.getProperty("key")` API直接获取，此种方式会拿到修改后的值
* 使用`@ConfigurationProperties(prefix = "user")`，此种方式也能拿到修改后的值
* 使用@Value注解，不能拿到修改后的值，需要搭配`@RefreshScope`使用，**重新访问bean时**会拿到修改后的值，如果for 循环去获取age属性，没有重新访问bean，不会生效。

#### 借助profile实现环境隔离

加载配置时，不仅仅加载了以 dataid 为 `${spring.application.name}.${file-extension:properties}` 为前缀的基础配置，还加载了dataid为 `${spring.application.name}-${profile}.${file-extension:properties}` 的基础配置 。

使用`-D spring.profiles.active=prd`来指定profile

#### 支持自定义namespace、group

```yml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: xxx
        group: xxx
```

#### 支持自定义dataId

```yml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        extension-configs:
          - data-id: custom1.yml
            # 默认DEFAULT_GROUP
            group: xxxx1
            # 默认值为false
            refresh: true
          - data-id: custom2.yml
            group: xxxx2
            refresh: true
```

- 通过 `spring.cloud.nacos.config.extension-configs[n].data-id` 的配置方式来支持多个 Data Id 的配置。
- 通过 `spring.cloud.nacos.config.extension-configs[n].group` 的配置方式自定义 Data Id 所在的组，不明确配置的话，默认是 DEFAULT_GROUP。
- 通过 `spring.cloud.nacos.config.extension-configs[n].refresh` 的配置方式来控制该 Data Id 在配置变更时，是否支持应用中可动态刷新， 感知到最新的配置值。默认是不支持的。
-  多个 Data Id 同时配置时，他的优先级关系是 `spring.cloud.nacos.config.extension-configs[n].data-id` 其中 n 的值越大，优先级越高。 
-  `spring.cloud.nacos.config.extension-configs[n].data-id` 的值必须带文件扩展名，文件扩展名既可支持 properties，又可以支持 yaml/yml。 此时 `spring.cloud.nacos.config.file-extension` 的配置对自定义扩展配置的 Data Id 文件扩展名没有影响。 

### 配置的优先级

Spring Cloud Alibaba Nacos Config 目前提供了三种配置能力从 Nacos 拉取相关的配置。

- A: 通过 `spring.cloud.nacos.config.shared-configs[n].data-id` 支持多个共享 Data Id 的配置
- B: 通过 `spring.cloud.nacos.config.extension-configs[n].data-id` 的方式支持多个扩展 Data Id 的配置
- C: 通过内部相关规则(应用名、应用名+ Profile )自动生成相关的 Data Id 配置

当三种方式共同使用时，他们的一个优先级关系是:A < B < C