### Spring Boot如何自动装配Jackson

Jackson的自动配置类为`JacksonAutoConfiguration`，由spring-boot-autoconfigure-*.jar中的META-INF/spring.factories文件中指定，如下所示：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
...
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
...
```

自动配置过程如下:

```java
@Configuration
@ConditionalOnClass(Jackson2ObjectMapperBuilder.class)
static class JacksonObjectMapperConfiguration {

    /**
     * 注册ObjectMapper对象,其中通过Jackson2ObjectMapperBuilder类构建ObjectMapper
     */
    @Bean
    @Primary
    @ConditionalOnMissingBean
    public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        return builder.createXmlMapper(false).build();
    }

}

@Configuration
@ConditionalOnClass(Jackson2ObjectMapperBuilder.class)
static class JacksonObjectMapperBuilderConfiguration {

    private final ApplicationContext applicationContext;

    JacksonObjectMapperBuilderConfiguration(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    /**
     * 创建Jackson2ObjectMapperBuilder对象，从上一步可知，创建ObjectMapper时需要该对象。
     * 其中可以通过Jackson2ObjectMapperBuilderCustomizer来定义Jackson2ObjectMapperBuilder。
     * 因此自定义Jackson行为的方式就呼之欲出了，只要在应用中注册一个我们的
     * Jackson2ObjectMapperBuilderCustomizer对象到容器中即可。
     */
    @Bean
    @ConditionalOnMissingBean
    public Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder(
        List<Jackson2ObjectMapperBuilderCustomizer> customizers) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
        builder.applicationContext(this.applicationContext);
        customize(builder, customizers);
        return builder;
    }
}
```

### 自定义Jackson行为

如以下代码所示

```java
package com.wangtao.springboottest.config;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.datatype.jdk8.Jdk8Module;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalTimeSerializer;
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;
import org.springframework.stereotype.Component;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

@Component
public class JacksonCustomizer implements Jackson2ObjectMapperBuilderCustomizer {

    private static final String STANDARD_PATTERN = "yyyy-MM-dd HH:mm:ss";

    private static final String DATE_PATTERN = "yyyy-MM-dd";

    private static final String TIME_PATTERN = "HH:mm:ss";

    @Override
    public void customize(Jackson2ObjectMapperBuilder builder) {
        // 初始化JavaTimeModule
        JavaTimeModule javaTimeModule = new JavaTimeModule();

        //处理LocalDateTime
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter
            .ofPattern(STANDARD_PATTERN);
        javaTimeModule.addSerializer(LocalDateTime.class, 
                                     new LocalDateTimeSerializer(dateTimeFormatter));
        javaTimeModule.addDeserializer(LocalDateTime.class, 
                                       new LocalDateTimeDeserializer(dateTimeFormatter));

        //处理LocalDate
        DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern(DATE_PATTERN);
        javaTimeModule.addSerializer(LocalDate.class, 
                                     new LocalDateSerializer(dateFormatter));
        javaTimeModule.addDeserializer(LocalDate.class, 
                                       new LocalDateDeserializer(dateFormatter));

        //处理LocalTime
        DateTimeFormatter timeFormatter = DateTimeFormatter.ofPattern(TIME_PATTERN);
        javaTimeModule.addSerializer(LocalTime.class, 
                                     new LocalTimeSerializer(timeFormatter));
        javaTimeModule.addDeserializer(LocalTime.class, 
                                       new LocalTimeDeserializer(timeFormatter));

        /*
         * 1. java.util.Date yyyy-MM-dd HH:mm:ss
         * 2. 支持JDK8 LocalDateTime、LocalDate、 LocalTime
         * 3. Jdk8Module模块支持如Stream、Optional等类
         * 4. 序列化时包含所有字段
         * 5. 在序列化一个空对象时时不抛出异常
         * 6. 忽略反序列化时在json字符串中存在, 但在java对象中不存在的属性
         */
        builder.simpleDateFormat(STANDARD_PATTERN)
                .modules(javaTimeModule, new Jdk8Module())
                .serializationInclusion(JsonInclude.Include.ALWAYS)
                .failOnEmptyBeans(false)
                .failOnUnknownProperties(false);
    }
}

```

