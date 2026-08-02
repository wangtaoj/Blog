### 一、什么是 Tomcat 异步请求

Servlet 3.0（Java EE 6）开始引入了**异步请求（Asynchronous Request Processing）**机制，Tomcat 从 7.x 开始支持该特性。

在传统 Servlet 模型中，一个 HTTP 请求会一直占用一个 Tomcat 工作线程，直到业务处理完成并返回响应。

```
客户端
    │
    ▼
Tomcat 工作线程
    │
    ├──业务处理（可能几十秒）
    │
    └──返回响应
```

如果业务处理时间较长（例如调用远程接口、AI 大模型、消息等待等），Tomcat 工作线程会一直处于阻塞状态。

异步请求的核心思想是：

> **让处理请求的 Tomcat 工作线程尽快归还线程池，由其他线程继续完成业务，并返回响应。**

因此，一个请求生命周期中可能会涉及多个线程：

- Tomcat 工作线程（接收请求）
- 业务线程（真正执行业务，然后返回响应或者分发给另外一个Servlet）

对应流程如下：

```
客户端
    │
    ▼
Tomcat线程(T1)
    │
startAsync()
    │
释放Tomcat线程
    │
──────────────
业务线程(T2)
    │
业务处理
    │
dispatch()/complete()
    │
    ▼
客户端
```

需要注意：

**异步请求并不会让业务自动变快，它只是减少了 Tomcat 工作线程的占用时间，提高服务器的并发能力。**

### 二、适用于哪些场景

Tomcat 异步请求主要适用于**耗时操作**。

典型场景包括：

#### 1. 调用远程服务

例如：

- RPC 调用
- HTTP 接口调用
- OpenFeign
- Dubbo

等待远程响应期间，没有必要一直占用 Tomcat 线程。

#### 2. 数据库存储较慢

例如：

- 大批量写库
- 导出数据
- 复杂统计 SQL

可以让业务线程完成处理，再恢复请求。

#### 3. AI 大模型

例如：

- ChatGPT
- DeepSeek
- Claude

模型生成答案通常持续数秒甚至数分钟，非常适合异步请求。

如果结合：

- ResponseBodyEmitter
- SseEmitter

还可以实现流式输出。

#### 4. SSE

Server-Sent Events：

服务器持续向浏览器推送数据。

Spring MVC 中：

```
SseEmitter
```

底层就是基于 Servlet AsyncContext。

### 三、使用示例

#### 业务线程自己写响应

主要流程如下:

1. `@WebServlet`开启异步支持
2. `Servlet`中调用`startAsync()`方法开启异步
3. 业务线程处理逻辑，写响应，最后调用`complete()`方法结束异步过程

开启异步：

```java
@WebServlet(value = "/async", asyncSupported = true)
public class AsyncServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) {
        // 开启异步, 这样容器就不会在servlet执行完后主动关闭响应流了
        AsyncContext asyncContext = request.startAsync();

        executor.execute(() -> {

            response.setContentType("text/plain");
            try {
                // doSomething(耗时操作)
                response.getWriter().write("Hello Async");
            } catch (Exception e) {
                try {
                    response.getWriter().write("failed");
                } catch (IOException ex) {
                    throw new RuntimeException(ex);
                }
            } finally {
                // 告知servlet容器可以关闭响应了
                asyncContext.complete();
            }
        });
    }
}
```

在这个例子中，由于servlet本身没有任何耗时动作，servlet执行完后，tomcat工作线程就会被释放掉，但是对应的响应流是没有被关闭的，只有调用`asyncContext.complete()`后，容器才会执行清理动作，关闭响应流。

#### 转发给另外一个servlet执行逻辑

当前servlet中的业务线程可以做一些预处理逻辑，然后再转发给另外一个servlet完成最终的响应，spring mvc中的异步底层都是这么玩的。进行转发后，就不用再调用`asyncContext.complete()`了，转发后的servlet执行完，容器会自动调用`syncContext.complete()`。

Async2Servlet.java

```java
@Slf4j
@WebServlet(value = "/async2", asyncSupported = true)
public class Async2Servlet extends HttpServlet {

    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) {
        log.info("{}", request);
        // 开启异步
        AsyncContext asyncContext = request.startAsync();
        executor.execute(() -> {
            // doSomething(预处理逻辑)
            // 通过保存预处理结果到request中
            request.setAttribute("result", "success");

            // 转发给/async3 servlet  (不带参数则是转发给自己)
            asyncContext.dispatch("/async3");
        });
    }
}

```

