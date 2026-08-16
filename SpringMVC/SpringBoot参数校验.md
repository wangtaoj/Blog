## JSR303(Bean Validation)标准

### 简介

JSR303 是 Java 的 Bean Validation 1.0 规范，后续版本有 JSR349（1.1）、JSR380（2.0）。

接口: jakarta.validation-api

实现: hibernate-validator

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.2.5.Final</version>
</dependency>
```

### 核心校验接口

JSR303 中最重要的两个接口：

#### Validator

用于校验 **Java Bean 对象**。

```java
public interface Validator {

   /**
    * 校验对象内部属性
    * @param object 要校验的对象, 不能为null，否则抛出IllegalArgumentException
    * @param groups 可变参，指定校验分组，不传时使用默认组Default
    * @return 不会返回null, 若校验全部通过, 返回empty set
    */
    <T> Set<ConstraintViolation<T>> √(T object, Class<?>... groups);

    
    <T> Set<ConstraintViolation<T>> validateProperty(T object,
                                                     String propertyName,
                                                     Class<?>... groups);

    <T> Set<ConstraintViolation<T>> validateValue(Class<T> beanType,
                                                  String propertyName,
                                                  Object value,
                                                  Class<?>... groups);
    ExecutableValidator forExecutables();
}
```

获取方式：

```java
try (ValidatorFactory factory = Validation.buildDefaultValidatorFactory()) {
    Validator validator = factory.getValidator();
}
```

#### ExecutableValidator

用于校验 **方法参数、返回值、构造器参数、构造器返回值**，在Bean Validation 1.1 引入。

```java
public interface ExecutableValidator {

  /**
   * 校验方法参数
   * @param object 调用方法的对象
   * @param method 校验的方法
   * @param parameterValues 方法实参
   * @param groups 校验分组, 不传时使用默认组Default
   */
  <T> Set<ConstraintViolation<T>> validateParameters(T object,
                             Method method,
                             Object[] parameterValues,
                             Class<?>... groups);

  <T> Set<ConstraintViolation<T>> validateReturnValue(T object,
                            Method method,
                            Object returnValue,
                            Class<?>... groups);

  <T> Set<ConstraintViolation<T>> validateConstructorParameters(Constructor<? extends T> constructor,
                                  Object[] parameterValues,
                                  Class<?>... groups);

 
  <T> Set<ConstraintViolation<T>> validateConstructorReturnValue(Constructor<? extends T> constructor,
                                   T createdObject,
                                   Class<?>... groups);
}
```

获取方式:

```java
try (ValidatorFactory factory = Validation.buildDefaultValidatorFactory()) {
    Validator validator = factory.getValidator();
    ExecutableValidator executableValidator = validator.forExecutables();
}
```

#### @Valid的具体作用

`@Valid` 是 Bean Validation 中的 **标记注解**，本身不包含任何校验规则，它的作用是**触发级联校验**。

##### 用在字段上

```java
public class Order {
    @Valid
    private User user;
}
```

当使用`Validator.validate(order)`校验 `Order` 对象时，如果 `user` 字段标注了 `@Valid`，则**会继续校验 `User` 对象内部的约束注解**（如 `@NotBlank`、`@Size` 等）。如果没有 `@Valid`，则不会校验 `User` 内部字段。

##### 用在方法参数上

```java
public void createUser(@Valid User user, @NotBlank(message="操作人不能为空") String operator) {
    
}
```

当使用`ExecutableValidator.validateParameters`对该方法进行参数校验时，**`@Valid` 会触发 `User` 对象内部的约束校验**。如果不加，则不会对`User`对象内部的属性进行校验，即便`User`对象内部的属性有约束注解(如 `@NotBlank`、`@Size` 等)

##### 用在返回值上

```java
public @Valid User getUser() {
    
}

