### 作用

`@JacksonXmlElementWrapper` 是 Jackson XML 模块提供的注解，用于控制`Collection`或`Array`字段在 XML 序列化/反序列化时是否使用包装元素，以及包装元素的名称和命名空间。

注：该注解只对`Collection`和`Array`字段有效。

### 行为解释

`XmlMapper`默认开启包装行为，可通过以下代码手动开启或关闭

```java
 XmlMapper mapper = XmlMapper.builder().defaultUseWrapper(false).build();
```

上面是一个全局开关，如果开启，即便字段上没有`@JacksonXmlElementWrapper`注解，也会添加包装元素，包装元素的名称就是属性名。

```java
@Target({ElementType.ANNOTATION_TYPE, ElementType.FIELD, ElementType.METHOD,ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface JacksonXmlElementWrapper
{
    
    public final static String USE_PROPERTY_NAME = "";
    
    String namespace() default USE_PROPERTY_NAME;
    
    /**
     * 包装元素名称
     * 默认为字段名称(@JacksonXmlProperty注解可以指定字段名称，若没有明确指定，则就是java字段名)
     */
    String localName() default USE_PROPERTY_NAME;

    /**
     * 用于控制是否添加包装元素, 可覆盖全局设置
     */
    boolean useWrapping() default true;
}
```

### 使用示例

#### 不加任何注解

```java
public class XmlTest {

    @AllArgsConstructor
    @Data
    public static class Item {
        public String id;
        public String name;
    }


    @AllArgsConstructor
    @Data
    public static class SingleContainer {

        public List<Item> itemList;
    }

    @Test
    public void testWrap() {
         SingleContainer obj = new SingleContainer(
            Arrays.asList(
                new Item("1", "a"),
                new Item("2", "b"))
        );

        String xml = XmlUtils.objToXml(obj);
        System.out.println(xml);
    }
}
```

结果如下:

```xml
<?xml version='1.0' encoding='UTF-8'?>
<SingleContainer>
  <itemList>
    <itemList>
      <id>1</id>
      <name>a</name>
    </itemList>
    <itemList>
      <id>2</id>
      <name>b</name>
    </itemList>
  </itemList>
</SingleContainer>
```

可以看到包装元素名称默认就是字段名。

#### 使用@JacksonXmlProperty注解指定字段名

```java
public class XmlTest {

    @AllArgsConstructor
    @Data
    public static class Item {
        public String id;
        public String name;
    }


    @AllArgsConstructor
    @Data
    public static class SingleContainer {

        @JacksonXmlProperty(localName = "item")
        public List<Item> itemList;
    }

    @Test
    public void testWrap() {
        SingleContainer obj = new SingleContainer(
            Arrays.asList(
                new Item("1", "a"),
                new Item("2", "b"))
        );

        String xml = XmlUtils.objToXml(obj);
        System.out.println(xml);
    }
}
```

结果

```xml
<?xml version='1.0' encoding='UTF-8'?>
<SingleContainer>
  <item>
    <item>
      <id>1</id>
      <name>a</name>
    </item>
    <item>
      <id>2</id>
      <name>b</name>
    </item>
  </item>
</SingleContainer>
```

可以看到包装元素会跟随字段名一起变化

#### 使用@JacksonXmlElementWrapper

```java
public class XmlTest {

    @AllArgsConstructor
    @Data
    public static class Item {
        public String id;
        public String name;
    }


    @AllArgsConstructor
    @Data
    public static class SingleContainer {

        @JacksonXmlElementWrapper(localName = "itemList")
        @JacksonXmlProperty(localName = "item")
        public List<Item> itemList;
    }

    @Test
    public void testWrap() {
        SingleContainer obj = new SingleContainer(
            Arrays.asList(
                new Item("1", "a"),
                new Item("2", "b"))
        );

        String xml = XmlUtils.objToXml(obj);
        System.out.println(xml);
    }
}
```

结果

```xml
<?xml version='1.0' encoding='UTF-8'?>
<SingleContainer>
  <itemList>
    <item>
      <id>1</id>
      <name>a</name>
    </item>
    <item>
      <id>2</id>
      <name>b</name>
    </item>
  </itemList>
</SingleContainer>
```

可以看到包装元素名称变成了itemList。

也就是说`@JacksonXmlElementWrapper`指定包装元素名，`@JacksonXmlProperty`指定集合项元素名，若`@JacksonXmlElementWrapper`没有指定，包装元素名就是`@JacksonXmlProperty`的值。而`@JacksonXmlProperty`的值默认为java字段名，当然也可自己指定。