Async3Servlet.java

```java
@Slf4j
@WebServlet(value = "/async3")
public class Async3Servlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        // request为ApplicationHttpRequest实例
        if (request instanceof HttpServletRequestWrapper) {
            log.info("{}", ((HttpServletRequestWrapper)request).getRequest());
        }
        response.setContentType("text/plain");
        // 获取预处理结果
        String result = (String) request.getAttribute("result");
        response.getWriter().write(result);
    }
}
```

转发后的servlet接收到的request、response都被包了一层，分别为`ApplicationHttpRequest`、`ApplicationHttpResponse`，但是内部持有的request以及response和转发之前的是同一个对象，因此可以拿到之前的请求参数，同一个响应流。

`Async3Servlet`的响应结果就是请求`Async2Servlet`的结果。

### 四、超时机制

Tomcat 异步请求支持超时控制，默认为30s，也可以单独设置，如下所示

```java
AsyncContext async = request.startAsync();
async.setTimeout(30000);
```

超时后会触发：

```java
async.addListener(new AsyncListener() {

    @Override
    public void onTimeout(AsyncEvent event) {
		// 超时回调可以写响应，也可以进行转发，响应超时之后的结果
    }

});
```

需要注意的是：

* **超时并不会强制停止业务线程，因此，业务线程需要自行处理中断、取消任务或检查超时状态。**
* **异步Servlet中只要调用了complete或者dispatch方法，超时就失效了，complete很好理解，代表异步过程结束了；调用完dispatch，比如转发后的servlet超时了，是不会触发onTimeout回调的。**
* **tomcat后台中有一个定时任务会检查异步过程是否超时，若超时，会派发一个超时事件给工作线程执行，工作线程看到事件类型是超时，便会触发onTimeout回调，源码位置: AbstractProtocol.startAsyncTimeout()。**

超时示例

```java
@Slf4j
@WebServlet(value = "/async4", asyncSupported = true)
public class Async4Servlet extends HttpServlet {

    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) {
        // 开启异步
        AsyncContext asyncContext = request.startAsync();
        // 设置1秒超时
        asyncContext.setTimeout(1000);
        Future<?> future = executor.submit(() -> {
            boolean isInterrupted = false;
            try {
                // 模拟超时操作
                TimeUnit.SECONDS.sleep(3);
                // 保存预处理结果到request中
                request.setAttribute("result", "success");
            } catch (InterruptedException e) {
                isInterrupted = true;
            }
            /*
             * 如果被中断了, 说明超时了, 就不要再调用转发了, 此时异步过程已经超时结束了，再调用会报错
             * 超时回调中，会将future取消，进而中断线程
             */
            if (!isInterrupted) {
                asyncContext.dispatch("/async3");
            }
        });

        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onComplete(AsyncEvent event)  {
                log.info("======= onComplete=========");
            }

            @Override
            public void onTimeout(AsyncEvent event)  {
                /*
                 * 这里回调如果不自己处理超时的响应, servlet容器会转发给统一的错误处理
                 * spring boot中为/error
                 */
                // 中断异步线程, 取消任务
                log.info("======= onTimeout=========");
                future.cancel(true);
                request.setAttribute("result", "timeout");
                asyncContext.dispatch("/async3");
            }

            @Override
            public void onError(AsyncEvent event)  {

            }

            @Override
            public void onStartAsync(AsyncEvent event) throws IOException {

            }
        });
    }
}

```

### 五、总结

Tomcat 异步请求的核心价值不是提高单个请求的执行速度，而是**提高服务器整体吞吐量和线程利用率**。

对于存在大量等待时间的业务（如 RPC、数据库、AI、大文件下载、SSE、长轮询等），异步请求能够显著减少 Tomcat 工作线程的占用，提高系统的并发处理能力。

在 Spring MVC 中，开发者通常无需直接操作 `AsyncContext`，而是使用 `DeferredResult`、`Callable`、`ResponseBodyEmitter`、`SseEmitter` 等高级抽象即可享受 Servlet 异步机制带来的优势。

理解其底层原理，有助于正确处理线程切换、超时控制、请求恢复以及流式响应等高级场景。