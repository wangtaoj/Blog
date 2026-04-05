在Spring MVC中，可以不直接使用HttpServletResponse对象实现下载行为，可以通过返回Resource对象来实现。

### 传统方式

```java
@RequestMapping("/download")
public void download(HttpServletResponse response) throws IOException {
    ClassPathResource resource = new ClassPathResource("assert/in.txt");
    try (InputStream is = resource.getInputStream()) {
        response.setContentType(MediaType.APPLICATION_OCTET_STREAM_VALUE);
        String contentDisposition = ContentDisposition.attachment()
            .filename("in.txt", StandardCharsets.UTF_8)
            .build()
            .toString();
        response.setHeader(HttpHeaders.CONTENT_DISPOSITION, contentDisposition);
        IOUtils.copy(is, response.getOutputStream());
    }
}
```

自己操作HttpServletResponse进行写入。

注意: 这里有一个容易出错的写法

```java
@RequestMapping("/download")
public void download(HttpServletResponse response) throws IOException {
    ClassPathResource resource = new ClassPathResource("assert/in.txt");
    response.setContentType(MediaType.APPLICATION_OCTET_STREAM_VALUE);
        String contentDisposition = ContentDisposition.attachment()
            .filename("in.txt", StandardCharsets.UTF_8)
            .build()
            .toString();
    response.setHeader(HttpHeaders.CONTENT_DISPOSITION, contentDisposition);
    // 习惯性关闭流
    try (OutputStream os = response.getOutputStream()) {
        /*
         * 这里获取输出流逻辑发生错误时, 将会导致SpringMVC统一异常处理无效
         * 第一：由于设置了Content-Type为APPLICATION_OCTET_STREAM，前后端分类项目统一异常处理一般返回的是
         * JSON对象，这样子SpringMVC无法找到支持Content-Type为APPLICATION_OCTET_STREAM并且响应对象是一个
         * JSON对象的HttpMessageConverter, 统一异常处理会报错
         * 第二：因为响应输出流被手动关闭了, 即便统一异常处理没有错误，这后续的写入也没有任何效果，前端拿不到错误信息
         */
        try (InputStream is = getInputStrean()) {
            IOUtils.copy(is, os);
        }
    }
}
```

**tomcat容器处理请求时会对HttpServletResponse的响应输出流进行关闭，因此无需手动关闭，必要时可以使用flush方法。**

**HttpServletResponse的响应输出流close后，后续再操作响应输出流，也不会报错，只是没有效果。**

### Resouce方式

Spring提供了很多Resource的实现，比如

* ClassPathResource，将类路径下的资源转成Resource对象
* FileSystemResource，将File对象转成Resource对象
* ByteArrayResource，将字节数组转成Resource对象，这个不适用于大对象，因为不是流式的，全部加载到内存了
* InputStreamResource，将InputStream输入流转成Resource对象

所以无论是字节数组、输入流、文件，都可以将其包装成Resource对象。

```java
@RequestMapping("/downloadByResource")
public ResponseEntity<Resource> downloadByResource() {
    String contentDisposition = ContentDisposition.attachment()
        .filename("in.txt", StandardCharsets.UTF_8)
        .build()
        .toString();
    return ResponseEntity.ok()
        .contentType( MediaType.APPLICATION_OCTET_STREAM)
        .header(HttpHeaders.CONTENT_DISPOSITION, contentDisposition)
        .body(new ClassPathResource("assert/in.txt"));
}
```

如果流式控制要求更高，可以用这个，自己控制一次写多少到输出流中

```java
@GetMapping("/download-stream")
public ResponseEntity<StreamingResponseBody> downloadStream() {

    StreamingResponseBody stream = outputStream -> {
        try (InputStream in = new FileInputStream("in.txt")) {
            byte[] buffer = new byte[8192];
            int len;
            while ((len = in.read(buffer)) != -1) {
                outputStream.write(buffer, 0, len);
                outputStream.flush();
            }
        }
    };
    
    String contentDisposition = ContentDisposition.attachment()
        .filename("in.txt", StandardCharsets.UTF_8)
        .build()
        .toString();
    return ResponseEntity.ok()
            .contentType( MediaType.APPLICATION_OCTET_STREAM)
            .header(HttpHeaders.CONTENT_DISPOSITION, contentDisposition)
            .body(stream);
}
```



