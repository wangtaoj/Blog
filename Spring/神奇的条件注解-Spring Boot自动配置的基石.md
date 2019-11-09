### 声明

> 源码基于Spring Boot 2.0.4 、Spring 5.0.8

### 1. 简介

条件注解是从Spring 4.0版本才开始出现的，作用就是当一个bean需要满足定义的条件后才会被Spring注册到容器中。而Spring Boot正是基于众多的条件注解来完成自动配置，而不覆盖用户自己定义的bean。最基本的条件注解是`@Conditional`以及各种衍生注解，诸如`@ConditionalOnClass`、`@ConditionalOnBean`、`@CondtionalOnProperty`等。条件注解需要搭配`Condition`接口一起使用，`Condition`接口中的`matches`方法定义了真正的判断逻辑。

### 2. 使用介绍

#### 2.1 @Conditonal

`@Conditional`注解是最基本的条件注解，可以自定义某种条件。其他的条件注解都是基于`@Conditional`注解而实现的。

```java
/**
 * 定义
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

	/**
	 * 实现了Condition接口的类，判断逻辑就是定义在这个接口中
	 */
	Class<? extends Condition>[] value();

}

/**
 * Condition接口定义
 */
@FunctionalInterface
public interface Condition {
    
    /**
     * 判断某个条件是否匹配
     */
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

因此在使用`@Conditional`注解时，先要定义自己想要的条件。接下来看看Spring Boot中内置的几个条件注解。

#### 2.2 内置注解定义

主要就看看几个常用的注解

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
	Class<?>[] value() default {};
	String[] name() default {};
}

@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnBean {
	Class<?>[] value() default {};
	String[] type() default {};
	Class<? extends Annotation>[] annotation() default {};
	String[] name() default {};
	SearchStrategy search() default SearchStrategy.ALL;
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
@Documented
@Conditional(OnPropertyCondition.class)
public @interface ConditionalOnProperty {
	String[] value() default {};
	String prefix() default "";
	String[] name() default {};
	String havingValue() default "";
	boolean matchIfMissing() default false;
}

```

观察后可以看到这些注解的定义上都存在`@Conditional`注解，其实真正起作用的还是`@Conditional`注解，只不过这些内置注解帮用户封装好了条件，方便使用。

#### 2.3 @ConditionalOnClass

这个注解看名字就知道作用了，当某个类在类路径里条件就满足。

```java
/**
 *  注意value和name属性声明的类最后会被合并，所有的类都在类路径上条件才会匹配
 *  如果value和name都没有指定，条件总是匹配
 */
public @interface ConditionalOnClass {
    /**
     * 类型
     */
	Class<?>[] value() default {};
    /**
     * 类的完全限定名
     */
	String[] name() default {};
}
```

#### 2.4 @ConditionalOnBean

这个注解的作用就是所声明的bean必须存在在容器里才通过，可以根据类型，也可以根据bean的名字。

```java
/**
 * 使用的时候需要注意的一点是如果类型，名字都没有的话。会做一个推断
 * 如果是在类上， 不做推断
 * 如果是在@Bean 方法上，类型推断为此方法的返回值
 *
 * 如果在类上使用这个注解，value(type)、name、annotation都没有指定将会抛出异常
 * 而在@Bean 方法上使用，不会抛出异常，因为会做一个类型推断
 */
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnBean {
    /**
     * 容器中有没有声明类型的bean存在
     */
	Class<?>[] value() default {};
    /**
     * 同上，只不过是用完全限定名来替代类型
     */
	String[] type() default {};
    /**
     * 寻找容器中有没有对应bean所代表的类被指定的注解类注解过
     */
	Class<? extends Annotation>[] annotation() default {};
    /**
     * bean的名字，容器中有没有这个名字的bean存在。
     */
	String[] name() default {};
    /**
     * 搜索范围
     * ALL: 所有上下文
     * CURRENT： 当前上下文
     * ANCESTORS：所有祖先上下文，但是不包括当前上下文
     */
	SearchStrategy search() default SearchStrategy.ALL;
}
```

#### 2.5 @CondtionalOnProperty

这个条件是用来判断配置文件中的定义的属性值与期待值是否一致的。

```java
/**
 * 这个有点奇怪的是属性名可以存在多个，但是期望值只有一个
 * 因此不能判断多个属性名，而每一个属性名对应的属性值不一样的情况。
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
@Documented
@Conditional(OnPropertyCondition.class)
public @interface ConditionalOnProperty {
    /**
     * 属性名字
     */
	String[] value() default {};
    /**
     * 如果不为空，会在属性名字上加前缀
     */
	String prefix() default "";
    /**
     * 同value, 属性名字
     */
	String[] name() default {};
    /**
     * 期望值，如果没有指定，默认为空。那么判断时只要属性值不是false都会通过
     */
	String havingValue() default "";
    /**
     * 如果指定的属性不存在，条件该通过还是不通过
     * 默认不通过
     */
	boolean matchIfMissing() default false;
}
```

