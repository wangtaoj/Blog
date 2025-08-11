### 通知
- @Before：前置通知
- @After：后置通知，无论是否发生异常都会执行
- @AfterReturning：后置通知，正常返回时执行
- @AfterThrowing：后置通知，异常返回时执行
- @Aroud：环绕通知
各种通知使用例子
```java
// 定义切点  
@Pointcut("execution(* com.wangtao.springInAction.service.IDBService.*(..))")  
public void pointCut() {  

}  

/**
 * 定义通知, 并引用切点  
 * 在方法执行前执行  
 */  
@Before("pointCut()")  
public void doBefore() {  
    System.out.println("开始事务");  
}  

@After("pointCut()")  
public void doAfter() {  
    System.out.println("在方法之后执行, 无论方法是否抛出异常");  
}  

/**  
 * 在方法正常返回后执行  
 * returning: 指定目标方法返回结果的参数名
 */  
@AfterReturning(value = "pointCut()", returning = "result")  
public void doAfterReturning(Object result) {  
    System.out.println(result);  
    System.out.println("提交事务");  
}  

/**
 * 在抛出异常时执行
 * throwing: 目标方法异常参数名字, 方便将异常对象绑定到通知方法参数中
 * 这里无须再将异常抛出去, 内部自己会将异常抛出去, 也就是说异常不会被吞掉
 */  
@AfterThrowing(value = "pointCut()", throwing = "e")  
public void doAfterThrowing(Exception e) {  
    System.out.println(e.getClass());  
    System.out.println("出现异常, 回滚");  
}  

/**
 * 环绕通知
 */
@Around(value = "pointCut()")  
public Object doRound(ProceedingJoinPoint jp) {  
    try {
        // 执行目标方法, 必须将结果返回回去, 不然调用目标方法会拿不到执行结果  
        return jp.proceed();  
    } catch (Throwable e) {  
        throw new RuntimeException(e);  
    }  
}
```
### 绑定参数

上面的例子中可以看到绑定方法的返回值或者异常，而绑定参数则需要借助`args`切点指示器，下面以前置通知为例。

```java
@Slf4j
@Aspect
@Component
public class AspectDemo {

    /**
     * 拦截被MyTransactional注解的方法，并且参数有两个，类型是String和int
     * argNames可以省略，作用是为了标记方法上参数名称，因为Java在没有开启-paramter参数编译时
     * 反射是获取不到参数名字的, 不过Spring通过ASM读取字节码文件依然可以获取到参数名称
     * 
     * 注: args里的参数名称是下面方法(advice), 不是拦截方法的参数, 根据参数名称来获取参数类型
     */
    @Before(value = "@annotation(com.wangtao.springboottest.aspect.MyTransactional) && args(param1, param2)", argNames = "param1,param2")
    public void bindArgs(String param1, int param2) {
        log.info("========before {}, {}=======", param1, param2);
    }
}
```

### 切点表达式

#### args

目标方法参数个数和类型需要一致才会被拦截，多个参数使用逗号分割

```java
/**
 * 用法一
 * 直接指定参数类型
 */
@Before(value = "args(java.lang.String, int)")
public void doBefore() {
    
}

/**
 * 用法二
 * 根据通知方法的参数名称推断参数类型，并且还可以将拦截方法的调用参数绑定到通知方法参数中
 * 注: 还是根据参数类型和个数来拦截，不是参数名称
 */
@Before(value = "args(param1, param2)", argNames = "param1,param2")
public void bindArgs(String param1, int param2) {
    log.info("========before {}, {}=======", param1, param2);
}
```

#### @annotation
方法存在指定的注解，将会被拦截
使用例子：
创建一个自定义注解

```java
package com.wangtao.aop.annotation;

@Documented  
@Retention(RetentionPolicy.RUNTIME)  
@Target({ElementType.METHOD, ElementType.TYPE})  
public @interface MyTransactional {  

}
```

创建一个切面
```java
package com.wangtao.aop.aspect;

@Aspect
@Component
public class AnnotationAspect {

    /**
     * 使用方式1
     * 通知方法中将注解作为参数，@annotation使用参数名称来推断具体的注解
     * 此方式可以直接获取到拦截方法的注解信息
     * 注: 此方式无法在@Pointcut切点注解使用, 因为没有注解参数信息，无法推断具体注解，运行时      * 会报错
     */
    @Around("@annotation(transactional)")
    public Object doRound(ProceedingJoinPoint jp, MyTransactional transactional) throws Throwable {
        Object result;
        System.out.println(transactional);
        try {
            result = jp.proceed();
            System.out.println("=====commit=====");
            return result;
        } catch (Throwable e) {
            System.out.println("=====rollback=====");
            throw e;
        }
    }
    
    /**
     * 使用方式2
     * @annotation指定注解完全限定名
     * 该方式无法在通知方法中直接获取到拦截方法的注解信息，需要先拿到拦截方法，再拿注解信息
     * 注: @annotation如果只写类名, 会自动加上切面所在的包名来进行推断
     */
    @Around("@annotation(com.wangtao.aop.aspect.MyTransactional)")
    public Object doRound(ProceedingJoinPoint jp) throws Throwable {
        Object result;
        MyTransactional transactional = getGoalMethod(jp).getAnnotation(MyTransactional.class);
        System.out.println(transactional);
        try {
            result = jp.proceed();
            System.out.println("=====commit=====");
            return result;
        } catch (Throwable e) {
            System.out.println("=====rollback=====");
            throw e;
        }
    }
    
    private Method getGoalMethod(ProceedingJoinPoint jp) {  
        MethodSignature signature = (MethodSignature)jp.getSignature();  
        return signature.getMethod();  
    }
}
```
#### @within

**声明方法的类(接口)**存在指定注解，将会被拦截，该指示器的目标是类(接口)上而不是方法。

注意是声明该方法的类，比如一个接口有一个默认方法，该接口存在指定的注解，如果一个类实现了这个接口并重写了这个默认方法，但是实现类上没有加上这个注解，这不会被拦截。另外一个实现类没有重写，调用这个默认方法会被拦截。因为重写后，声明该方法的类从接口变成了该实现类。

#### @target

调用该方法的**对象所属的类**存在指定注解，将会被拦截。

与`@within`不一样，强调的是当前实际调用对象所在的类，而不是方法声明的类。

注意：该指示器会把容器中不相关的bean也会生成代理类，虽然执行方法时实际不会拦截，这点实在无法理解Spring为什么这么干。

使用方式与`@annotation`一致。

### 技巧

#### 获取拦截方法
```java
public Method getGoalMethod(ProceedingJoinPoint jp) {  
    MethodSignature signature = (MethodSignature)jp.getSignature();  
    return signature.getMethod();  
}
```
拿到拦截方法之后，便可以拿到各种信息了，如注解，参数类型，参数个数，返回值类型等。