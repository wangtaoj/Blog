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
@Component
public class JacksonCustomizer implements Jackson2ObjectMapperBuilderCustomizer {

    public JacksonCustomizer() {
        super();
    }

    @Override
    public void customize(Jackson2ObjectMapperBuilder builder) {
        // 初始化JavaTimeModule
        JavaTimeModule javaTimeModule = JavaTimeModuleUtils.create();

        SimpleModule simpleModule = new SimpleModule();
        // 添加BigDecimal的自定义序列化器
        simpleModule.addSerializer(new BigDecimalSerializer());

        /*
         * 1. java.util.Date yyyy-MM-dd HH:mm:ss
         * 2. 支持JDK8 LocalDateTime、LocalDate、 LocalTime
         * 3. Jdk8Module模块支持如Stream、Optional等类
         * 4. 序列化时包含所有字段
         * 5. 在序列化一个空对象时时不抛出异常
         * 6. 忽略反序列化时在json字符串中存在, 但在java对象中不存在的属性
         * 7. BigDecimal.toPlainString()方法, 这样不会有科学计数法(序列化后仍是数字, 不是字符串)
         *    由于上面注册了自定义的BigDecimal序列化器, 该配置便没有效果了
         */
        builder.simpleDateFormat(JavaTimeModuleUtils.STANDARD_PATTERN)
                .timeZone(TimeZone.getDefault())
                .modules(javaTimeModule, new Jdk8Module(), simpleModule)
                .serializationInclusion(JsonInclude.Include.ALWAYS)
                .failOnEmptyBeans(false)
                .failOnUnknownProperties(false)
                .featuresToEnable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN);
    }
}

```

JavaTimeModuleUtils.java

```java
public final class JavaTimeModuleUtils {

    public static final String STANDARD_PATTERN = "yyyy-MM-dd HH:mm:ss";

    public static final String DATE_PATTERN = "yyyy-MM-dd";

    public static final String TIME_PATTERN = "HH:mm:ss";

    private JavaTimeModuleUtils() {}

    public static JavaTimeModule create() {
        // 初始化JavaTimeModule
        JavaTimeModule javaTimeModule = new JavaTimeModule();

        //处理LocalDateTime
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern(STANDARD_PATTERN);
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(dateTimeFormatter));
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(dateTimeFormatter));

        //处理LocalDate
        DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern(DATE_PATTERN);
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(dateFormatter));
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(dateFormatter));

        //处理LocalTime
        DateTimeFormatter timeFormatter = DateTimeFormatter.ofPattern(TIME_PATTERN);
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(timeFormatter));
        javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(timeFormatter));
        return javaTimeModule;
    }
}
```

BigDecimalSerializer.java

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
            gen.writeString(value.stripTrailingZeros().toPlainString());
        } else {
            gen.writeNull();
        }
    }
}
```