### 3. 源码分析

这里以`@ConditionalOnClass`为例来分析条件注解的工作原理。因为我对这个比较感兴趣。

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
	Class<?>[] value() default {};
	String[] name() default {};
}
```

从定义我们知道匹配条件实在这个`OnClassCondition`类实现的。

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
class OnClassCondition extends SpringBootCondition
		implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {
    
}

public abstract class SpringBootCondition implements Condition {
    @Override
	public final boolean matches(ConditionContext context,
			AnnotatedTypeMetadata metadata) {
        // 方法体略
    }
}
```

从类的定义可以看到`ConditionalOnClass`继承了`SpringBootCondition`，从而间接实现了`Condition`接口。

接下来具体看代码分析

```java

/**
 * SpringBootCondition类
 * SpringBootCondition实现了通用的匹配框架，具体的匹配行为交给子类自己去做
 * 子类只需要实现getMatchOutcome方法即可
 */
@Override
public final boolean matches(ConditionContext context,
                             AnnotatedTypeMetadata metadata) {
    String classOrMethodName = getClassOrMethodName(metadata);
    try {
        // 得到匹配结果，由子类实现。因此重点看这个方法即可
        ConditionOutcome outcome = getMatchOutcome(context, metadata);
        logOutcome(classOrMethodName, outcome);
        recordEvaluation(context, classOrMethodName, outcome);
        // 返回结果
        return outcome.isMatch();
    }
    catch (NoClassDefFoundError ex) {
        throw new IllegalStateException(
            "Could not evaluate condition on " + classOrMethodName + " due to "
            + ex.getMessage() + " not "
            + "found. Make sure your own configuration does not rely on "
            + "that class. This can also happen if you are "
            + "@ComponentScanning a springframework package (e.g. if you "
            + "put a @ComponentScan in the default package by mistake)",
            ex);
    }
    catch (RuntimeException ex) {
        throw new IllegalStateException(
            "Error processing condition on " + getName(metadata), ex);
    }
}

public abstract ConditionOutcome getMatchOutcome(ConditionContext context,
			AnnotatedTypeMetadata metadata);

```

```java
/**
 * OnClassCondition
 */
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
                                        AnnotatedTypeMetadata metadata) {
    ClassLoader classLoader = context.getClassLoader();
    // 消息
    ConditionMessage matchMessage = ConditionMessage.empty();
    // 获取@ConditionalOnClass注解 value以及name属性声明的所有类
    // 只有这些类全部在类路径上条件才通过
    List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
    if (onClasses != null) {
        // 获取所有不在类路径上的类名
        List<String> missing = getMatches(onClasses, MatchType.MISSING, classLoader);
        // 只要缺失集合不为空，说明存在指定类不在类路径上，匹配失败
        if (!missing.isEmpty()) {
            return ConditionOutcome
                .noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
                         .didNotFind("required class", "required classes")
                         .items(Style.QUOTE, missing));
        }
        /*
         * 到这里算是指定的类都在类路径上了，本来应该直接返回了
         * 但是Spring Boot把@ConditionalOnMissingClass注解的逻辑也写在这个类
         * 可以去看@ConditionalOnMissingClass注解的定义
         */
        matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
            .found("required class", "required classes").items(Style.QUOTE,
                                                               getMatches(onClasses, MatchType.PRESENT, classLoader));
    }
    // 获取@ConditionalOnMissingClass注解 value以及name属性声明的所有类
    // 下面逻辑则刚好与上面相反
    List<String> onMissingClasses = getCandidates(metadata,
                                                  ConditionalOnMissingClass.class);
    if (onMissingClasses != null) {
        // 存在集合
        List<String> present = getMatches(onMissingClasses, MatchType.PRESENT,
        // 存在集合不为空， 不匹配                                  classLoader);
        if (!present.isEmpty()) {
            return ConditionOutcome.noMatch(
                ConditionMessage.forCondition(ConditionalOnMissingClass.class)
                .found("unwanted class", "unwanted classes")
                .items(Style.QUOTE, present));
        }
        matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
            .didNotFind("unwanted class", "unwanted classes")
                                          .items(Style.QUOTE,
                                                                    	getMatches(onMissingClasses, MatchType.MISSING, classLoader));
    }
    // 匹配成功                                      
    return ConditionOutcome.match(matchMessage);
}

```

