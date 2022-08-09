### 前言

Mybatis-Plus可以使用`LambdaQueryWrapper`构造`where`条件，如下所示

```java
LambdaQueryWrapper<Example> wrapper = Wrappers.lambdaQuery();
wrapper.eq(Example::getName, "user-1");
```

实则是借助了`lambda`表达式序列化原理来获取到列名的。

### 序列化与反序列化

`lambda`表达式序列化也是要求函数式接口实现`Serializable`接口的，`lambda`表达式序列化后真实存在的类为`java.lang.invoke.SerializedLambda`，一个`lambda`表达式有一个看不见的方法，方法名为`writeReplace`，而这个方法的返回值就是`SerializedLambda`。

准备函数式接口

```java
/**
 * 继承Serializable接口, 这样SFunction的实现类也就实现了Serializable接口
 */
@FunctionalInterface
public interface SFunction<T, R> extends Function<T, R>, Serializable {
}

public class User {

    private Integer id;

    private String name;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```



```java
package com.wangtao.lambda.serialize;

import org.junit.Assert;
import org.junit.Test;

import java.io.*;
import java.lang.invoke.SerializedLambda;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

/**
 * @author wangtao
 * Created at 2022/8/9 22:31
 */
public class LambdaSerializedTest {
    
	/**
	 * 程序结果:
	 * class com.wangtao.lambda.serialize.LambdaSerializedTest$$Lambda$4/708049632
	 * 反序列化回来的结果并不是java.lang.invoke.SerializedLambda, 是因为调用了SerializedLambda
	 * 中的readResolve方法还原成了真实的类型, 参照SerializedLambda源码说明
	 */
    @Test
    public void fun1() {
        SFunction<User, String> sFunction = User::getName;
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             ObjectOutputStream oos = new ObjectOutputStream(baos)) {
            oos.writeObject(sFunction);
            oos.flush();
            // 反序列化
            byte[] bytes = baos.toByteArray();
            try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bytes))) {
                Object obj = ois.readObject();
                System.out.println(obj.getClass());
            }
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

```

### 获取SerializedLambda

第一种方式，使用反射调用`writeReplace`方法

```java
@Test
public void fun2() {
    SFunction<User, String> sFunction = User::getName;
    try {
        System.out.println(sFunction.getClass());
        Method method = sFunction.getClass().getDeclaredMethod("writeReplace");
        method.setAccessible(true);
        SerializedLambda serializedLambda = (SerializedLambda) method.invoke(sFunction);
        Assert.assertEquals("getName", serializedLambda.getImplMethodName());
        Assert.assertEquals(User.class.getName().replace(".", "/"), serializedLambda.getImplClass());
    } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
        e.printStackTrace();
    }
}
```

第二种方式，使用序列化反序列化获取，这里有点小技巧，从上面看，正常反序列化回来时是获取不到`SerializedLambda`的，但是我们可以复制一份`SerializedLambda`源代码，然后去掉它的`readResolve`方法就不会还原成真实的类了。

com.wangtao.lambda.serialize.SerializedLambda

