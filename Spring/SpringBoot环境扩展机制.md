### 前言

Spring Boot在启动时，会先创建`Environment`实例，然后再创建`ApplicationContext`上下文。在创建`Environment`时，提供了扩展机制给用户对`Environment`实例进行修改，如Spring Boot默认使用的application.yml属性配置文件。

### 如何使用该机制

* 编写类实现`EnvironmentPostProcessor`接口。
* 在类路径下的META-INF目录下建立`spring.factories`，然后将类写到该文件中。

例子代码如下所示

```java
package com.wangtao.springboottest.env;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.env.EnvironmentPostProcessor;
import org.springframework.boot.env.YamlPropertySourceLoader;
import org.springframework.core.Ordered;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.PropertySource;
import org.springframework.core.io.ClassPathResource;

import java.io.IOException;
import java.util.List;

public class UserEnvPostProcessor implements EnvironmentPostProcessor, Ordered {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        // 加载类路径下的yml文件, 并将其添加到Spring环境中
        final String filename = "user.yml";
        YamlPropertySourceLoader loader = new YamlPropertySourceLoader();
        try {
            List<PropertySource<?>> propertySources = loader.load(filename, new ClassPathResource(filename));
            propertySources.forEach(source -> environment.getPropertySources().addLast(source));
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 最低优先级执行, 低于别的EnvironmentPostProcessor实例执行
     */
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}

```

spring.factories

```properties
org.springframework.boot.env.EnvironmentPostProcessor=\
com.wangtao.springboottest.env.UserEnvPostProcessor
```

注: key固定为`org.springframework.boot.env.EnvironmentPostProcessor`，value为自定义类的全路径名，可以指定多个，逗号分割。

### 验证

```java
package com.wangtao.springboottest.env;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

@Data
@ConfigurationProperties(prefix = "user")
public class UserProperties {

    private String nickname;

    private Integer age;
}
```

启动类

```java
package com.wangtao.springboottest;

@EnableConfigurationProperties({UserProperties.class})
@SpringBootApplication
public class SpringBootTestApplication implements SchedulingConfigurer {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootTestApplication.class, args);
    }
}
```

user.yml

```yaml
user:
  nickname: wangtao
  age: 25
```

### 总结

使用这种方式加载属性配置文件有一个好处，会在`ApplicationContext`上下文创建之前就加载好。而`@PropertySource`这种方式是在上下文创建的过程中随着bean的注册而加载的，并且默认情况下该注解不支持yml格式。

