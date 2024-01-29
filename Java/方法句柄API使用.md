### 基本使用步骤

第一步，构造要调用方法的`MethodType`，由返回值类型+参数列表类型组成。

第二步，获取`Lookup`实例，一般使用`MethodHandles`类中提供的静态方法获取，最常用的`MethodHandles.lookup()`。

第三步，调用`Lookup`实例的findXXX方法获取到`MethodHandle`, 即方法句柄。主要有`findConstructor`、`findStatic`、`findVirtual`、`findSpecial`等这几个查找方法。

第四步，调用`MethodHandle`实例的`bindTo`方法来绑定一个对象，相当于该对象来调用这个查到的方法。注意`bindTo`会返回一个新的`MethodHandle`实例。

第五步，调用`MethodHandle`实例的`invokeXXX`方法来执行具体查到的方法。主要有`invoke`、`invokeExact`、`invokeWithArguments`。

### API使用

```java
public class MethodHandleTest {

    public static class GrandFather {

        public void say(String message) {
            System.out.println("GrandFather: " + message);
        }
    }

    public static class Father extends GrandFather {

        public void say(String message) {
            System.out.println("Father: " + message);
        }
    }

    public static class Son extends Father {

        public final void say(String message) {
            System.out.println("Son: " + message);
        }
    }

    /**
     * 支持方法多态调用, 因此像私有方法这种不能重写的方法是不支持的
     * 私有方法需要使用findSpecial方法
     */
    @Test
    public void testFindVirtual() throws Throwable{
        // 第一个参数为参数返回值类型, 其余参数为方法参数类型
        MethodType methodType = MethodType.methodType(void.class, String.class);
        MethodHandle tmp = MethodHandles.lookup()
                .findVirtual(GrandFather.class, "say", methodType);
        /*
         * 绑定对象, 相当于该对象调用这个方法
         * 如果不绑定, 则调用invoke系列方法时, 第一个参数是调用对象, 其余参数是方法参数
         * 注: 会返回一个新的MethodHandle
         */
        MethodHandle methodHandle = tmp.bindTo(new Son());
        // 打印Son: Hello
        methodHandle.invokeWithArguments("Hello");

        methodHandle = tmp.bindTo(new Father());
        // 打印Father: Hello
        methodHandle.invokeWithArguments("Hello");

        methodHandle = tmp.bindTo(new GrandFather());
        // 打印GrandFather: Hello
        methodHandle.invokeWithArguments("Hello");

        // 只支持绑定到Son这个类型的实例(包括子类), 因为使用的Son.class查找
        methodHandle = MethodHandles.lookup()
                .findVirtual(Son.class, "say", methodType)
                .bindTo(new Son());
        methodHandle.invokeWithArguments("Hello");
    }

    /**
     * 1. 这个方法不支持多态
     * 2. 关于方法查找逻辑:
     *    如果第一个参数与第四个参数一样, 则只从指定的这个类中查找方法, 查找不到抛异常
     *    如果不一样, 第四个参数必须是第一个参数的子类型, 则第一个参数是查找的上界,
     *    查找的起点类是第四个参数的父类, 查到方法则停止, 否则继续往父类找。
     * 3. 权限检查:
     *    MethodHandles.LookUp类中存在一个lookupClass属性
     *    在执行findSpecial方法时会进行一个检查, lookupClass必须与第四个参数是同一个类型
     *    从JDK9新增了一个另外, 如果第一个参数是接口, 则可以不一致。
     *    这样子即便直接使用MethodHandles.lookup()也能查找接口中的默认方法。
     *    具体逻辑参见findSpecial方法中的checkSpecialCaller方法
     *
     *    注: MethodHandles.lookup()返回的实例lookupClass属性为调用lookup()方法所在的类
     */
    @Test
    public void testFindSpecial() throws Throwable {
        MethodType methodType = MethodType.methodType(void.class, String.class);
        /*
         * JDK9才有privateLookupIn这个方法, JDK8无
         * 如果直接使用MethodHandles.lookup()获取Lookup实例, 则lookupClass=MethodHandleTest.class
         * 因为lookupClass属性值为执行MethodHandles.lookup()所在的类。
         * 那么lookupClass与第四个参数Son.class不一致, 执行findSpecial会报错
         * 因此使用MethodHandles.privateLookupIn修改lookupClass属性为Son.class
         *
         * 在JDK8中该如何呢, 可通过反射获取LookUp构造方法, 传入lookupClass即可
         * 或者不通过反射, 则只能在Son这个类中使用MethodHandles.lookup()了, 因为此时
         * lookupClass=Son.class, 与第四个参数Son.class一致, 则可以满足检查
         */
        MethodHandle methodHandle = MethodHandles.privateLookupIn(Son.class, MethodHandles.lookup())
                .findSpecial(GrandFather.class, "say", methodType, Son.class)
                .bindTo(new Son());
        /*
         * 尽管绑定的对象是Son的实例
         * 打印Father: Hello, 如果Father中没有此方法, 则打印GrandFather: Hello
         */
        methodHandle.invokeWithArguments("Hello");

        // 查到的方法为Son中的say方法
        methodHandle = MethodHandles.privateLookupIn(Son.class, MethodHandles.lookup())
                .findSpecial(Son.class, "say", methodType, Son.class)
                .bindTo(new Son());
        // 打印Son: Hello
        methodHandle.invokeWithArguments("Hello");
    }
}
```

