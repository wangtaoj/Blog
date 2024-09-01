### Jackson对于Java对象属性的可见性
Jackson在序列化时会对Java对象中哪些属性进行序列化呢?主要有以下几种
* 属性的修饰符为public
* 属性的修饰符为private, 但是提供public的get方法
* 对象中没有此属性, 但是类中有public的get方法

也就是说序列化时能不能够访问到此属性, 反序列化时能不能设置此属性的值.

**注:** 

1. 对于第三种情况, 键就是getXXX中的XXX, 值就是get方法的返回值。
2. 对于属性是boolean类型的(不包括包装类型Boolean), 那么该属性的get方法是isXXX. 如下:
`private boolean exist`
`public boolean isExist()`

当然也可以使用@JsonIgnore注解来指定忽略那个属性, 使这个属性不参与序列化和反序列化.

### Jackson对于BigDecimal的处理

默认情况下调用`BigDecimal`中的`toString`方法来返回值，在JSON中值是一个数字不是字符串

```json
{
    "amt": 10000000
}
```

但是`toString`方法会有些问题，有时候拿到的值不是数字的原本形式，可能会变成科学计数法

```java
/**
 * 对于一个很小的数字就会变成科学计数法
 * 经测试，非常大的数字不会
 */
@Test
public void testToString() {
    // 1E-7
    System.out.println(new BigDecimal("0.0000001"));
}

@Test
public void testJson() throws JsonProcessingException {
    ObjectMapper objectMapper = new ObjectMapper();
    Map<String, Object> map = Map.of("amt", new BigDecimal("0.0000001"));
    // {"amt":1E-7}
    System.out.println(objectMapper.writeValueAsString(map));
}
```

针对这种情况，我们可以通过配置，让jackson调用`BigDecimal`的`toPlainString`方法来避免这种行为

```java
@Test
public void testJson() throws JsonProcessingException {
    ObjectMapper objectMapper = new ObjectMapper();
    // 告诉jackson对于BigDecimal类型, 使用toPlainString方法
    objectMapper.enable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN);
    Map<String, Object> map = Map.of("amt", new BigDecimal("0.0000001"));
    // {"amt":0.0000001}
    System.out.println(objectMapper.writeValueAsString(map));
}
```

不过到目前为止，如果是一个非常大的数字，在前端JS时，可能会存在精度问题，可以让jackson将`BigDecimal`序列化成一个字符串。

Jackson提供了一个配置，**不过是针对所有数字的，所以不要使用它。**

```java
@Test
public void testJson() throws JsonProcessingException {
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.enable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN);
    objectMapper.enable(JsonWriteFeature.WRITE_NUMBERS_AS_STRINGS.mappedFeature());
    Map<String, Object> map = Map.of("amt", new BigDecimal("0.0000001"));
    // {"amt":"0.0000001"}
    System.out.println(objectMapper.writeValueAsString(map));
}
```

最好的办法是针对`BigDecimal`类型写一个序列化器

```java
public class BigDecimalSerializer extends StdSerializer<BigDecimal> {

    @Serial
    private static final long serialVersionUID = -82683766308966384L;

    public BigDecimalSerializer() {
        super(BigDecimal.class);
    }

    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        if (Objects.nonNull(value)) {
            // 字符串格式
            gen.writeString(value.stripTrailingZeros().toPlainString());
        } else {
            gen.writeNull();
        }
    }
}
```

使用，可以在POJO字段上单独使用，也可以全局注册

全局注册代码如下:

```java
SimpleModule simpleModule = new SimpleModule();
// 处理BigDecimal, 序列化时使用字符串
simpleModule.addSerializer(new BigDecimalSerializer());
// 注册模块
objectMapper.registerModules(simpleModule);
```

总结:

* `objectMapper.enable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN)`
* `objectMapper.enable(JsonWriteFeature.WRITE_NUMBERS_AS_STRINGS.mappedFeature())`，针对所有数字类型，不要使用
* 自定义BigDecimal的序列化器

### Jackson对于时间的处理

#### 默认情况下
* 对于java.util.Date、java.sql.Date、java.sql.Timestamp这些时间类, 都会序列化成一个long型的时间戳
* 对于Java8中新的时间类, 则会序列化成一个数组, 格式为[年, 月, 日, 时, 分, 秒, 毫秒] 或者 [年, 月, 日], 当然要使用新的时间类, 需要加入jsr310模块, 在初始化时需要注册时间模块
`objectMapper.registerModule(new JavaTimeModule());`