public @NotBlank(message="用户名称不能为空") String getUserName() {
    
}
```

当使用`ExecutableValidator.validateReturnValue`对该方法进行返回值校验时，**`@Valid` 会触发 返回的`User` 对象内部的约束校验**。

##### 注意事项

- 方法参数和返回值校验时，`@Valid` 不会触发 `null` 对象的内部校验；如果 `@Valid` 字段为 `null`，级联校验会被跳过。若字段本身不能为 `null`，需要额外加 `@NotNull`。如`void createUser(@NotNull @Valid User user)。`

### 校验异常

如果校验不通过时，抛出异常`ConstraintViolationException`，这是一个`RuntimeException`，接收一个`Set<ConstraintViolation>`作为参数

```java
public class ConstraintViolationException extends ValidationException {

    private final Set<ConstraintViolation<?>> constraintViolations;

    
    public ConstraintViolationException(String message,
                                        Set<? extends ConstraintViolation<?>> constraintViolations) {
        super( message );

        if ( constraintViolations == null ) {
            this.constraintViolations = null;
        }
        else {
            this.constraintViolations = new HashSet<>( constraintViolations );
        }
    }

   
    public ConstraintViolationException(Set<? extends ConstraintViolation<?>> constraintViolations) {
        this(
                constraintViolations != null ? toString( constraintViolations ) : null,
                constraintViolations
        );
    }

    
    public Set<ConstraintViolation<?>> getConstraintViolations() {
        return constraintViolations;
    }

    private static String toString(Set<? extends ConstraintViolation<?>> constraintViolations) {
        return constraintViolations.stream()
            .map( cv -> cv == null ? "null" : cv.getPropertyPath() + ": " + cv.getMessage() )
            .collect( Collectors.joining( ", " ) );
    }
}

```

### Validator使用示例

#### 简单示例

```java
public class Jsr303ValidatorTest {

    @Setter
    @Getter
    public static class User {

        @NotBlank(message = "用户名不能为空")
        @Size(min = 2, max = 20, message = "用户名长度必须在2-20之间")
        private String username;

        @Valid
        private Address address;

    }

    @Setter
    @Getter
    public static class Address {

        @NotBlank(message = "省份不能为空")
        private String province;

        @NotBlank(message = "城市不能为空")
        private String city;
    }

    @Test
    public void testValidateObject() {
        try (ValidatorFactory factory = Validation.buildDefaultValidatorFactory()) {
            Validator validator = factory.getValidator();

            User user = new User();
            user.setUsername("a");

            Address address = new Address();
            user.setAddress(address);
			
            // 不会返回null, 若校验全部通过, 返回empty set
            Set<ConstraintViolation<User>> violations = validator.validate(user);
            if (!violations.isEmpty()) {
                throw new ConstraintViolationException(violations);
            }
        }
    }
}
```

执行结果

javax.validation.ConstraintViolationException: username: 用户名长度必须在2-20之间, address.province: 省份不能为空, address.city: 城市不能为空

- username字段约束被校验。
- 由于 `address` 字段标了 `@Valid`，`Address` 内部的约束也被级联校验。

#### 分组校验

分组校验用于同一个实体在不同业务场景下使用时按照设置的组来控制是否生效。例如：创建时 `id` 必须为空，更新时 `id` 不能为空。

分组校验无法做到**同一个字段在不通场景下，都需要校验时，但是校验规则不一致**，比如管理员创建时密码长度只需大于4为，普通用户创建时密码长度必须大于8。

```java

public class Jsr303ValidatorTest {

    public interface UpdateGroup extends Default { }

    @Setter
    @Getter
    public static class User {

        @NotNull(message = "更新时ID不能为空", groups = {UpdateGroup.class})
        private Long id;

        @NotBlank(message = "用户名不能为空")
        @Size(min = 2, max = 20, message = "用户名长度必须在2-20之间")
        private String username;

        @Valid
        private Address address;

    }

    @Setter
    @Getter
    public static class Address {

