### 前言

在看Spring源码的时候，经常会有处理注解的时候，比如从方法上获取注解，类上获取注解，注解属性别名。JDK中自带的获取注解API有点简单，不会从父类方法或者接口上的方法去查找，不能为属性定义别名等，因此Spring封装了一个便利的工具类，更加方便的去获取注解信息。

### JDK自带方法

`AnnotatedElement`为获取注解的顶层接口，`Class`、`Method`都实现了这个接口。

关于元注解`@Inherited`

被`@Inherited`标记的注解是可以被子类继承的，注意必须在类上，接口或者方法上都是不能被继承的。

下面看下API的使用

先准备几个注解用来测试

```java
/**
 * 被@Inherited标记
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface InheriteClassCase {
}

/**
 * 被@Inherited标记
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface InheriteInterfaceCase {
}

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface NoInheriteCase {
}

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface OtherCase {
}
```

测试类如下

```java
public class JdkAnnotationTest {

    @InheriteClassCase
    @NoInheriteCase
    private static abstract class ParentClassCase {

        @InheriteClassCase
        abstract void invokeClass();
    }

    @InheriteInterfaceCase
    private interface ParentInterfaceCase {

        void invokeInterface();
    }

    @OtherCase
    private class SonCase extends ParentClassCase implements ParentInterfaceCase {

        @Override
        public void invokeClass() {

        }

        @Override
        public void invokeInterface() {

        }
    }

    @Test
    public void testApi() throws NoSuchMethodException {
        // 获取自己声明的注解, 只能拿到OtherCase
        Annotation[] declaredAnnotations = SonCase.class.getDeclaredAnnotations();
        Assert.assertEquals(1, declaredAnnotations.length);
        Assert.assertEquals(OtherCase.class, declaredAnnotations[0].annotationType());

        /*
         * 获取存在的注解(自己声明的或者继承的)
         * 目标注解必须被@Inherited标记才能被继承
         * 且必须在类上
         */
        InheriteClassCase inheriteClassCase = SonCase.class.getAnnotation(InheriteClassCase.class);
        Assert.assertNotNull(inheriteClassCase);

        /*
         * NoInheriteCase虽然在父类存在, 但是没有被@Inherited标记, 不能被继承
         */
        NoInheriteCase noInheriteCase = SonCase.class.getAnnotation(NoInheriteCase.class);
        Assert.assertNull(noInheriteCase);

        // InheriteInterfaceCase虽然被@Inherited标记, 但是在接口上, 也不能获取到
        InheriteInterfaceCase inheriteInterfaceCase = SonCase.class.getAnnotation(InheriteInterfaceCase.class);
        Assert.assertNull(inheriteInterfaceCase);

        Method invokeClassMethod = SonCase.class.getMethod("invokeClass");
        // InheriteClassCase在父类方法中, 子类方法并没有被标注, 因此不能获取到
        InheriteClassCase classCase = invokeClassMethod.getAnnotation(InheriteClassCase.class);
        Assert.assertNull(classCase);
    }
}
```

### Spring AnnotationUtils

该类主要增强了以下几点

* 查找元注解，`getAnnotation`和`findAnnotation`都支持。
* 查找范围扩大，可在整个继承体系上查找(类 + 接口)，仅`findAnnotation`支持。
* 属性别名，通过@AliasFor注解，给属性起一个别名，这样指定一个属性，获取别名属性时也能拿到一样的值，`getAnnotation`和`findAnnotation`都支持。

何为元注解，一个注解A可以在注解B上注解，那么注解A称为注解B的元注解。一个类上存在注解B，该工具类可以直接获取到元注解A。

API使用

