### 前言

通过输出流的write方法可能只是会写到操作系统的缓冲区(page cache)中，再由操作系统在合适的时机将缓冲区的数据同步到磁盘中。Linux提供了两个函数fsync()、fdatasync()来强制让操作系统将数据同步到磁盘，它们的区别是是否需要同步文件的元数据，如果访问时间，修改时间，文件大小等信息，而fdatasync()不会同步文件的元数据。

### Java主动同步数据的API

- `FileDescriptor.sync()`
- `FileChannel.force()`或者`MappedByteBuffer.force()`
- 使用`RandomAccessFile`以`rws`或者`rwd`模式打开文件

#### FileDescriptor.sync()

在创建`FileOutputStream`输出流，会自动创建`FileDescriptor`对象，我们可以通过`getFD`方法拿到`FileDescriptor`对象。
```java
public FileOutputStream(File file, boolean append) throws FileNotFoundException
{
    String name = (file != null ? file.getPath() : null);
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkWrite(name);
    }
    if (name == null) {
        throw new NullPointerException();
    }
    if (file.isInvalid()) {
        throw new FileNotFoundException("Invalid file path");
    }
    this.fd = new FileDescriptor();
    fd.attach(this);
    this.append = append;
    this.path = name;

    open(name, append);
}
```

```java
try (FileOutputStream fos = new FileOutputStream("in.txt")) {  
    // write something
    fos.getFD().sync();  
}
```

**`sync`方法是一个native方法，底层调用了`fsync`**

### `FileChannel.force()`或者MappedByteBuffer.force()

通过nio中的FileChannel的force方法同步，其中`force(true)`相当于`fsync`，`force(false)`相当于`fdatasync`。也就是说`force(true)`会同步元数据。

通过`FileChannel.map()`方法拿到的`MappedByteBuffer`原理一样。

```java
try (FileOutputStream fos = new FileOutputStream("in.txt")) {  
    // write something
    // sync
    fos.getChannel().force(true);  
}

// 直接使用FileChannel
try (FileChannel channel = FileChannel.open(Paths.get("hello.txt"), StandardOpenOption.CREATE,tandardOpenOption.WRITE)) {
    // write something
    // sync
    channel.force(true);
}
```

### RandomAccessFile以rws或者rwd模式打开文件

```java
// 等于force(true), fsync
RandomAccessFile file = new RandomAccessFile("hello.txt", "rws"));
// 等于force(false), fdatasync
RandomAccessFile file = new RandomAccessFile("hello.txt", "rwd"));
```

### OutputStream.flush

flush方法只是将缓存在堆内存的数据通过输出流的write方法写入，并不保证操作系统将数据同步到磁盘中，这点可以看注释文档。

比如常见的`BufferedOutputStream`，我们在写入方法后都要调用flush方法(close方法也会自动调用flush)。来看看它的实现

```java
public synchronized void flush() throws IOException {
    // 刷新缓冲区
    flushBuffer();
    out.flush();
}

/**
 * 如果有数据, 将java堆内存的字节数组通过实际的输出流写出(也就是写入到操作系统)
 */
private void flushBuffer() throws IOException {
    if (count > 0) {
        // out通常是FileOutputStream，通过构造方法传入, 当然也可以是别的输出流
        out.write(buf, 0, count);
        count = 0;
    }
}

/**
 * BufferedOutputStream写入操作
 * 实际是往缓冲区字节数组追加元素，如果发现数据长度已经大于缓冲区字节数组长度时，就把数据刷到
 * 操作系统
 */
public synchronized void write(int b) throws IOException {
    if (count >= buf.length) {
        flushBuffer();
    }
    buf[count++] = (byte)b;
}
```