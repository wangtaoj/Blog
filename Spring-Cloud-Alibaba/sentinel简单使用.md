



### 代码示例

[项目地址](https://github.com/wangtaoj/SpringCloudAlibaba)，见sentinel-example

[sentinel官方文档](https://sentinelguard.io/zh-cn/docs/introduction.html)

### 控制台搭建

为了避免不必要的麻烦，直接下载对应sentinel版本的控制台，具体版本号可以去spring cloud alibaba的pom去找，本项目使用1.8.1版本。[下载地址](https://github.com/alibaba/Sentinel/releases/tag/1.8.1)

sentinel控制台是一个Spring Boot项目，通过java -jar的方式运行即可。新建一个start.bat，内容如下

```bash
java -Dserver.port=8888 -Dcsp.sentinel.dashboard.server=localhost:8888 -Dproject.name=sentinel-dashboard -jar ./sentinel-dashboard-1.8.1.jar
```

因为写的是相对路径，保证start.bat与sentinel-dashboard-1.8.1.jar在同一个目录下，双击start.bat即可运行控制台，端口为8888。浏览器输入`localhost:8888`即可访问控制台的登录页，用户名/密码默认为sentinel/sentinel。

### 客户端接入控制台

1. 客户端需要引入 Transport 模块来与 Sentinel 控制台进行通信 

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
</dependency>
```

因为`spring-cloud-starter-alibaba-sentinel`包含了Transport模块，因此可以省略，使用sentinel时，基本都会先引入`spring-cloud-starter-alibaba-sentinel`。

2. applicatioin.yml文件配置控制台的基本信息

```yaml
spring:
  application:
    name: sentinel-example
  cloud:
    sentinel:
      transport:
        port: 8719
        # 控制台地址
        dashboard: 127.0.0.1:8888
```

 其中 `spring.cloud.sentinel.transport.port` 端口配置会在应用对应的机器上启动一个 Http Server，该 Server 会与 Sentinel 控制台做交互。比如 Sentinel 控制台添加了一个限流规则，会把规则数据 push 给这个 Http Server 接收，Http Server 再将规则注册到 Sentinel 中， 默认值为8719。

当**首次**访问sentinel客户端的的controller接口时，可以在 **簇点链路** 中看到以这个接口路径命名的资源，在这里可以通过控制台手动设置流控、降级等规则。**值得注意的是当sentinel客户端引用重新启动后，配置将失效，也就是没有持久化**。因为控制台把页面配置的的数据推送到了sentinel客户端，只是保存到内存中。

### 从nacos配置中心读取规则信息

上面说到控制台的手工配置的数据并没有持久化，应用启动后便失效了，我们可以借助nacos配置中心，将配置存储到nacos中，这样应用启动后便可以重新从nacos配置中心读取数据初始化规则了。值得一提的该做法还不是很完善，**如果在nacos配置中心修改了内容，应用是可以感知到并生效的，因为nacos会推送最新的配置内容过来。但是如果在控制台修改了配置规则，是不会同步修改到nacos配置中心里的，因此应用重新启动后，在控制台做的修改将会丢失**。如果需要做到控制台修改，同步修改nacos配置中心，需要修改控制台的源码，略微有点繁琐，便留给日后探索了。

1. 引入sentinel-datasource-nacos数据源

   ```xml
   <dependency>
       <groupId>com.alibaba.csp</groupId>
       <artifactId>sentinel-datasource-nacos</artifactId>
   </dependency>
   ```

2. 在applicatioin.yml指定配置位置信息

```yaml
spring:
  application:
    name: sentinel-example
    sentinel:
      # 饥饿加载
      eager: true
      transport:
        port: 8719
        dashboard: 127.0.0.1:8888
      datasource:
        # 自定义(datasource是一个Map, 因此key是自定义的)
        flow-config:
          nacos:
            server-addr: 127.0.0.1:8848
            username: nacos
            password: nacos
            group-id: SENTINEL_GROUP
            data-id: sentinel-flow-rules.json
            data-type: json
            # 表示该配置为流控规则(见枚举类RuleType)
            rule-type: FLOW
```

配置信息的key定义在`SentinelProperties`类中，不用去记的。

```java
@ConfigurationProperties(prefix = SentinelConstants.PROPERTY_PREFIX)
@Validated
public class SentinelProperties {

	private boolean eager = false;

	private boolean enabled = true;

	private String blockPage;

	/**
	 * 数据源配置
	 */
	private Map<String, DataSourcePropertiesConfiguration> datasource = new TreeMap<>(
			String.CASE_INSENSITIVE_ORDER);

	private Transport transport = new Transport();
}
```

```java
public class DataSourcePropertiesConfiguration {

	private FileDataSourceProperties file;

    /**
     * nacos配置
     */
	private NacosDataSourceProperties nacos;

	// ... 略
}
```

```java
public class AbstractDataSourceProperties {

	@NotEmpty
	private String dataType = "json";

	@NotNull
	private RuleType ruleType;
    
    // ... 略

}

public class NacosDataSourceProperties extends AbstractDataSourceProperties {

	private String serverAddr;

	private String username;

	private String password;

	@NotEmpty
	private String groupId = "DEFAULT_GROUP";

	@NotEmpty
	private String dataId;

	private String endpoint;

	private String namespace;

	private String accessKey;

	private String secretKey;
}
```

其中有一个地方需要注意下，就是`data-id`必须是全名，如果在nacos中心命名有带后缀的话，那么配置时也得加下，因为没有一个像配置中心那边一样有一个`file-extension`属性，我以为会拼上`data-type`的值，但是并没有。

3. 在nacos配置中心新建一个流控规则的配置文件

   新建一个dataId=sentinel-flow-rules.json，group=SENTINEL_GROUP，namespace为默认(public)，内容如下

   ```json
   [
       {
           "resource": "/hello",
           "limitApp": "default",
           "grade": 1,
           "count": 2
       }
   ]
   ```

   内容是一个JSON数组，因为可以配置多个流控规则，字段以及值参考`FlowRule.java`类，如果是将降级规则，则参考`DegradeRule.java`类。本例定义一个`/hello`资源，规则是`qps`，阈值是`2`，即如果每秒访问该资源超过2次便会触发流控。

4. 总结

   以流控规则为例，讲解了如何配置应用从nacos配置中心读取sentinel规则配置，其它的规则类似，比如再配置一个降级规则，只需要再追加一个配置即可

```yaml
spring:
  application:
    name: sentinel-example
    sentinel:
      # 饥饿加载
      eager: true
      transport:
        port: 8719
        dashboard: 127.0.0.1:8888
      datasource:
        # 自定义(datasource是一个Map, 因此key是自定义的)
        flow-config:
          nacos:
            server-addr: 127.0.0.1:8848
            username: nacos
            password: nacos
            group-id: SENTINEL_GROUP
            data-id: sentinel-flow-rules.json
            data-type: json
            # 表示该配置为流控规则(见枚举类RuleType)
            rule-type: FLOW
         degrade-config:
           nacos:
             server-addr: 127.0.0.1:8848
             username: nacos
             password: nacos
             group-id: SENTINEL_GROUP
             data-id: sentinel-degrade-rules.json
             data-type: json
            # 表示该配置为降级规则(见枚举类RuleType)
             rule-type: DEGRADE
```

### sentinel简单使用

```java
@RequestMapping("/sentinel")
@RestController
public class SentinelController {

    @SentinelResource(value = "/hello", blockHandler = "helloBlockHandler", fallback = "helloFallback")
    @GetMapping("/hello")
    public String hello(boolean throwException) {
        if (throwException) {
            throw new IllegalArgumentException();
        }
        return "Hello, Sentinel";
    }

    public String helloBlockHandler(boolean throwException, BlockException e) {
        System.out.println(throwException);
        System.out.println(e.getMessage());
        return "Hello, Sentinel Block.";
    }

    public String helloFallback(boolean throwException) {
        System.out.println(throwException);
        return "Hello Exception";
    }
}
```

最重要的就是`@SentinelResource`注解了，用来定义一个资源。其中`value`表示资源名称，`blockHandler`表示发生了流控、降级等规则时执行的方法，`fallback`表示发生了异常后要执行的方法。它们的方法签名要和源方法一样，返回值也一致。其中`blockHandler`方法需要在参数后面加一个`BlockException`类型的参数，从该异常可以得知时触发了流控规则还是降级规则，或者其它规则。当然了，可以将这些方法写到另外一个类中，但是这些方法此时必须是**静态方法**了，使用`blockHandlerClass`、`fallbackClass`来指定是哪个类。

### 与open feign结合

`@FeignClient`注解中有一个`fallback`属性，当访问远程接口出现错误时，便会降级执行`fallback`属性指定类的中方法，默认情况下，没有服务降级组件(如sentinel、 Hystrix)时，即便指定也是没有效果的。本节便借助sentinel来达到这个目的。

加入下面这行配置即可启用fegin调用服务时发生降级处理

```yacas
feign:
  sentinel:
    enabled: true
```

模拟例子，代码如下

Controller接口

```java
@RequestMapping("/sentinel")
@RestController
public class SentinelController {

    @Autowired
    private UserService userService;

    @GetMapping("/detailUser/{userId}")
    public UserVO detailUser(@PathVariable Integer userId) {
        return userService.detailUser(userId);
    }
}

```
feign接口
```java
@FeignClient(value = "nacos-producer", path = "/api/users", 
             configuration = NacosProducerFeignConfig.class,
             fallback = UserServiceFallback.class)
public interface UserService {

    @GetMapping("/{userId}")
    UserVO detailUser(@PathVariable Integer userId);

    @GetMapping("/getServerPort")
    String getServerPort();
}

```

fallback实现

```java
public class UserServiceFallback implements UserService {

    @Override
    public UserVO detailUser(Integer userId) {
        return new UserVO(userId, "fallback-user", 20);
    }

    @Override
    public String getServerPort() {
        return null;
    }
}
```

将fallback实现放到子容器中

```java
/**
 * 类上不加@Configuration注解, 是不想让该配置全局生效
 */
public class NacosProducerFeignConfig {

    /**
     * 放到nacos-producer子容器中
     */
    @Bean
    public UserService userServiceFallback() {
        return new UserServiceFallback();
    }
}

```

**nacos-producer接口实现效果是，如果userId>0，则正常返回，否则抛出一个不合法的参数异常**

测试

输入`http://localhost:8083/sentinel/detailUser/1`正常返回

输入`http://localhost:8083/sentinel/detailUser/-1`发生降级处理，执行fallback指定的实现。