### 使用限制

如果一个类中有多个集合类型，想要包装元素名不一样，而集合项元素名一样，使用这两个注解配合是无法做到的。因为集合项元素名是通过`@JacksonXmlProperty`指定，如果一样则会报名字冲突。就像java类中字段名也不能冲突一样。

```java
public class XmlTest {

    @AllArgsConstructor
    @Data
    public static class Item {
        public String id;
        public String name;
    }


    @AllArgsConstructor
    @Data
    public static class MultiContainer {

        @JacksonXmlElementWrapper(localName = "itemList1")
        @JacksonXmlProperty(localName = "item")
        public List<Item> itemList1;

        @JacksonXmlElementWrapper(localName = "itemList2")
        @JacksonXmlProperty(localName = "item")
        public List<Item> itemList2;
    }

    @Test
    public void testWrap() {
        MultiContainer obj = new MultiContainer(
            Arrays.asList(new Item("1", "a"), new Item("2", "b")),
            Arrays.asList(new Item("3", "a"), new Item("4", "b"))
        );

        String xml = XmlUtils.objToXml(obj);
        System.out.println(xml);
    }
}
```

报错如下

com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Conflicting getter definitions for property "item"

### 使用自定义序列化解决如上问题

#### 实现

包装元素还是交由`@JacksonXmlElementWrapper`指定，自定义序列化器用来替换集合元素项名称。

XmlElement.java

```java
/**
 * 自定义注解，用于指定 XML 元素名称。
 * 配合 {@link ObjectXmlSerializer} 使用，可在序列化集合类型时指定集合项元素名称。
 * <p>
 * 设计目的：
 * 当同一个对象中有多个集合类型（如 List），且希望它们的包装名不同，但集合项元素名称相同时，
 * 使用 Jackson 原生的 {@code @JacksonXmlElementWrapper} 和 {@code @JacksonXmlProperty} 注解无法满足需求，
 * 通过本注解 + {@link ObjectXmlSerializer} 可以实现该功能。
 * </p>
 *
 * @see ObjectXmlSerializer
 */
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface XmlElement {

    /**
     * 指定 XML 元素的本地名称（即标签名）。
     *
     * @return 元素名称
     */
    String localName();
}
```

ObjectXmlSerializer.java

```java
/**
 * 自定义 Jackson 序列化器，用于指定 XML 元素的标签名。
 * 实现 {@link ContextualSerializer} 接口，以便在序列化时动态获取 {@link XmlElement} 注解中指定的名称。
 * <p>
 * 使用方式：
 * 在需要自定义名称的 List 或容器字段上，同时标注：
 * <pre>{@code
 * @XmlElement(localName = "bean")
 * @JsonSerialize(contentUsing = ObjectXmlSerializer.class)
 * private List<Bean> beanList;
 * }</pre>
 * <p>
 * 序列化结果示例：
 * <pre>{@code
 * <ContainerBean>
 *     <BeanList1>
 *         <bean>...</bean>
 *     </BeanList1>
 *     <BeanList2>
 *         <bean>...</bean>
 *     </BeanList2>
 * </ContainerBean>
 * }</pre>
 *
 * @see XmlElement
 * @author wangtao20
 * Created at 2026-08-19
 */
public class ObjectXmlSerializer extends JsonSerializer<Object> implements ContextualSerializer {

    /**
     * XML元素名称
     */
    private String elementName;

    public ObjectXmlSerializer() {

    }

    /**
     * 带元素名称的构造方法，由 {@link #createContextual} 调用。
     *
     * @param elementName XML元素名称
     */
    public ObjectXmlSerializer(String elementName) {
        this.elementName = elementName;
    }

    /**
     * 序列化方法，覆盖当前节点的元素名称。
     *
     * @param object     要序列化的对象
     * @param gen        JSON 生成器（实际为 {@link ToXmlGenerator}）
     * @param serializers 序列化提供者
     * @throws IOException 序列化异常
     */
    @Override
    public void serialize(Object object, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        if (this.elementName == null) {
            throw new IllegalArgumentException("elementName is null, please use @XmlElement to set the element name");
        }

        /*
         * 此时的nextName为@JacksonXmlProperty指定的名称
         * 将当前节点元素名替换为指定的elementName
         */
        ToXmlGenerator xmlGen = (ToXmlGenerator) gen;
        xmlGen.setNextName(new QName(this.elementName));

        // 委托给默认序列化器处理值内容
        serializers.defaultSerializeValue(object, gen);
    }

    /**
     * 上下文序列化工厂方法，用于从字段/方法上读取 {@link XmlElement} 注解，
     * 并构造带有指定元素名称的序列化器实例。
     *
     * @param prov     序列化提供者
     * @param property 当前序列化的属性（字段或方法）
     * @return 如果注解存在则返回新的 {@link ObjectXmlSerializer} 实例，否则返回 null（表示不干预）
     */
    @Override
    public JsonSerializer<?> createContextual(SerializerProvider prov, BeanProperty property) {
        XmlElement annotation = property.getAnnotation(XmlElement.class);
        // 若用户指定了容器元素名称，则使用用户指定的名称
        if (annotation != null) {
            return new ObjectXmlSerializer(annotation.localName());
        }
        return this;
    }
}
```

