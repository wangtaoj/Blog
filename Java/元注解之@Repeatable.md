### 简介
`@Repeatable`注解从JDK 1.8开始引入，为了解决注解不能在同一个地方重复使用而出现的。在JDK 1.8以前，下面这段代码将不能通过编译。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Identity {

    String value();
}

// 不能重复使用
@Identity("父亲")
@Identity("教师")
public class RepeatableTest {

}

```

### 使用

先定义注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Identities.class)
@Documented
public @interface Identity {

    String value();
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Identities {

    Identity[] value();
}

```

为了使得`@Identity`注解可以重复使用，需要注意以下几点。

* 在定义时用`@Repeatable`标记一个容器注解`@Identities`
* 该容器注解`@Identities`必须要声明一个value方法，并且返回值是一个`Identity[]`类型，否则编译报错。

测试类

```java
@Identity("父亲")
@Identity("教师")
public class RepeatableTest {

    public static void main(String[] args) {
        // 获取注解信息
        // 值得注意的是参数传的是Identities.class，而不是Identity.class。
        // 原因看下面原理介绍
        Identities identitiesAnnotation = RepeatableTest.class.
            getAnnotation(Identities.class);
        Identity[] identities = identitiesAnnotation.value();
        Arrays.stream(identities).forEach(identity -> 
           System.out.println(identity.value()));
    }
}
```

### 原理

如果我们将编译后的RepeatableTest.class文件反编译下，将得到以下源代码。

```java
@Identities({@Identity("父亲"), @Identity("教师")})
public class RepeatableTest {
    public RepeatableTest() {
    }
    
    // 省略main方法
}
```

可以看到重复使用`@Identity`注解其实是一个语法糖，这也是为什么上面`getAnnotation`方法传的参数是`Identities.class`的原因。继续使用`Class`类提供的方法做检测。

```java
@Identity("父亲")
@Identity("教师")
public class RepeatableTest {

    public static void main(String[] args) {
        // 返回false
        System.out.println(RepeatableTest.class.isAnnotationPresent(Identity.class));
        // 返回null
        System.out.println(RepeatableTest.class.getAnnotation(Identity.class));
        // 由此可见RepeatableTest类中确实没有被Identity直接注解。
        
        // 另外一种简便的方法获取注解信息
        // 使用getAnnotationsByType方法, 此方法为JDK 1.8新增
        Identity[] identities = RepeatableTest.class
            .getAnnotationsByType(Identity.class);
        Arrays.stream(identities).forEach(identity ->
                System.out.println(identity.value()));

        Identities[] identitiesAnnotation = RepeatableTest.class.
            getAnnotationsByType(Identities.class);
        // 输出父亲
        System.out.println(identitiesAnnotation[0].value()[0].value());
    }
}
```