```java
/**
 * 获取@ConditionalOnClass 或者 @ConditionalOnMissingClass
 * value以及name属性指定的类名
 */
private List<String> getCandidates(AnnotatedTypeMetadata metadata,
			Class<?> annotationType) {
    /*
     * 属性集合, 第二个参数为true意思是说将class -> 完全限定名
     * 这个方法的厉害之处是如果指定的Class不存在时，也能通过字节码获取到类的完全限定名
     * 如果不转换的话，指定的Class不存在时，获取到属性值将是一个异常对象(ClassNotFoundException)
     * 
     * AnnotatedTypeMetadata还有一个getAnnotationAttributes方法，取到的是单个值
     * 而getAllAnnotationAttributes方法取的是多个值
     * 
     * 我们知道@ConditionalOnClass注解并没有被元注解@Repeatable标注
     * 因此@ConditionalOnClass是不能被重复在类上使用的，那么value属性值应该是单个的才对啊，
     * 返回值就是一个String[]才对，为什么用getAllAnnotationAttributes方法呢?
     * 这就是getAllAnnotationAttributes神奇之处，不仅会获取到直接使用@ConditionalOnClass注解
     * 指定的值，如果有别的注解被@ConditionalOnClass注解了也能获取到
     *
     * 举例如下
     * @ConditionalOnClass(value = {A.class})
     * @AnotherConditionalOnClas
     * public class AppConfig{ }
     * 
     * @ConditionalOnClass(value = {B.class})
     * public @interface AnotherConditionalOnClas {}
     * 获取到的结果:
     * {value: [new String[]{A.class}, new String[]{B.class}]}
     */
    MultiValueMap<String, Object> attributes = metadata
        .getAllAnnotationAttributes(annotationType.getName(), true);
    if (attributes == null) {
        return Collections.emptyList();
    }
    List<String> candidates = new ArrayList<>();
    // 添加value属性指定的类
    addAll(candidates, attributes.get("value"));
    // 添加name属性指定的类
    addAll(candidates, attributes.get("name"));
    return candidates;
}

private void addAll(List<String> list, List<Object> itemsToAdd) {
    if (itemsToAdd != null) {
        for (Object item : itemsToAdd) {
            Collections.addAll(list, (String[]) item);
        }
    }
}
```

```java
/**
 * 根据MatchType参数获取在类路径上的类名还是不在类路径上的类名
 * 代码非常易懂，不用过多解析。
 */
private List<String> getMatches(Collection<String> candidates, MatchType matchType,
			ClassLoader classLoader) {
    List<String> matches = new ArrayList<>(candidates.size());
    for (String candidate : candidates) {
        if (matchType.matches(candidate, classLoader)) {
            matches.add(candidate);
        }
    }
    return matches;
}

private enum MatchType {
    PRESENT {

        @Override
        public boolean matches(String className, ClassLoader classLoader) {
            return isPresent(className, classLoader);
        }

    },

    MISSING {

        @Override
        public boolean matches(String className, ClassLoader classLoader) {
            return !isPresent(className, classLoader);
        }

    };

    private static boolean isPresent(String className, ClassLoader classLoader) {
        if (classLoader == null) {
            classLoader = ClassUtils.getDefaultClassLoader();
        }
        try {
            forName(className, classLoader);
            return true;
        }
        catch (Throwable ex) {
            return false;
        }
    }

    private static Class<?> forName(String className, ClassLoader classLoader)
        throws ClassNotFoundException {
        if (classLoader != null) {
            return classLoader.loadClass(className);
        }
        return Class.forName(className);
    }

    public abstract boolean matches(String className, ClassLoader classLoader);

}
```

至此`@ConditionalOnClass`注解的工作原理便分析完毕了，但是这些条件注解是在哪里生效的呢?

答案当然是去解析`@Configuration`注解配置的类去找了。解析入口为`ConfigurationClassPostProcessor`

具体的解析工作委托给了`ConfigurationClassParser`类，而注册bean的工作委托给了

`ConfigurationClassBeanDefinitionReader`类了。

```java
/*
 * ConfigurationClassParser类
 * 解析配置类
 */
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    // 判断条件注解, 只针对ConfigurationPhase.PARSE_CONFIGURATION
    // 像@ConditionalOnBean其实只在ConfigurationPhase.REGISTER_BEAN阶段生效
    if (this.conditionEvaluator.shouldSkip(
        configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }
    // 以下代码略
}

/** 
 * ConfigurationClassBeanDefinitionReader类
 * 将解析完后的配置类注册到容器中
 */
private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, 
    TrackedConditionEvaluator trackedConditionEvaluator) {

    // 判断条件注解，会针对ConfigurationPhase.REGISTER_BEAN
    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry
            .containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(
            configClass.getMetadata().getClassName());
        return;
    }

    // 如果本身没有被注册，注册自己
    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    // 注册@bean 方法定义的bean
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }
	// 注册@ImportResource注解引入的XML配置定义的bean
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

如果对注解配置的工作原理感兴趣，可参考此文章[Spring 注解配置原理](https://www.cnblogs.com/wt20/p/11823783.html) 。

### 四. 总结

本文从源码层面详细地介绍了Spring 4 引入的条件注解，并且介绍了Spring Boot中几个内置的条件注解该如何使用。掌握好之后相信对其他内置条件注解也是手到擒来，对于理解Spring Boot自动配置原理也有很大的帮助。