提供一个默认的集合项元素名为**sdo**的序列化器，与JAXB类似。

SdoObjectXmlSerializer.java

```java
/**
 * 默认集合项元素名称为sdo, 与JAXB风格保持一致
 */
public class SdoObjectXmlSerializer extends ObjectXmlSerializer {

    public SdoObjectXmlSerializer() {
        super("sdo");
    }
}
```

#### 使用1

```java
public class XmlTest {

    @AllArgsConstructor
    @Data
    public static class Item {
        public String id;
        public String name;
    }


    @AllArgsConstructor
    @Data
    public static class MultiContainer {

        /**
         * @JacksonXmlElementWrapper可以省略，默认值为java字段名
         * @JsonSerialize使用contentUsing而不是using
         * contentUsing针对的是集合中的元素
         */
        @JsonSerialize(contentUsing = ObjectXmlSerializer.class)
        @XmlElement(localName = "item")
        @JacksonXmlElementWrapper(localName = "itemList1")
        public List<Item> itemList1;

        @JsonSerialize(contentUsing = ObjectXmlSerializer.class)
        @XmlElement(localName = "item")
        @JacksonXmlElementWrapper(localName = "itemList2")
        public List<Item> itemList2;
    }

    @Test
    public void testWrap() {
        MultiContainer obj = new MultiContainer(
            Arrays.asList(new Item("1", "a"), new Item("2", "b")),
            Arrays.asList(new Item("3", "a"), new Item("4", "b"))
        );

        String xml = XmlUtils.objToXml(obj);
        System.out.println(xml);
    }
}

```

结果

```xml
<?xml version='1.0' encoding='UTF-8'?>
<MultiContainer>
  <itemList1>
    <item>
      <id>1</id>
      <name>a</name>
    </item>
    <item>
      <id>2</id>
      <name>b</name>
    </item>
  </itemList1>
  <itemList2>
    <item>
      <id>3</id>
      <name>a</name>
    </item>
    <item>
      <id>4</id>
      <name>b</name>
    </item>
  </itemList2>
</MultiContainer>
```

#### 使用2

```java
public class XmlTest {

    @AllArgsConstructor
    @Data
    public static class Item {
        public String id;
        public String name;
    }


    @AllArgsConstructor
    @Data
    public static class MultiContainer {

        @JsonSerialize(contentUsing = SdoObjectXmlSerializer.class)
        public List<Item> itemList1;

        @JsonSerialize(contentUsing = SdoObjectXmlSerializer.class)
        public List<Item> itemList2;
    }

    @Test
    public void testWrap() {
        MultiContainer obj = new MultiContainer(
            Arrays.asList(new Item("1", "a"), new Item("2", "b")),
            Arrays.asList(new Item("3", "a"), new Item("4", "b"))
        );

        String xml = XmlUtils.objToXml(obj);
        System.out.println(xml);
    }
}
```

结果

```xml
<?xml version='1.0' encoding='UTF-8'?>
<MultiContainer>
  <itemList1>
    <sdo>
      <id>1</id>
      <name>a</name>
    </sdo>
    <sdo>
      <id>2</id>
      <name>b</name>
    </sdo>
  </itemList1>
  <itemList2>
    <sdo>
      <id>3</id>
      <name>a</name>
    </sdo>
    <sdo>
      <id>4</id>
      <name>b</name>
    </sdo>
  </itemList2>
</MultiContainer>
```