### invoke系列方法的区别

```java
public class MethodHandleInvokeTest {

    public static class Caculator {

        public Integer sum(Integer num1, Integer num2) {
            return num1 + num2;
        }
    }

    /**
     * invokeExact: 参数和返回值需要精确匹配, 不会自动类型转换, 使用参数声明的静态类型
     * invoke: 参数和返回值会自动类型转换, 比如转型、数字类型提升、拆箱装箱(包装类型,原始类型)
     * invokeWithArguments: invoke方法的升级版本, 支持使用数组作为可变参数传递
     */
    @Test
    public void testInvoke() throws Throwable {
        MethodType methodType = MethodType.methodType(Integer.class, Integer.class, Integer.class);
        MethodHandle tmp = MethodHandles.lookup()
                .findVirtual(Caculator.class, "sum", methodType);
        /*
         * 绑定对象, 相当于该对象调用这个方法
         * 如果不绑定, 则调用invoke系列方法时, 第一个参数是调用对象, 其余参数是方法参数
         * 注: 会返回一个新的MethodHandle
         */
        MethodHandle methodHandle = tmp.bindTo(new Caculator());
        Integer one = 1;
        // ok
        methodHandle.invoke(1, 1);

        // error, 因为参数类型是int, 不是Integer
        assertException(WrongMethodTypeException.class, () -> methodHandle.invokeExact(1, 1));
        // error, 因为返回类型是void, 不是Integer
        assertException(WrongMethodTypeException.class, () -> methodHandle.invokeExact(one, one));
        // ok, 参数类型是Integer, 返回值也是Integer
        Integer result = (Integer)methodHandle.invokeExact(one, one);
        Assert.assertEquals(2, result.intValue());

        Object[] args = new Object[] {1, 1};
        // error, invoke和invokeExact不接受数组作为可变参数, 会认为args是一个参数
        assertException(WrongMethodTypeException.class, () -> methodHandle.invoke(args));
        // ok, invokeWithArguments支持数组作为可变参数, 会当做两个参数看待
        methodHandle.invokeWithArguments(args);
    }

    private <T extends Throwable> void assertException(Class<T> expectedType, Executable executable) {
        try {
            executable.execute();
        } catch (Throwable e) {
            if (expectedType.isInstance(e)) {
                return;
            }
        }
        Assert.fail();
    }

    @FunctionalInterface
    public interface Executable {
        void execute() throws Throwable;
    }
}
```

### 在动态代理中使用

在Mybatis的Mapper接口中是可以写默认方法的，这个就是借助方法句柄实现的。

```java
public class MethodHandleWithProxyTest {

    /**
     * 动态代理中借助MethodHandle执行接口中的默认方法
     * 如果直接调用method.invoke(proxy, args)则会无限递归
     * proxy.method() -> InvocationHandler.invoke() -> proxy.method()
     */
    @Test
    public void testExecuteDefaultAtProxy() {
        Mapper proxyInstance = (Mapper) Proxy.newProxyInstance(
                Mapper.class.getClassLoader(),
                new Class<?>[]{Mapper.class},
                new DefaultHandler()
        );
        String sql = "select * from user";
        proxyInstance.executeWithPage(sql, 0 , 10);
    }

    public interface Mapper {
        void execute(String sql);

        default void executeWithPage(String sql, int offset, int size) {
            execute(sql + " limit " + offset + ", " + size);
        }
    }

    public static class DefaultHandler implements InvocationHandler {

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            }
            if (method.isDefault()) {
                MethodType methodType = MethodType.methodType(method.getReturnType(), method.getParameterTypes());
                /*
                 * 1. 指定lookupClass为方法声明的类, 即接口Mapper.class
                 * 2. 查找接口中对应的默认方法, 限制findSpecial第一个参数和第四个参数都是Mapper.class
                 * 3. 这样lookupClass=findSpecial第四个参数, 可以通过权限检查
                 * 4. 绑定方法到代理对象中, 这样子默认方法如果有调用接口其他方法, 可以让其他方法拥有代理增强
                 *    多态体现
                 */
                Class<?> lookupClass = method.getDeclaringClass();
                MethodHandle methodHandle = MethodHandles.privateLookupIn(lookupClass, MethodHandles.lookup())
                        .findSpecial(lookupClass, method.getName(), methodType, lookupClass)
                        .bindTo(proxy);
                return methodHandle.invokeWithArguments(args);
            } else if("execute".equals(method.getName())) {
                System.out.println(args[0]);
                return null;
            } else {
                throw new UnsupportedOperationException(method.getName());
            }
        }
    }
}
```

