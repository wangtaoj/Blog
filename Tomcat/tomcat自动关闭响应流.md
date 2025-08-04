### 小知识点

如下代码没有主动调用flush或者close方法时，浏览器也能拿到响应数据，是因为tomcat内部的Servlet中的service方法执行完毕后，会自动调用close方法刷新缓冲区的数据。

```java
@GetMapping("/hello")
public void hello(HttpServletResponse response) throws IOException {
    // 只是写到响应流内部的缓冲区(ByteBuffer)中，当写入的内容超过缓冲区大小时会自动刷新数据
    response.getOutputStream().write("hello world".getBytes(StandardCharsets.UTF_8));
}
```

tomcat中的响应输出流实现为`CoyoteOutputStream`

CoyoteOutputStream.java

```java
@Override
public void write(byte[] b, int off, int len) throws IOException {
    boolean nonBlocking = checkNonBlockingWrite();
    ob.write(b, off, len);
    if (nonBlocking) {
        checkRegisterForWrite();
    }
}

@Override
public void flush() throws IOException {
    boolean nonBlocking = checkNonBlockingWrite();
    ob.flush();
    if (nonBlocking) {
        checkRegisterForWrite();
    }
}

@Override
public void close() throws IOException {
    ob.close();
}


```

OutputBuffer.java

```java
public void write(byte b[], int off, int len) throws IOException {
    if (suspended) {
        return;
    }
    writeBytes(b, off, len);
}

private void writeBytes(byte b[], int off, int len) throws IOException {
    if (closed) {
        return;
    }
    // 追加到内部的ByteBuffer中，没有剩余空间时会写入到底层的socket中，同时清空ByteBuffer
    append(b, off, len);
    bytesWritten += len;

    // if called from within flush(), then immediately flush
    // remaining bytes
    if (doFlush) {
        flushByteBuffer();
    }
}
```

接下来是tomcat内部执行顺序

Http11Processor.service -> CoyoteAdapter.service -> org.apache.coyote.Response.finishResponse -> OutputBuffer.close

其中CoyoteAdapter.service会调用Servlet的service方法执行响应逻辑

CoyoteAdapter.service

```java
// 这一行会触发Servlet的执行
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```