```java
public class AnnotationUtilsTest {

    private static class MetaAnnotationCase {
        @GetMapping("/request")
        public void request(){}
    }

    @InheriteClassCase
    @NoInheriteCase
    private static abstract class ParentClassCase {

        @GetMapping
        @InheriteClassCase
        abstract void invokeClass();
    }

    @InheriteInterfaceCase
    private interface ParentInterfaceCase {

        @GetMapping
        void invokeInterface();
    }

    @OtherCase
    private class SonCase extends ParentClassCase implements ParentInterfaceCase {

        @Override
        public void invokeClass() {

        }

        @Override
        public void invokeInterface() {

        }
    }

    /**
     * getAnnotation基本使用和AnnotatedElement.getAnnotation一样
     * 不过扩展了属性别名、元注解查找
     */
    @Test
    public void testGetAnnotation() throws NoSuchMethodException {
        Method requestMethod = MetaAnnotationCase.class.getMethod("request");

        // 直接存在
        GetMapping getMapping = AnnotationUtils.getAnnotation(requestMethod, GetMapping.class);
        Assert.assertNotNull(getMapping);

        // 查找元注解(RequestMapping注解了GetMapping)
        RequestMapping requestMapping = AnnotationUtils.getAnnotation(requestMethod, RequestMapping.class);
        Assert.assertNotNull(requestMapping);

        // JDK @Inherited语义
        InheriteClassCase inheriteClassCase = AnnotationUtils.getAnnotation(SonCase.class, InheriteClassCase.class);
        Assert.assertNotNull(inheriteClassCase);

        // 接口, @Inherited不生效
        InheriteInterfaceCase inheriteInterfaceCase = AnnotationUtils.getAnnotation(SonCase.class, InheriteInterfaceCase.class);
        Assert.assertNull(inheriteInterfaceCase);

        // 方法上, @Inherited不生效
        Method invokeClassMethod = SonCase.class.getDeclaredMethod("invokeClass");
        InheriteClassCase inheriteClassCaseAtMethod = AnnotationUtils.getAnnotation(invokeClassMethod, InheriteClassCase.class);
        Assert.assertNull(inheriteClassCaseAtMethod);
    }

    /**
     * 在getAnnotation基础上，扩展了搜索范围(类+接口)
     */
    @Test
    public void testFindAnnotation() throws NoSuchMethodException {
        Method invokeClassMethod = SonCase.class.getDeclaredMethod("invokeClass");
        Method invokeInterfaceMethod = SonCase.class.getDeclaredMethod("invokeInterface");

        // 命中父类上的注解(即便没有@Inherited)
        NoInheriteCase noInheriteCase = AnnotationUtils.findAnnotation(SonCase.class, NoInheriteCase.class);
        Assert.assertNotNull(noInheriteCase);

        // 命中接口上的注解
        InheriteInterfaceCase inheriteInterfaceCase = AnnotationUtils.findAnnotation(SonCase.class, InheriteInterfaceCase.class);
        Assert.assertNotNull(inheriteInterfaceCase);

        // 命中父类或者接口中的方法上的注解
        GetMapping getMapping = AnnotationUtils.findAnnotation(invokeInterfaceMethod, GetMapping.class);
        Assert.assertNotNull(getMapping);

        // 查找元注解(父类方法存在GetMapping)
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(invokeClassMethod, RequestMapping.class);
        Assert.assertNotNull(requestMapping);
    }

    /**
     * isAnnotationXXX
     */
    @Test
    public void testIsAnnotation() {
        // Class直接声明的注解，不会考虑@Inherited语义，也不会在继承体系上搜索
        Assert.assertFalse(AnnotationUtils.isAnnotationDeclaredLocally(InheriteClassCase.class, SonCase.class));
        Assert.assertTrue(AnnotationUtils.isAnnotationDeclaredLocally(InheriteClassCase.class, ParentClassCase.class));

        // 指定注解存在且是通过@Inherited继承的，不是直接声明的
        Assert.assertTrue(AnnotationUtils.isAnnotationInherited(InheriteClassCase.class, SonCase.class));
        Assert.assertFalse(AnnotationUtils.isAnnotationInherited(InheriteClassCase.class, ParentClassCase.class));

        // 判断一个注解类型是不是另外一个注解类型的元注解
        Assert.assertTrue(AnnotationUtils.isAnnotationMetaPresent(GetMapping.class, RequestMapping.class));
    }

    @Test
    public void testAttrAliasFor() throws NoSuchMethodException {
        Method requestMethod = MetaAnnotationCase.class.getDeclaredMethod("request");
        String[] value = new String[] {"/request"};
        GetMapping getMapping = AnnotationUtils.getAnnotation(requestMethod, GetMapping.class);
        Objects.requireNonNull(getMapping);
        Assert.assertArrayEquals(value, getMapping.value());
        // 虽然只指定了value的值, 但是path属性=value属性, 因为value和path通过@AliasFor互相指定了别名
        Assert.assertArrayEquals(value, getMapping.path());

        AnnotationAttributes annotationAttributes = AnnotationUtils.getAnnotationAttributes(requestMethod, getMapping);
        Assert.assertArrayEquals(value, annotationAttributes.getStringArray("value"));
        Assert.assertArrayEquals(value, annotationAttributes.getStringArray("path"));
    }
}
```

### Spring AnnotatedElementUtils

此类进一步增强了@AliasFor注解，可以合并属性

还是以@GetMapping、@RequestMapping为例

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = RequestMethod.GET)
public @interface GetMapping {

    /**
     * Alias for {@link RequestMapping#name}.
     */
    @AliasFor(annotation = RequestMapping.class)
    String name() default "";

