### 概要

Nacos做为Spring Cloud Alibaba的服务治理组件，可以提供服务注册、服务发现等功能。

Nacos官方文档地址

* [Nacos中文文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)
* [Nacos服务端下载地址](https://github.com/alibaba/nacos/releases)
* [Nacos Discovery](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery)

本节具体代码

[项目地址](https://github.com/wangtaoj/SpringCloudAlibaba)，见nacos-provider、nacos-consumer

### nacos安装

从上述下载地址中下载 nacos-server-$version.zip，本项目使用的使用的为 2.0.3 版本，然后解压即可。

单机模式启动

```bash
# linux
sh startup.sh -m standalone
# windows
startup.cmd -m standalone
```

关闭服务器

```bash
# linux
sh shutdown.sh
# windows
shutdown.cmd
```

启动后，可以访问http://localhost:8848/nacos/index.html地址，进入nacos管理控制台。默认用户名以及密码为nacos/nacos。

### 服务注册

新建nacos-provider maven项目，继承父项目springcloud-alibaba-parent。其中springcloud-alibaba-parent项目是已经建好的基项目，用于约束Spring Boot、Spring Cloud、Spring Cloud Alibaba的版本。

第一步，引入nacos服务发现依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

第二步，使用 `@EnableDiscoveryClient` 注解开启服务注册发现功能，在Spring Boot项目的启动类加上该注解即可。

第三步，在application.yml文件编写nacos需要的配置

```yaml
spring:
  application:
  	# 向nacos服务端注册的服务名
    name: nacos-producer
  cloud:
    nacos:
      discovery:
        # nacos服务端地址
        server-addr: 127.0.0.1:8848
```

### 服务调用(ribbon)

两个不同的微服务调用时，可通过Spring提供的RestTemplate进行http调用，而这有一个弊端就是要写死ip以及端口信息。通过整合ribbon以及feign，便可以通过服务名来访问服务接口。ribbon会把服务名翻译成对应的ip和端口，如果这个服务有多个实例时，默认的策略是轮询，这会起到一个负载均衡的作用。而feign的作用是帮我们更加简化调用写法，不用显示使用RestTemplate来发请求，通过接口像写Spring MVC Controller方法一样。

新建nacos-consumer maven项目，然后集成ribbon

第一步，引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

第二步，整合RestTemplate和ribbon，只需一个注解，创建RestTemplate时，加入@LoadBalanced

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

import java.time.Duration;

import static java.time.temporal.ChronoUnit.SECONDS;

@Configuration
public class RestTemplateConfig {

    /**
     * 集成ribbon
     */
    @LoadBalanced
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.setConnectTimeout(Duration.of(30, SECONDS))
                .build();
    }
}

```

在nacos-producer服务编写一个用户接口，用于调用，代码如下

```java
package com.wangtao.nacosprovider.controller;

import com.wangtao.nacosprovider.vo.UserVO;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;
import java.util.Random;
import java.util.stream.Collectors;
import java.util.stream.IntStream;


@RequestMapping("/api/users")
@RestController
public class UserController {

    private static final Map<Integer, UserVO> MOCK_USERS_MAP;

    static {
        Random random = new Random();
        MOCK_USERS_MAP = IntStream.range(1, 11)
                .mapToObj(i -> new UserVO(i, "user-" + i, random.nextInt(101)))
                .collect(Collectors.toMap(UserVO::getId, u -> u));
    }

    @GetMapping("/{id}")
    public UserVO detailUser(@PathVariable Integer id) {
        return MOCK_USERS_MAP.get(id);
    }
}

```

现在要在nacos-consumer服务中调用该接口，实现一个邮件发送接口，代码如下所示

```java
package com.wangtao.nacosconsumer.controller;

import com.wangtao.nacosconsumer.service.UserService;
import com.wangtao.nacosconsumer.vo.UserVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RequestMapping("/api/email")
@RestController
public class EmailController {

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private UserService userService;

    @GetMapping("/sendEmail")
    public String sendEmail(Integer userId) {
        // 使用服务名替代ip和端口
        String url = "http://nacos-producer/api/users/" + userId;
        UserVO user = restTemplate.getForObject(url, UserVO.class);
        if (user == null) {
            return "send email error, not found user!";
        }
        return "send email to " + user.getName();
    }
}

```

### 服务调用(feign)

从上面看到服务调用写法还是略微有点繁琐，现在整合feign，可以做到像调用本地方法一样。

集成feign

第一步，引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

第二步，使用@EnableFeignClients开启feign的功能，会为被@FeignClient注解的接口创建代理类，在启动类加上该注解即可。

如何调用？

编写接口，且使用@FeignClient来标注

```java
@FeignClient("nacos-producer")
public interface UserService {

    @GetMapping("/api/users/{userId}")
    UserVO detailUser(@PathVariable Integer userId);
}
```

其中注解@FeignClient value属性填写要调用接口所在的服务名，然后注入该接口调用方法就行，如下所示

```java
package com.wangtao.nacosconsumer.controller;

import com.wangtao.nacosconsumer.service.UserService;
import com.wangtao.nacosconsumer.vo.UserVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RequestMapping("/api/email")
@RestController
public class EmailController {
    
    @Autowired
    private UserService userService;

    @GetMapping("/sendEmailByFeign")
    public String sendEmailByFeign(Integer userId) {
        UserVO user = userService.detailUser(userId);
        if (user == null) {
            return "send email error, not found user!";
        }
        return "send email to " + user.getName();
    }
}

```

集成feign之后，可以看到服务调用变得非常简洁了，大功告成。

