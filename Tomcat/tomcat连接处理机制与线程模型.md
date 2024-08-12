### 前提

Tomcat中Connector实现主要有3种，NIO、NIO2、APR，其中NIO是默认方式。

NIO基于`ServerSocketChannel`

NIO2基于`AsynchronousServerSocketChannel`

后面都是基于NIO模式来进行阐述。

### Tomcat线程池

tomcat中的线程池实现为`ThreadPoolExecutor`，基本上是复制了JDK中的`ThreadPoolExecutor`，与JDK中的线程池主要区别在于**阻塞队列实现，execute方法多了一层异常捕获，超出最大线程数量后会抛出异常，捕获它将任务插入到阻塞队列中**。

JDK中的线程池提交逻辑是，小于核心线程数量时，新建线程来执行任务，超过核心线程数量后，放入阻塞队列中，阻塞队列满了，新建线程来执行任务，当线程数量超过最大线程数时，则执行拒绝策略。

tomcat中的线程池提交逻辑是，小于核心线程数量时，新建线程来执行任务，超过核心线程数量后，**还是新建线程来执行任务，当线程数量超过最大线程数时，将任务放入阻塞队列中，阻塞队列满了，则抛出异常。**

**tomcat的目标是要尽可能的多开线程来处理请求，而不是优先放入阻塞队列，因为tomcat中的任务主要是IO密集型，并且要尽快响应请求数据给用户，而不是排队等着。**

### Tomcat线程池阻塞队列

tomcat中的阻塞队列为`TaskQueue`，它继承了`LinkedBlockingQueue`，主要看offer方法实现。

```java
public class TaskQueue extends LinkedBlockingQueue<Runnable> {
    
    private transient volatile ThreadPoolExecutor parent = null;

    @Override
    public boolean offer(Runnable o) {
        //we can't do any checks
        if (parent==null) {
            return super.offer(o);
        }
        // 线程池的数量已经达到最大值, 走原逻辑
        if (parent.getPoolSizeNoLock() == parent.getMaximumPoolSize()) {
            return super.offer(o);
        }
        // 线程池中的任务小于等于线程数量，此时有线程来执行新增加的任务，因此直接插入到队列中
        if (parent.getSubmittedCount() <= parent.getPoolSizeNoLock()) {
            return super.offer(o);
        }
        // 线程值中的数量小于最大值, 返回false, 代表队列满了，于是线程池则会新建线程来执行任务
        if (parent.getPoolSizeNoLock() < parent.getMaximumPoolSize()) {
            return false;
        }
        //if we reached here, we need to add it to the queue
        return super.offer(o);
    }
}
```

### Tomcat线程池execute方法

```java
@Override
public void execute(Runnable command) {
    // 提交任务数量加1, 任务执行完后会减1，逻辑在afterExecute中
    submittedCount.incrementAndGet();
    try {
        // 执行提交逻辑, 这个和JDK中的线程池execute一模一样
        executeInternal(command);
    } catch (RejectedExecutionException rx) {
        /*
         * DK中提交任务时线程数量达到最大数量后会执行拒绝策略，默认是抛出异常
         * 这里捕获该异常将任务插入到阻塞队列中
         * 这样就实现了先新增线程执行任务，达到最大线程数量后将任务插入到阻塞队列中
         */
        if (getQueue() instanceof TaskQueue) {
            // If the Executor is close to maximum pool size, concurrent
            // calls to execute() may result (due to Tomcat's use of
            // TaskQueue) in some tasks being rejected rather than queued.
            // If this happens, add them to the queue.
            final TaskQueue queue = (TaskQueue) getQueue();
            if (!queue.force(command)) {
                submittedCount.decrementAndGet();
                throw new RejectedExecutionException(sm.getString("threadPoolExecutor.queueFull"));
            }
        } else {
            submittedCount.decrementAndGet();
            throw rx;
        }
    }
}
```

### Tomcat connector的几个重要参数

* maxConnections，默认值为8192，允许的最大socket连接数，当连接数达到最大值，acceptor线程将会阻塞，不再调用`ServerSocketChannel.accept()`方法接收新的socket连接。
* acceptCount，默认值为100，操作系统挂起socket连接的最大值，等同于`ServerSocketChannel.bind()`方法的`backlog`参数。当客户端连接到服务端时，如果服务端没有调用accept获取连接，这个连接将会处于一个队列中，当队列中的连接数超过`backlog`参数后，客户端将会报错connection refuse。而调用accept方法则会从队列中取走一个连接。
* minSpareThreads，默认值10，线程池的核心线程数量。
* maxThreads，默认值200，线程池的最大线程数量。

总结：

因此tomcat默认可以处理的连接总数是maxConnections + acceptCount，超出后则会直接报错。

tomcat处理请求时，都会使用一个线程来处理，如果需要处理的连接数量(已经发起请求数据了，空闲的连接不算)大于线程池数量时，多余的连接请求则会被加入到阻塞队列中。

### 对应的Spring Boot配置

* server.tomcat.max-connections
* server.tomcat.accept-count
* server.tomcat.threads.min-spare
* server.tomcat.threads.max