    /**
     * Alias for {@link RequestMapping#value}.
     */
    @AliasFor(annotation = RequestMapping.class)
    String[] value() default {};
}
```

@GetMapping的定义对于value属性，是@RequestMapping注解的value属性别名，SpringMVC中真实起作用的其实是@RequestMapping注解，但是我们使用@GetMapping指定value属性时，SpringMVC在解析时需要拿到@RequestMapping注解的value属性，此时可以通过该工具类来获取。

例子：

```java
public class AnnotatedElementUtilsTest {

    private static class MetaAnnotationCase {
        @GetMapping("/request")
        public void request() {
        }
    }

    @Test
    public void testGetMetaAnnotationTypes() throws NoSuchMethodException {
        Method requestMethod = MetaAnnotationCase.class.getMethod("request");

        String[] res = {RequestMapping.class.getName(), Mapping.class.getName()};
        // 获取元注解类型, 会排除java.lang包下面的注解, 会递归搜索
        Set<String> metaAnnotationTypes = AnnotatedElementUtils.getMetaAnnotationTypes(requestMethod, GetMapping.class);
        Assert.assertArrayEquals(res, metaAnnotationTypes.toArray());
    }

    /**
     * 合并@AliasFor指定的别名属性
     */
    @Test
    public void testGetMergedAnnotationAttributes() throws NoSuchMethodException {
        Method requestMethod = MetaAnnotationCase.class.getDeclaredMethod("request");
        String[] value = new String[] {"/request"};
        String[] defaultValue = new String[] {};
        RequestMapping requestMapping = AnnotationUtils.getAnnotation(requestMethod, RequestMapping.class);
        Objects.requireNonNull(requestMapping);
        // 指定了GetMapping的value属性, 但是通过AnnotationUtils方式拿不到值
        Assert.assertArrayEquals(defaultValue, requestMapping.value());

        // 通过AnnotatedElementUtils的merge系列方法可以合并属性值
        // getXXX, 只会搜索自己和@Inherited语义继承的注解
        requestMapping = AnnotatedElementUtils.getMergedAnnotation(requestMethod, RequestMapping.class);
        Assert.assertNotNull(requestMapping);
        Assert.assertArrayEquals(value, requestMapping.value());

        // findXXX会寻找父类+接口
        requestMapping = AnnotatedElementUtils.findMergedAnnotation(requestMethod, RequestMapping.class);
        Assert.assertNotNull(requestMapping);
        Assert.assertArrayEquals(value, requestMapping.value());

        AnnotationAttributes mergedAnnotationAttributes = AnnotatedElementUtils.getMergedAnnotationAttributes(requestMethod, RequestMapping.class);
        Assert.assertNotNull(mergedAnnotationAttributes);
        Assert.assertArrayEquals(value, mergedAnnotationAttributes.getStringArray("value"));

        mergedAnnotationAttributes = AnnotatedElementUtils.findMergedAnnotationAttributes(requestMethod, RequestMapping.class, false, false);
        Assert.assertNotNull(mergedAnnotationAttributes);
        Assert.assertArrayEquals(value, mergedAnnotationAttributes.getStringArray("value"));

    }

}
```

getOrFindAll系列

```java
public class AnnotatedElementUtilsTest {

    private static class MetaAnnotationCase {
        @GetMapping("/request")
        public void request() {
        }

        @GetMapping("/getMapping")
        @RequestMapping("/requestMapping")
        public void repeat() {
        }
    }

    /**
     * 拿到所有指定注解列表
     */
    @Test
    public void testGetOrFindAll() throws NoSuchMethodException {
        Method repeatMethod = MetaAnnotationCase.class.getDeclaredMethod("repeat");
        // getXXX, 只会搜索自己和@Inherited语义继承的注解, 返回所有的, 不仅仅是第一个
        // 返回的是LinkedHashSet, 因此有顺序
        Set<RequestMapping> requestMappingSet = AnnotatedElementUtils.getAllMergedAnnotations(repeatMethod, RequestMapping.class);
        String[][] value = {{"/requestMapping"}, {"/getMapping"}};
        int i = 0;
        for (RequestMapping requestMapping : requestMappingSet) {
            Assert.assertArrayEquals(value[i++], requestMapping.value());
        }

        // findXXX, 会搜索类+接口
        requestMappingSet = AnnotatedElementUtils.findAllMergedAnnotations(repeatMethod, RequestMapping.class);
        i = 0;
        for (RequestMapping requestMapping : requestMappingSet) {
            Assert.assertArrayEquals(value[i++], requestMapping.value());
        }
    }

}
```

