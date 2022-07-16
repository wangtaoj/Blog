### 1. yaml 配置

```yaml
user:
  names:
    - name1
    - name2
  members:
    - id: 1
      username: name1
      age: 1
    - id: 2
      username: name2
      age: 2
  props:
    username: xxx
    gender: male
  birthday: "19980101"
  create-time: "2021-08-27 18:00:00"
```

### 2. Java 类
UserProperties.java

```java
@ConfigurationProperties(prefix = "user")
@Component
@ToString
@Getter
@Setter
public class UserProperties {

    private List<String> names;

    private List<User> members;

    private Map<String, String> props;

    @DateTimeFormat(pattern = "yyyyMMdd")
    private LocalDate birthday;

    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date createTime;
}
```

### 3. 解析

* 对于List、Set类型，值用**-**分割，且中间带有空格。
* 对于Map类型，Key、Value另起一行且缩进
* 对于时间类型，可使用`@DateTimeFormat`指定日期格式，注意19980101若不加引号，将被视为整形，转成LocalDate类型时将会报错，可以使用双引号包裹
* Java中需要使用`@ConfigurationProperties`注解来注入属性值，简单使用`@Value`注解不生效