        @NotBlank(message = "省份不能为空")
        private String province;

        @NotBlank(message = "城市不能为空")
        private String city;
    }


    @Test
    public void testValidateObjectUseGroup() {
        try (ValidatorFactory factory = Validation.buildDefaultValidatorFactory()) {
            Validator validator = factory.getValidator();

            User user = new User();
            user.setUsername("a");

            Address address = new Address();
            user.setAddress(address);
			// 传入不同的组可以观察到不同的校验结果
            Set<ConstraintViolation<User>> violations = validator.validate(user, UpdateGroup.class);
            if (!violations.isEmpty()) {
                throw new ConstraintViolationException(violations);
            }
        }
    }
}
```

说明:

`UpdateGroup`继承 `Default` 的作用：当校验 `CreateGroup` 或 `UpdateGroup` 时，会同时校验属于 `Default` 组的约束，避免校验时需要指定`CreateGroup`以及`Default`两个组。

##### 校验时不传分组参数或者传入Default.class

校验结果为: javax.validation.ConstraintViolationException: address.city: 城市不能为空, address.province: 省份不能为空, username: 用户名长度必须在2-20之间

可以看到id属性不会被校验，因为约束注解必须在`UpdateGroup`组才会生效

##### 校验时分组参数传入UpdateGroup.class:

校验结果为: javax.validation.ConstraintViolationException: username: 用户名长度必须在2-20之间, id: 更新时ID不能为空, address.city: 城市不能为空, address.province: 省份不能为空

可以看到id属性被校验了

### 方法参数与返回值校验示例

```java
public class Jsr303ValidatorTest {

    @Setter
    @Getter
    public static class User {

        @NotBlank(message = "用户名不能为空")
        @Size(min = 2, max = 20, message = "用户名长度必须在2-20之间")
        private String username;

    }

    /**
     * 返回值不能为 null, 并且需要级联校验内部约束
     * 参数 user 不能为 null，并且需要级联校验内部约束
     * 参数 operator不能为空
     */
    public @NotNull @Valid User createUser(@Valid @NotNull User user, @NotBlank String operator) {
        User returnUser = new User();
        returnUser.setUsername("b");
        return returnUser;
    }