可以根据这些配置顺藤摸瓜找到`TomcatWebServerFactoryCustomizer`配置类，可以看这个类怎么将这些配置给设置到tomcat中。

### Tomcat连接处理机制

NIO模式核心的类为`NioEndpoint`、`Acceptor`、`Poller`。

`NioEndpoint.start()`方法会创建`ServerSocketChannel`监听连接，同时开启`Acceptor`、`Poller`两个线程。

Acceptor线程无线循环调用`ServerSocketChannel.accept()`获取新的SocketChannel连接，然后将其设置为非阻塞模式，注册读事件，供Poller线程进行事件循环。在每次调用accept之前，都会检查连接数量有没有达到maxConnections，如果达到了则会先阻塞，直到已有的连接关闭。**accept是阻塞式的，读取请求数据是非阻塞式的。**

poller线程也是一个无限循环，主要是调用`Selector.select()`方法获取可以读写的连接，然后分发给线程池来执行。每次循环主要做3件事件。

* 将Acceptor注册的`PollerEvent`事件转换成`SelectionKey事件`，这里为啥要折中下，**因为Selector中的方法存在很多锁，因此在单线程中处理是比较好的选择，否则容易出现死锁现象。**比如`Selector.select`方法和`SocketChannel.register(selector)`就会死锁。
* 调用`Selector.select`方法获取就绪的`SelectionKey`，分发给线程池来处理请求逻辑。
* 处理超时的socket连接，tomcat在http1.1协议下，因为keepAlive默认开启，是不会主动关闭socket连接的，如果已经建立的socket连接没有主动关闭，在20s(默认值)没有发送请求时，tomcat会主动关闭该socket连接。

**关于acceptCount**

`NioEndpoint.java`

```java
public void bind() throws Exception {
    initServerSocket();

    setStopLatch(new CountDownLatch(1));

    // Initialize SSL if needed
    initialiseSsl();
}  


protected void initServerSocket() throws Exception {

    serverSock = ServerSocketChannel.open();
    InetSocketAddress addr = new InetSocketAddress(getAddress(), getPortWithOffset());
    // 关键点
    serverSock.bind(addr, getAcceptCount());
    // 这里是阻塞模式哦，也就是接收连接是阻塞的
    serverSock.configureBlocking(true);
} 
```

**关于maxConnections**

`Acceptor.java`

```java
/**
 * 简写形式，伪代码
 */
public void run() {
    while (true) {
        endpoint.countUpOrAwaitConnection();
        SocketChannel result = serverSock.accept();
        // 向Poller中注册PollerEvent
    }
}

protected void countUpOrAwaitConnection() throws InterruptedException {
    // -1不控制连接数量
    if (maxConnections==-1) {
        return;
    }
    // 控制连接数量, 作用和jdk中的Semaphore类似
    LimitLatch latch = connectionLimitLatch;
    if (latch!=null) {
        // 达到maxConnections线程会阻塞
        latch.countUpOrAwait();
    }
}
```

**关于minSpareThreads、maxThreads**

`NioEndpoint.java`

```java
@Override
public void startInternal() throws Exception {

    if (getExecutor() == null) {
        createExecutor();
    }

    // 开启Poller线程
    poller = new Poller();
    Thread pollerThread = new Thread(poller, getName() + "-Poller");
    pollerThread.setPriority(threadPriority);
    pollerThread.setDaemon(true);
    pollerThread.start();
	// 开启Acceptor线程
    startAcceptorThread();
}

public void createExecutor() {
    internalExecutor = true;
    // 使用虚拟线程(JDK21)
    if (getUseVirtualThreads()) {
        executor = new VirtualThreadExecutor(getName() + "-virt-");
    } else {
        // 使用tomcat定制的阻塞队列，改变线程池提交行为，阻塞队列是无界的，容量为Integer.MAX
        TaskQueue taskqueue = new TaskQueue();
        TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
        // 创建线程池, 核心线程数量，最大线程数量
        executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
        taskqueue.setParent( (ThreadPoolExecutor) executor);
    }
}
```

### Socket连接关闭时机

两种正常关闭情况

* 客户端主动关闭socket，此时会触发READ事件，Poller线程会分发任务给线程池，读取数据时发现已到达流末尾，返回-1，发生EOFExcetion，关闭连接。
* Poller线程每一次循环的最后，只要满足清理条件(清理时间满足，线程当前比较空闲，没有就绪的事件和要转换的PollerEvent)，都会去关闭已经超时的socket连接(超时判定，超过20s没有发送请求数据，20s是默认时间)

### 简易版tomcat连接实现

核心机制与tomcat一样，简化了处理请求逻辑，只能处理以\n结尾的请求数据，并简单打印到控制台。

[TomcatServer](https://github.com/wangtaoj/JavaCore/blob/master/src/main/java/com/wangtao/nio/tomcat/TomcatServer.java)

[TomcatClient](https://github.com/wangtaoj/JavaCore/blob/master/src/main/java/com/wangtao/nio/tomcat/TomcatClient.java)