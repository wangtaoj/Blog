### 基本使用

```java
@Component
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/asset/**")
            .addResourceLocations("classpath:/asset/");
    }
}
```

只需要实现`WebMvcConfigurer`接口，重写其中的`addResourceHandlers`方法即可。

其中路径遵循`Ant Match`语法规则，资源位置(resourceLocations)遵循`Spring Resource`路径语法，内部会将字符串转成`Resource`实例。

简要介绍`Ant Match`语法

* **?**：匹配单个字符

* **\***：匹配零个或者多个字符
* ******：匹配零个或者多个目录(包括子目录)

### 定位静态资源逻辑

光看样例其实对于资源的定位逻辑其实很迷惑，就和nginx的反向代理一样。

定位资源取决于两个要素，一个是从请求路径获取资源的相对路径部分，另外就是配置的资源所在的目录。

#### 获取资源相对路径规则

* 如果没有通配符，就是请求路径，比如映射配置为/asset/doc.html，那么就是/asset/doc.html。

* 如果存在通配符，那么就是取通配符匹配的那部分。

  举例如下

  映射配置为/asset/**，请求路径为/asset/doc.html，那么取得的值为doc.html

  映射配置为/asset/**，请求路径为/asset/js/main.js，那么取得的值为js/main.js

  映射配置为/asset/*.html，请求路径为/asset/index.html，那么取得值index.html

通配符取相对路径值可参考`AntPathMatcher.extractPathWithinPattern`方法

#### 获取资源最终路径

将获取到的相对资源路径和配置的资源所在的目录结合就是最终资源位置。

使用`Resource.createRelative`方法获取，对于`ClassPathResource`来说，**如果当前资源路径存在**/**，则拼接，如果不存在，则只使用参数部分。**

```java
Resource resource = new ClassPathResource("asset/");
// 表示类路径下的asset/doc.html
Resource relative = resource.createRelative("doc.html");

Resource resource = new ClassPathResource("asset");
// 表示类路径下的doc.html，丢弃了asset，因为后面没有/
Resource relative = resource.createRelative("doc.html");
```

因此对于上面示例，若请求路径为/asset/js/index.js，则定位的资源即类路径下的/asset/js/index.js

### 常用例子

```java
/**
 * /asset/js/index.js
 * 最终资源: js/index.js + /static/ = /static/js/index.js
 */
registry.addResourceHandler("/asset/**").addResourceLocations("classpath:/static/");

/**
 * /asset/view/index.html
 * 最终资源: view/index.html + /static/ = /static/view/index.html
 *
 * /asset/index.html
 * 最终资源: index.html + /static/ = /static/index.html
 */
registry.addResourceHandler("/asset/**/*.html").addResourceLocations("classpath:/static/");

/**
 * 无通配符
 * 请求路径: /asset/index.html
 * 最终资源: /asset/index.html + /static/ = /static/asset/index.html
 * 多余的前导斜杠会自动去掉
 */
registry.addResourceHandler("/asset/index.html").addResourceLocations("classpath:/static/");

/**
 * 资源目录后面少写斜杠，资源目录丢弃
 * /asset/js/index.js
 * 最终资源: js/index.js = js/index.js
 * 
 * 注: 这样子即便找到了该资源，springmvc会做一个检查，检查这个资源是否在允许的目录下，这个允许的
 * 目录默认就是配的资源路径目录即/static，因此仍然会报404，资源不存在的。
 * 因此资源目录务必末尾添加斜杠
 */
registry.addResourceHandler("/asset/**").addResourceLocations("classpath:/static");
```

**注：资源目录路径末尾务必要增加斜杠，用于暗示是目录，否则会丢掉这个目录，导致出现资源找不到现象。**

### Spring Boot默认配置的资源映射

在Spring Boot中，默认为配置请求路径为/**，即拦截所有路径，资源目录位置为以下几个

* classpath:/META-INF/resources/
* classpath:/resources/
* classpath:/static/
* classpath:/public/

寻找的优先级从上到下，即classpath:/META-INF/resources/优先级最高

自动配置类为`WebMvcAutoConfiguration`，找到`addResourceHandlers`方法即可。

关闭默认的资源映射配置

```yaml
spring:
  web:
    resources:
      # 不要添加默认的ResourceHandler(/**等)
      add-mappings: false
```

### 底层依赖类

方便读取源码

* HandleMapping为SimpleUrlHandlerMapping。
* Handler为ResourceHttpRequestHandler。
* HandlerAdapter为HttpRequestHandlerAdapter。
* 自动配置为WebMvcConfigurationSupport类的resourceHandlerMapping方法，注册了一个SimpleUrlHandlerMapping。