    @Test
    public void testMethodParameterAndReturnValueValid() throws Exception {
        Method method = this.getClass().getMethod("createUser", User.class, String.class);
        try (ValidatorFactory factory = Validation.buildDefaultValidatorFactory()) {
            Validator validator = factory.getValidator();
            ExecutableValidator executableValidator = validator.forExecutables();

            User user = new User();
            user.setUsername("a");

            String operator = null;
            Object[] args = new Object[] {user, operator};
            Set<ConstraintViolation<Jsr303ValidatorTest>> violations = executableValidator.validateParameters(this, method, args);
            if (!violations.isEmpty()) {
                throw new ConstraintViolationException(violations);
            }
            Object returnValue = method.invoke(this, args);
            violations = executableValidator.validateReturnValue(this, method, returnValue);
            if (!violations.isEmpty()) {
                throw new ConstraintViolationException(violations);
            }
        }
    }
}
```

运行结果

```tex
javax.validation.ConstraintViolationException: createUser.operator: 不能为空, createUser.user.username: 用户名长度必须在2-20之间
```

调整参数，使其满足约束，观察返回值校验

```java
@Test
public void testMethodParameterAndReturnValueValid() throws Exception {
    Method method = this.getClass().getMethod("createUser", User.class, String.class);
    try (ValidatorFactory factory = Validation.buildDefaultValidatorFactory()) {
        Validator validator = factory.getValidator();
        ExecutableValidator executableValidator = validator.forExecutables();

        User user = new User();
        user.setUsername("nick");

        String operator = "zhangsan";
        Object[] args = new Object[] {user, operator};
        Set<ConstraintViolation<Jsr303ValidatorTest>> violations = executableValidator.validateParameters(this, method, args);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
        Object returnValue = method.invoke(this, args);
        violations = executableValidator.validateReturnValue(this, method, returnValue);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
    }
}
```

运行结果

```tex
javax.validation.ConstraintViolationException: createUser.<return value>.username: 用户名长度必须在2-20之间
```

### 自定义校验注解

当内置注解不能满足业务时，可以自定义校验器，例如手机号校验

```java
@Documented
@Constraint(validatedBy = { PhoneValidator.class })
@Target({FIELD, METHOD, PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
public @interface Phone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<Phone, String> {

    private static final Pattern PATTERN = Pattern.compile("^1[3-9]\\d{9}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value == null || PATTERN.matcher(value).matches();
    }
}
```

自定义注解中，使用@Constraint(validatedBy = { PhoneValidator.class })指定校验规则就行。

### 额外注意事项

#### Validator 是线程安全的

`Validator` 实例可以重复使用，建议在应用启动时创建一次，后续复用。`ValidatorFactory` 也可以缓存。

#### 大多数约束对 `null` 视为有效

例如 `@Size`、`@Min`、`@Email` 等约束，如果值为 `null`，默认会被视为合法。如需禁止 `null`，请额外添加 `@NotNull`。

#### 分组与默认组的关系

- 未指定 `groups` 的约束默认属于 `Default` 组。
- 执行 `validator.validate(obj)` 时只校验 `Default` 组。
- 执行 `validator.validate(obj, MyGroup.class)` 时只校验 `MyGroup` 组，**不会自动校验 `Default` 组**，除非 `MyGroup` 继承了 `Default`。

## SpringBoot参数校验

### 简介

Spring提供了自己的`org.springframework.validation.Validator`校验接口，但是会基于JSR303标准，Spring自己的`Validator`默认实现是就是基于JSR303的`javax.validation.Validator`。

### 底层 Validator 接口及常用实现

#### org.springframework.validation.Validator

```java
public interface Validator {
    boolean supports(Class<?> clazz);
    void validate(Object target, Errors errors);
}
```

#### SmartValidator

```java
public interface SmartValidator extends Validator {
    void validate(Object target, Errors errors, Object... validationHints);
}
```

`SmartValidator` 扩展了`org.springframework.validation.Validator`，增加了分组 hints。

#### 常用实现

Spring 通过桥接类把 Bean Validation 的 `Validator` 适配成 Spring 的 `Validator`：

| 实现类                      | 说明                                                         |
| :-------------------------- | :----------------------------------------------------------- |
| `LocalValidatorFactoryBean` | 同时实现了 `javax.validation.Validator`、`javax.validation.ValidatorFactory`、`org.springframework.validation.Validator`、`SmartValidator` |
| `SpringValidatorAdapter`    | 把 `javax.validation.Validator` 适配成 Spring `Validator`<br/>同时实现了`SmartValidator`、`javax.validation.Validator` |
| `Hibernate Validator`       | 底层真正执行约束校验的引擎                                   |

### 自动配置

Spring Boot 通过 `ValidationAutoConfiguration` 自动注册 `LocalValidatorFactoryBean`，并且会给这个bean设置@Primary。

另外在SpringMVC的自动配置中，`WebMvcAutoConfiguration.EnableWebMvcConfiguration.mvcValidator`也会配置一个名为`mvcValidator`的`org.springframework.validation.Validator`实例，具体实现是`ValidatorAdapter`，内部持有`LocalValidatorFactoryBean`

注: 

* `ValidatorAdapter`没有实现`javax.validation.Validator`接口。
* `RequestMappingHandlerAdapter`使用的是`mvcValidator`这个bean。

WebMvcAutoConfiguration.EnableWebMvcConfiguration.java

```java
@Bean
@Override
public Validator mvcValidator() {
    if (!ClassUtils.isPresent("javax.validation.Validator", getClass().getClassLoader())) {
        return super.mvcValidator();
    }
    /*
     * 这里会从容器中寻找javax.validation.Validator bean
     * 也就会找到ValidationAutoConfiguration自动注册的LocalValidatorFactoryBean了
     */
    
    return ValidatorAdapter.get(getApplicationContext(), getValidator());
}
```

### 两种机制

#### SpringMVC Controller 方法参数校验

##### 发生时机

**参数解析、绑定** 阶段。

##### 使用场景

- `@RequestBody` 参数：`@Validator @RequestBody User user`
- `@ModelAttribute` 参数：`@Validator @ModelAttribute User user`，对于复杂对象(非简单类型)，`@ModelAttribute`可以省略。

##### 底层执行原理

当使用 `@Valid` / `@Validated` 标注在 `@RequestBody`、`@ModelAttribute` 参数上时。`RequestMappingHandlerAdapter`执行Controller方法时，会先挨个收集方法的每一个参数进行绑定，绑定前会先进行校验，使用的正是`mvcValidator`这个bean，底层就是`javax.validation.Validator.validate()`方法，对单个对象进行校验。

##### 对于@RequestBody

由 `RequestResponseBodyMethodProcessor` 解析参数。它内部会调用 `AbstractMessageConverterMethodArgumentResolver#validateIfApplicable`。

RequestResponseBodyMethodProcessor.java

```java
/**
 * 解析参数
 */
@Override
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

    parameter = parameter.nestedIfOptional();
    Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
    String name = Conventions.getVariableNameForParameter(parameter);

    if (binderFactory != null) {
        WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
        if (arg != null) {
            // 校验参数
            validateIfApplicable(binder, parameter);
            if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
            }
        }
        if (mavContainer != null) {
            mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
        }
    }

    return adaptArgumentIfNecessary(arg, parameter);
}
```

AbstractMessageConverterMethodArgumentResolver.java

```java
protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
    Annotation[] annotations = parameter.getParameterAnnotations();
    for (Annotation ann : annotations) {
        /*
         * 它会检查参数上是否有 @Valid 或 @Validated，如果有，就调用 WebDataBinder#validate。
         * DataBinder#validate 最终调用 Spring Validator，也就是 LocalValidatorFactoryBean 
         * 再委托给 Hibernate Validator 的 ValidatorImpl#validate。
         */
        Object[] validationHints = ValidationAnnotationUtils.determineValidationHints(ann);
        if (validationHints != null) {
            binder.validate(validationHints);
            break;
        }
    }
}
```

DataBinder.java

```java
public void validate(Object... validationHints) {
    Object target = getTarget();
    Assert.state(target != null, "No target to validate");
    BindingResult bindingResult = getBindingResult();
    // Call each validator with the same binding result
    for (Validator validator : getValidators()) {
        if (!ObjectUtils.isEmpty(validationHints) && validator instanceof SmartValidator) {
            ((SmartValidator) validator).validate(target, bindingResult, validationHints);
        }
        else if (validator != null) {
            validator.validate(target, bindingResult);
        }
    }
}
```

##### 对于@ModelAttribute

有`ModelAttributeMethodProcessor`解析，内部方法`resolveArgument`也会调用`validateIfApplicable`

ModelAttributeMethodProcessor.java

```java
protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
    for (Annotation ann : parameter.getParameterAnnotations()) {
        Object[] validationHints = ValidationAnnotationUtils.determineValidationHints(ann);
        if (validationHints != null) {
            binder.validate(validationHints);
            break;
        }
    }
}
```

##### 校验不通过时抛出的异常

如果校验不通过则抛出`BindException`或者`MethodArgumentNotValidException`(继承自`BindException`)。

##### 校验的前提

* 方法参数必须使用`@Valid` 或者 `@Validated`标注
* 必须是`@RequestBody`、`@ModelAttribute`参数上

##### 局限性

这种机制只会在参数解析器主动调用 `validateIfApplicable` 时触发，通常只有 `@RequestBody`、`@ModelAttribute` 这类对象绑定参数才会触发。因为底层实现是基于`javax.validation.Validator.validate()`，此方法本来就是针对对象内部属性校验设计的，如果是基本类型，是没有作用的，因此就没有必要去校验了。

```java
@GetMapping("/hello")
public String hello(@RequestParam @NotBlank String name) {
    return "hello " + name;
}
```

这里 `name` 上有 `@NotBlank`，但不会生效。

原因是：

`@RequestParam` 由 `RequestParamMethodArgumentResolver` 解析，它不会调用 `DataBinder#validate`，简单类型即便是调用也没有效果

#### 方法级 AOP 代理校验

该机制就比较通用了，SpringBoot应用中都能使用，不一定非要在Controller方法中。而且不仅仅只是校验方法参数，方法返回值也会校验。

##### 发生时机

这种校验发生在 **Spring Bean 方法调用前后**，由 AOP 代理完成。

##### 使用方式

只需在类上标注`@Validated`即可，然后方法参数和返回值可以是用JSR303中的约束注解进行校验

类上只能使用`@Validated`来开启，`@Valid`不行，但是`@Valid`可以放到方法参数或者返回值上，用于级联触发对象内部属性校验。

##### 底层执行原理

Spring 通过 `MethodValidationPostProcessor` 在 Bean 初始化后创建 AOP 代理。在目标方法执行前后通过JSR303中的`ExecutableValidator.validateParameters`、`ExecutableValidator.validateReturnValue`来对方法参数以及返回值进行校验。

ValidationAutoConfiguration.java

```java
@Bean
@ConditionalOnMissingBean(search = SearchStrategy.CURRENT)
public static MethodValidationPostProcessor methodValidationPostProcessor(Environment environment,
        @Lazy Validator validator, ObjectProvider<MethodValidationExcludeFilter> excludeFilters) {
    FilteredMethodValidationPostProcessor processor = new FilteredMethodValidationPostProcessor(
            excludeFilters.orderedStream());
    boolean proxyTargetClass = environment.getProperty("spring.aop.proxy-target-class", Boolean.class, true);
    processor.setProxyTargetClass(proxyTargetClass);
    processor.setValidator(validator);
    return processor;
}
```

MethodValidationPostProcessor.java

```java
private Class<? extends Annotation> validatedAnnotationType = Validated.class;
@Override
public void afterPropertiesSet() {
    // 代理触发条件, 类上必须有@Validated注解(会从继承链上寻找该注解)
    Pointcut pointcut = new AnnotationMatchingPointcut(this.validatedAnnotationType, true);
    this.advisor = new DefaultPointcutAdvisor(pointcut, createMethodValidationAdvice(this.validator));
}

protected Advice createMethodValidationAdvice(@Nullable Validator validator) {
    return (validator != null ? new MethodValidationInterceptor(validator) : new MethodValidationInterceptor());
}
```

MethodValidationInterceptor.java

```java
@Override
@Nullable
public Object invoke(MethodInvocation invocation) throws Throwable {
    // Avoid Validator invocation on FactoryBean.getObjectType/isSingleton
    if (isFactoryBeanMetadataMethod(invocation.getMethod())) {
        return invocation.proceed();
    }

    Class<?>[] groups = determineValidationGroups(invocation);

    // Standard Bean Validation 1.1 API
    ExecutableValidator execVal = this.validator.forExecutables();
    Method methodToValidate = invocation.getMethod();
    Set<ConstraintViolation<Object>> result;

    Object target = invocation.getThis();
    Assert.state(target != null, "Target must not be null");
    // 参数校验
    try {
        result = execVal.validateParameters(target, methodToValidate, invocation.getArguments(), groups);
    }
    catch (IllegalArgumentException ex) {
        // Probably a generic type mismatch between interface and impl as reported in SPR-12237 / HV-1011
        // Let's try to find the bridged method on the implementation class...
        methodToValidate = BridgeMethodResolver.findBridgedMethod(
                ClassUtils.getMostSpecificMethod(invocation.getMethod(), target.getClass()));
        result = execVal.validateParameters(target, methodToValidate, invocation.getArguments(), groups);
    }
    if (!result.isEmpty()) {
        throw new ConstraintViolationException(result);
    }

    Object returnValue = invocation.proceed();
    // 返回值校验
    result = execVal.validateReturnValue(target, methodToValidate, returnValue, groups);
    if (!result.isEmpty()) {
        throw new ConstraintViolationException(result);
    }

    return returnValue;
}
```

##### 校验不通过时抛出的异常

校验不通过抛出`ConstraintViolationException`

### 两种机制的对比

| 对比项         | SpringMVC参数绑定校验                                        | 方法级 AOP 校验                                              |
| :------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 发生时机       | 参数解析、绑定阶段                                           | 方法调用前后，AOP 代理                                       |
| 触发条件       | 参数上标注 `@Valid` 或者`@Validated`                         | 类上标注 `@Validated`                                        |
| 支持简单参数   | 不支持                                                       | 支持                                                         |
| 支持返回值校验 | 不支持                                                       | 支持                                                         |
| 异常类型       | `MethodArgumentNotValidException`或者`BindException`         | `ConstraintViolationException`                               |
| 适用层         | Controller                                                   | Controller、Service 等任意 Spring Bean                       |
| 校验机制       | `Spring Validator` -> 委托给JSR303 `Validator.validate`(校验当个对象) | JSR303 `ExecutableValidator`的`validateParameters`以及`validateReturnValue` |

### 使用示例

增加依赖

```pom
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

#### Spring MVC参数绑定校验示例

```java
public class User {

    @NotBlank(message = "姓名不能为空")
    private String name;

    @Min(value = 1, message = "年龄必须大于 0")
    private int age;

    @Valid
    @NotNull(message = "地址不能为空")
    private Address address;
}

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public String create(@Valid @RequestBody User user) {
        return "创建成功";
    }
}
```

#### 方法级异常示例

```java
@Service
@Validated
public class UserService {

    /**
     * 简单参数
     */
    public @NotBlank String getUserName(@NotBlank String userId) {
        return "用户-" + userId;
    }

    /**
     * 复杂对象
     * user对象本身不能为空
     * @Valid 触发user对象内部的属性校验
     */
    public User create(@Valid @NotNull User user) {
        return user;
    }
}
```

#### 统一异常处理

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * SpringMVC参数绑定校验
     */
    @ExceptionHandler(BindException.class)
    public ResponseEntity<ServerResponse<?>> bindException(BindException e, HttpServletRequest request) {
        log.error("{} encounter a error.", request.getServletPath(), e);
        // 取第一个错误的信息
        String errMsg = Objects.requireNonNull(e.getFieldError()).getDefaultMessage();
        ServerResponse<?> serverResponse = ServerResponse.error(ResponseEnum.PARAM_ILLEGAL, errMsg);
        return new ResponseEntity<>(serverResponse, ResponseEnum.PARAM_ILLEGAL.getHttpStatus());
    }

    /**
     * 方法级AOP代理校验
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ServerResponse<?>> constraintViolationException(ConstraintViolationException e,
                                                                          HttpServletRequest request) {
        log.error("{} encounter a error.", request.getServletPath(), e);
        // 取第一个错误的信息
        String errMsg = e.getConstraintViolations().stream().map(ConstraintViolation::getMessage).findFirst().orElse(null);
        ServerResponse<?> serverResponse = ServerResponse.error(ResponseEnum.PARAM_ILLEGAL, errMsg);
        return new ResponseEntity<>(serverResponse, ResponseEnum.PARAM_ILLEGAL.getHttpStatus());
    }
}
```