```java
package com.wangtao.lambda.serialize;

import java.io.Serializable;
import java.lang.invoke.MethodHandleInfo;
import java.util.Objects;

/**
 * @author wangtao
 * Created at 2022/8/9 22:41
 */
public final class SerializedLambda implements Serializable {
    private static final long serialVersionUID = 8025925345765570181L;
    private final Class<?> capturingClass;
    private final String functionalInterfaceClass;
    private final String functionalInterfaceMethodName;
    private final String functionalInterfaceMethodSignature;
    private final String implClass;
    private final String implMethodName;
    private final String implMethodSignature;
    private final int implMethodKind;
    private final String instantiatedMethodType;
    private final Object[] capturedArgs;

    /**
     * Create a {@code SerializedLambda} from the low-level information present
     * at the lambda factory site.
     *
     * @param capturingClass The class in which the lambda expression appears
     * @param functionalInterfaceClass Name, in slash-delimited form, of static
     *                                 type of the returned lambda object
     * @param functionalInterfaceMethodName Name of the functional interface
     *                                      method for the present at the
     *                                      lambda factory site
     * @param functionalInterfaceMethodSignature Signature of the functional
     *                                           interface method present at
     *                                           the lambda factory site
     * @param implMethodKind Method handle kind for the implementation method
     * @param implClass Name, in slash-delimited form, for the class holding
     *                  the implementation method
     * @param implMethodName Name of the implementation method
     * @param implMethodSignature Signature of the implementation method
     * @param instantiatedMethodType The signature of the primary functional
     *                               interface method after type variables
     *                               are substituted with their instantiation
     *                               from the capture site
     * @param capturedArgs The dynamic arguments to the lambda factory site,
     *                     which represent variables captured by
     *                     the lambda
     */
    public SerializedLambda(Class<?> capturingClass,
                            String functionalInterfaceClass,
                            String functionalInterfaceMethodName,
                            String functionalInterfaceMethodSignature,
                            int implMethodKind,
                            String implClass,
                            String implMethodName,
                            String implMethodSignature,
                            String instantiatedMethodType,
                            Object[] capturedArgs) {
        this.capturingClass = capturingClass;
        this.functionalInterfaceClass = functionalInterfaceClass;
        this.functionalInterfaceMethodName = functionalInterfaceMethodName;
        this.functionalInterfaceMethodSignature = functionalInterfaceMethodSignature;
        this.implMethodKind = implMethodKind;
        this.implClass = implClass;
        this.implMethodName = implMethodName;
        this.implMethodSignature = implMethodSignature;
        this.instantiatedMethodType = instantiatedMethodType;
        this.capturedArgs = Objects.requireNonNull(capturedArgs).clone();
    }

    /**
     * Get the name of the class that captured this lambda.
     * @return the name of the class that captured this lambda
     */
    public String getCapturingClass() {
        return capturingClass.getName().replace('.', '/');
    }

    /**
     * Get the name of the invoked type to which this
     * lambda has been converted
     * @return the name of the functional interface class to which
     * this lambda has been converted
     */
    public String getFunctionalInterfaceClass() {
        return functionalInterfaceClass;
    }

    /**
     * Get the name of the primary method for the functional interface
     * to which this lambda has been converted.
     * @return the name of the primary methods of the functional interface
     */
    public String getFunctionalInterfaceMethodName() {
        return functionalInterfaceMethodName;
    }

    /**
     * Get the signature of the primary method for the functional
     * interface to which this lambda has been converted.
     * @return the signature of the primary method of the functional
     * interface
     */
    public String getFunctionalInterfaceMethodSignature() {
        return functionalInterfaceMethodSignature;
    }

    /**
     * Get the name of the class containing the implementation
     * method.
     * @return the name of the class containing the implementation
     * method
     */
    public String getImplClass() {
        return implClass;
    }

    /**
     * Get the name of the implementation method.
     * @return the name of the implementation method
     */
    public String getImplMethodName() {
        return implMethodName;
    }

    /**
     * Get the signature of the implementation method.
     * @return the signature of the implementation method
     */
    public String getImplMethodSignature() {
        return implMethodSignature;
    }

    /**
     * Get the method handle kind (see {@link MethodHandleInfo}) of
     * the implementation method.
     * @return the method handle kind of the implementation method
     */
    public int getImplMethodKind() {
        return implMethodKind;
    }

    /**
     * Get the signature of the primary functional interface method
     * after type variables are substituted with their instantiation
     * from the capture site.
     * @return the signature of the primary functional interface method
     * after type variable processing
     */
    public final String getInstantiatedMethodType() {
        return instantiatedMethodType;
    }

    /**
     * Get the count of dynamic arguments to the lambda capture site.
     * @return the count of dynamic arguments to the lambda capture site
     */
    public int getCapturedArgCount() {
        return capturedArgs.length;
    }

    /**
     * Get a dynamic argument to the lambda capture site.
     * @param i the argument to capture
     * @return a dynamic argument to the lambda capture site
     */
    public Object getCapturedArg(int i) {
        return capturedArgs[i];
    }

    @Override
    public String toString() {
        String implKind= MethodHandleInfo.referenceKindToString(implMethodKind);
        return String.format("SerializedLambda[%s=%s, %s=%s.%s:%s, " +
                        "%s=%s %s.%s:%s, %s=%s, %s=%d]",
                "capturingClass", capturingClass,
                "functionalInterfaceMethod", functionalInterfaceClass,
                functionalInterfaceMethodName,
                functionalInterfaceMethodSignature,
                "implementation",
                implKind,
                implClass, implMethodName, implMethodSignature,
                "instantiatedMethodType", instantiatedMethodType,
                "numCaptured", capturedArgs.length);
    }
}

```

**注:**

**此类与`java.lang.invoke.SerializedLambda`基本一模一样，唯一的区别是包名不一样，并且没有`readResolve`方法**

```java
@Test
public void fun3() {
    SFunction<User, String> sFunction = User::getName;
    try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
         ObjectOutputStream oos = new ObjectOutputStream(baos)) {
        oos.writeObject(sFunction);
        oos.flush();
        // 反序列化
        byte[] bytes = baos.toByteArray();
        try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bytes)) {

            /**
                 * resolveClass可以允许在反序列化时替换类型, 但是需要满足下面条件
                 * 1. 类名必须一致, 包名可以不一样
                 * 2. serialVersionUID必须和原来的类一样(这个是反序列化的必要条件, 不然会报错)
                 */
            @Override
            protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
                Class<?> clazz = super.resolveClass(desc);
                return clazz == SerializedLambda .class ? com.wangtao.lambda.serialize.SerializedLambda.class : clazz;
            }
        }) {
            com.wangtao.lambda.serialize.SerializedLambda serializedLambda = (com.wangtao.lambda.serialize.SerializedLambda ) ois.readObject();
            Assert.assertEquals("getName", serializedLambda.getImplMethodName());
            Assert.assertEquals(User.class.getName().replace(".", "/"), serializedLambda.getImplClass());
        }
    } catch (IOException | ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

当拿到`SerializedLambda`后，我们就能获取`lambda`表达式的很多元数据了，比如`getImplMethodName()`就能拿到`方法引用`的方法名称了，见上面程序。

### 总结

回到前言的例子

```java
LambdaQueryWrapper<Example> wrapper = Wrappers.lambdaQuery();
wrapper.eq(Example::getName, "user-1");
```

`wrapper.eq`的第一个参数就是一个lambda表达式，当传递一个方法引用时，便可以使用上述两种方式获取到`SerializedLambda`实例，调用`getImplMethodName`方法就能获取到方法名`getName`了，通过截取就能得到属性名(列名)`name`了，就可以构造`where`条件了。