#### 序列化器

java.util.Date：DateSerializer

LocalDateTime：LocalDateTimeSerializer

LocalDate：LocalDateSerializer

LocalTime：LocalTimeDeserializer

最终序列化成什么形式，其实主要看的是序列化器中DateFormat(旧的日期API)、DateTimeFormatter(jsr310)

每次序列化时，若不是序列化成默认格式时(@JsonFormat注解可以修改当前字段的序列化格式)，会基于全局配置的DateFormat(DateTimeFormatter) + @JsonFormat注解的配置生成一个新的DateFormat(DateTimeFormatter)用来格式化，这个逻辑位于序列化器中的**createContextual**方法中。

#### 格式化

1. 全局处理

1.1 java.util.Date，即DateSerializer的配置。

```java
// 设置默认的DateFormat以及默认的时区，new SimpleDateFormat()默认时区是TimeZone.getDefault()
objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
// 内部同时会调整DateFormat的时区, 默认为UTC，而不是TimeZone.getDefault()
objectMapper.setTimeZone(TimeZone.getDefault());
```

如果单一设置DateFormat而不设置TimeZone，在存在@JsonFormat注解时，会产生一些诡异现象，因为@JsonFormat注解如果不指定timeZone属性，则会使用全局的timeZone，即UTC，会覆盖掉DateFormat中的时区TimeZone.getDefault()。

就会导致字段没有@JsonFormat注解，使用的是TimeZone.getDefault()，而存在@JsonFormat注解(没有明确指定时区)则使用的是UTC时区，从而格式化的结果相差8小时，国内属于东八区。

此方式对于java.util.Date、java.time.LocalDateTime、java.time.LocalDate(对于新的时间类前提是有注册过JavaTimeModule模块)序列化会产生一些变化。

**本质上对JSR310产生影响是因为`setDateFormat`方法会取消将日期序列化成时间戳这么一个全局特征，而不是对他们的序列化器做了什么修改，使得JSR310的序列化器采用序列化器默认的DateTimeFormatter去格式化，而不是产生一个数组[2018,10,17]这样。**

Date: 2018-10-27 14:30:30
LocalDateTime: 2018-10-27T14:30:30.475
LocalDate: 2018-10-27
LocalTime:14:30:30

1.2 jsr310，指定全局格式

```
JavaTimeModule javaTimeModule = new JavaTimeModule();
DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(dateTimeFormatter));
javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(dateTimeFormatter));
objectMapper.registerModule(javaTimeModule);
```

2. 局部处理
在字段上增加@JsonFormat注解，可以指定pattern、timeZone等属性。

