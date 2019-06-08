### RPC的主要目标

RPC即远程过程调用，主要是为了解决分布式系统服务之间的调用问题，在远程调用时就像调用本地服务一样，调用者不用感知远程调用的逻辑。接下来将会使用JDK动态代理以及Socket来实现一个简单的RPC服务。

### RPC简易实现

使用动态代理技术将远程调用逻辑封装起来，返回一个代理对象，这样就使得调用者完全不用理会远程调用的逻辑，就像调用本地方法一样。具体看代码

先看客户端代码

```java
/**
 * 该类主要用来封装请求参数
 */
public class RPCRequest implements Serializable {

    private static final long serialVersionUID = 6301788037460580636L;

    /**
     * 请求的服务类
     */
    private String serviceClass;

    /**
     * 请求的方法名
     */
    private String methodName;

    /**
     * 请求方法的参数类型
     */
    private Class<?>[] argTypes;

    /**
     * 请求参数
     */
    private Object[] args;

    public RPCRequest(String serviceName) {
        this.serviceClass = serviceName;
    }

    public String getServiceClass() {
        return serviceClass;
    }

    public void setServiceClass(String serviceClass) {
        this.serviceClass = serviceClass;
    }

    public String getMethodName() {
        return methodName;
    }

    public void setMethodName(String methodName) {
        this.methodName = methodName;
    }

    public void setArgs(Object[] args) {
        this.args = args;
    }

    public Object[] getArgs() {
        return args;
    }

    public Class<?>[] getArgTypes() {
        return argTypes;
    }

    public void setArgTypes(Class<?>[] argTypes) {
        this.argTypes = argTypes;
    }
}
```

```java
public class RPCInvocation implements InvocationHandler {

    private static final String REMOTE_HOST = "127.0.0.1";

    private static final int PORT = 8081;

    private RPCRequest request;

    /** 请求接口的具体实现类完全限定名 **/
    public RPCInvocation(String serviceName) {
        this.request = new RPCRequest(serviceName);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        InetAddress address = InetAddress.getByName(REMOTE_HOST);
        try (Socket socket = new Socket(address, PORT);
             ObjectOutputStream os = new ObjectOutputStream(socket.getOutputStream());
             ObjectInputStream is = new ObjectInputStream(socket.getInputStream())) {
            // 发送请求参数(服务类, 方法名字, 方法参数)
            request.setArgs(args);
            request.setArgTypes(method.getParameterTypes());
            request.setMethodName(method.getName());
            os.writeObject(request);
            // 读取结果并返回
            return is.readObject();
        }
    }
}
```

```java
public class RPCUtils {

    private RPCUtils() {

    }

    /**
     * @param clazz 请求的接口
     * @param implClassName 调用远程的具体实现类完全限定名
     * @return 返回指定接口的代理实现
     */
    @SuppressWarnings("unchecked")
    public static <T> T getRemoteInstance(Class<T> clazz, String implClassName) {
        return (T) Proxy.newProxyInstance(RPCUtils.class.getClassLoader(), new Class<?>[]			{clazz}, new RPCInvocation(implClassName));
    }
}
```

再看服务端代码

```java
public class RPCServer {
    
    public static void main(String[] args) throws Exception{
        new RPCServer().run();
    }

    public void run() throws Exception {
        try (ServerSocket serverSocket = new ServerSocket(8081)) {
            System.out.println("RPC服务已启动, 监听8081端口");
            for (; ; ) {
                try (Socket socket = serverSocket.accept();
                     ObjectOutputStream os = new ObjectOutputStream(
                         socket.getOutputStream());
                     ObjectInputStream is = new ObjectInputStream(
                         socket.getInputStream())) {
                    System.out.println("开始处理来自主机-" + socket.getInetAddress().
                                       getHostName() + "的请求!");
                    Object temp = is.readObject();
                    RPCRequest request = (RPCRequest) temp;
                    // 请求类
                    Class<?> serviceClass = Class.forName(request.getServiceClass());
                    Object serviceObj = serviceClass.getConstructor().newInstance();
                    // 方法参数以及参数类型
                    Class<?>[] argTypes = request.getArgTypes();
                    Object[] args = request.getArgs();
                    // 请求方法
                    Method method = serviceClass.getMethod(
                        request.getMethodName(), argTypes);
                    Object result = method.invoke(serviceObj, args);
                    os.writeObject(result);
                    System.out.println("主机-" + socket.getInetAddress().
                                       getHostName() + "的请求处理完毕!");

                } catch (Exception e) {
                    throw new RuntimeException("远程过程调用失败", e);
                }
            }
        }
    }
}

```

客户端如何调用？

假设系统A，系统B都依赖了Caculator接口，其中B系统有一个具体实现类CaculatorImpl实现了此接口，现在A系统想要通过远程调用B系统中CaculatorImpl来使用计算器功能。

```JAVA
/** A、B系统共同依赖 **/
public interface Caculator {

    Integer add(Integer a, Integer b);

    int minus(int a, int b);
}

/** Caculator接口的一个具体实现，位于B系统 **/
public class CaculatorImpl implements Caculator {

    @Override
    public Integer add(Integer a, Integer b) {
        return a + b;
    }

    @Override
    public int minus(int a, int b) {
        return a - b;
    }
}
```

A系统客户端调用

```java	
public class Client {
    public static void main(String[] args) {
        Caculator caculator = RPCUtils.getRemoteInstance(Caculator.class, "com.wangtao.b.service.impl.CaculatorImpl");
        System.out.println(caculator.add(1, 2));
        System.out.println(caculator.minus(2, 1));
    }
}
```