### 一个自定义的JSON工具类
``` java
package com.wangtao.social.common.core.util;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.datatype.jdk8.Jdk8Module;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalTimeSerializer;
import com.wangtao.social.common.core.jackson.BigDecimalSerializer;

import java.io.IOException;
import java.text.SimpleDateFormat;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.Collections;
import java.util.List;
import java.util.TimeZone;

/**
 * @author wangtao
 * Created at 2023-09-23
 */
public class JsonUtils {

    private static final ObjectMapper objectMapper = new ObjectMapper();

    private static final String STANDARD_PATTERN = "yyyy-MM-dd HH:mm:ss";
    private static final String DATE_PATTERN = "yyyy-MM-dd";
    private static final String TIME_PATTERN = "HH:mm:ss";

    static {

        //设置java.util.Date时间类的序列化以及反序列化的格式
        objectMapper.setDateFormat(new SimpleDateFormat(STANDARD_PATTERN));

        /*
         * 设置时区, Jackson默认值为UTC
         * 设置成国内时区(Asia/Shanghai), 这样@JsonFormat中不用指定时区了
         * 同时会把上面的DateFormat时区设置成指定的这个值
         * 实际上new SimpleDateFormat()默认时区也是TimeZone.getDefault()
         * 在这里相当于仅仅修改了Jackson的timeZone属性, 这对于@JsonFormat注解会产生影响
         * 当@JsonFormat注解没有指定timezone属性时
         * DateSerializer：java.util.Date的序列化器, 会采用Jackson的设置的timeZone和上面设置的dateFormat
         * 生成一个新的DateFormat来进行序列化。
         * 因此字段中没有@JsonFormat注解，使用的DateFormat便是上面设置的
         * 存在@JsonFormat注解时，会基于上面设置的DateFormat，将@JsonFormat注解上的信息合成一个新的DateFormat
         *
         * 最佳实践: dateFormat和timeZone同时设置，保证一致，避免@JsonFormat中没有指定timezone属性时，而采用
         * 默认的UTC时区(ObjectMapper没有设置timeZone时)
         *
         * 注: 该方法对于JDK8新加的时间无效, 只影响DateFormat。JDK8新加的时间采用DateTimeFormatter
         *
         * 源码提示: 序列化器类中的createContextual方法
         */
        objectMapper.setTimeZone(TimeZone.getDefault());

        // 初始化JavaTimeModule
        JavaTimeModule javaTimeModule = new JavaTimeModule();

        // 处理LocalDateTime
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern(STANDARD_PATTERN);
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(dateTimeFormatter));
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(dateTimeFormatter));

        // 处理LocalDate
        DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern(DATE_PATTERN);
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(dateFormatter));
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(dateFormatter));

        // 处理LocalTime
        DateTimeFormatter timeFormatter = DateTimeFormatter.ofPattern(TIME_PATTERN);
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(timeFormatter));
        javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(timeFormatter));

        SimpleModule simpleModule = new SimpleModule();
        // 处理BigDecimal, 序列化时使用字符串
        simpleModule.addSerializer(new BigDecimalSerializer());

        // 注册模块
        objectMapper.registerModules(javaTimeModule, simpleModule);

        // 注册JDK新增的一些类型, 比如Optional
        objectMapper.registerModule(new Jdk8Module());

        // 包含所有字段
        objectMapper.setSerializationInclusion(JsonInclude.Include.ALWAYS);

        // 在序列化一个空对象时时不抛出异常
        objectMapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);

        // 忽略反序列化时在json字符串中存在, 但在java对象中不存在的属性
        objectMapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        // BigDecimal.toPlainString(), 这样不会有科学计数法(序列化后仍是数字, 不是字符串)
        objectMapper.enable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN);
    }

    private JsonUtils() {

    }

    /**
     * 将Java对象序列化成一个JSON对象或者JSON数组.
     *
     * @param object Java对象
     * @return 返回一个JSON格式的字符串
     */
    public static String objToJson(Object object) {
        if (object == null) {
            return null;
        }
        try {
            return objectMapper.writeValueAsString(object);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException(String.format("parse %s to json error", object), e);
        }
    }

    /**
     * 将JSON对象反序列化成一个Java原生对象, 不支持泛型.
     *
     * @param json JSON对象
     * @param cls  Java对象原始类型的class对象
     * @param <T>  Java对象的原始类型
     * @return 返回一个T类型的对象
     */
    public static <T> T jsonToObj(String json, Class<T> cls) {
        if (json == null || json.isEmpty()) {
            return null;
        }
        try {
            return objectMapper.readValue(json, cls);
        } catch (IOException e) {
            throw new IllegalArgumentException(String.format("parse %s to obj error", json), e);
        }
    }

    /**
     * 将JSON反序列化成一个Java对象, 支持泛型.
     * TypeReference是一个抽象类, 用来构造类型
     * 调用方式: 传入一个TypeReference的匿名实现类即可
     * User user = jsonToObj(json, new TypeReference<User>(){})
     * List<User> users = jsonToObj(json, new TypeReference<List<User>>(){})
     *
     * @param json          JSON对象
     * @param typeReference 类型引用
     * @param <T>           返回值类型
     * @return 返回一个Java对象
     */
    public static <T> T jsonToObj(String json, TypeReference<T> typeReference) {
        if (json == null || json.isEmpty()) {
            return null;
        }
        try {
            return objectMapper.readValue(json, typeReference);
        } catch (Exception e) {
            throw new IllegalArgumentException(String.format("parse %s to obj error", json), e);
        }
    }

    /**
     * 将一个JSON数组反序列化成一个List对象.
     *
     * @param json JSON数组
     * @param cls  Java对象原始类型的class对象
     * @param <T>  Java对象的原始类型
     * @return 返回一个List列表
     */
    public static <T> List<T> jsonToList(String json, Class<T> cls) {
        if (json == null || json.isEmpty()) {
            return Collections.emptyList();
        }
        try {
            JavaType javaType = objectMapper.getTypeFactory().constructCollectionType(List.class, cls);
            return objectMapper.readValue(json, javaType);
        } catch (IOException e) {
            throw new IllegalArgumentException(String.format("parse %s to obj error", json), e);
        }
    }
}

```