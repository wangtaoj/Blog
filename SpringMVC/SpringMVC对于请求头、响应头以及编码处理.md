### 基本知识

http协议中，请求行、请求头部分都是采用的ascii编码，是不支持中文的，若携带不支持的字符，需要使用进行编码，比如常见的urlEncode。而请求体是支持任意编码的，通过Content-Type请求头的charset部分来告知服务端请求体使用何种方式编码。

响应行、响应头、响应体亦如是。

Content-Type格式

text/html;charset=utf-8

**单个头部如果有多个值时，某些头部可以使用逗号进行分割，如Cache-Control、Accept、Content-Type，在Http1.1中，使用多个同名头字段来表示，任意头部都可。**

### 如何获取请求头

第一种方式，使用servlet中的api

```java
@GetMapping("/receiveHeader")
public void receiveHeader(HttpServletRequest request) {
    // 获取第一个值
    // String value = request.getHeader("myHead");
    // 获取所有值
    Enumeration<String> myHead = request.getHeaders("myHead");
    while (myHead.hasMoreElements()) {
        System.out.println(myHead.nextElement());
    }
}
```

第二种方式，使用SpringMVC提供的@RequestHeader注解

```java
/**
 * 功能比较强大，支持类型转换
 * 接收多个值可使用容器类型、数组类型接收。
 * 当然了这个方式也是封装了下servlet的原生api实现的
 */
@GetMapping("/receiveHeader")
public void receiveHeader(@RequestHeader(name="myHead") List<Integer> myHeads) {
    System.out.println(myHeads);
}
```

### 如何设置响应头

第一种方式，使用servlet中的api

```java
@GetMapping("/setHeader")
public String setHeader(HttpServletResponse response) {
    // 覆盖
    response.setHeader("myHead1", "value");
    // 添加一个同名头字段
    response.addHeader("myHead2", "value1");
    response.addHeader("myHead2", "value2")
    // 特殊响应头, 有些响应头比较重要，直接作为response的属性
    // 当调用setHeader或者addHeader方法时，内部直接调用对应的setXxx方法
    response.setContentType("text/html;charset=utf-8");
    return "success";
}
```

第二种方式，使用SpringMVC提供的HttpEntity、ResponseEntity(继承了HttpEntity，多了响应码)

```java
@GetMapping("/setHeader1")
public HttpEntity<String> setHeader1() {
    HttpHeaders headers = new HttpHeaders();
    // add方法, 添加一个同名头字段
    headers.add("myHead1", "value1");
    headers.add("myHead1", "value2");
    headers.addAll("myHead2", List.of("value1", "value2"));
    // set方法
    headers.set("myHead3", "value");
    // 特殊响应头
    headers.setContentType(MediaType.TEXT_HTML);
    return new HttpEntity<>("success", headers);
}
```

SpringMVC提供的这种方式肯定也是封装了servlet的api实现的，它在调用响应流写响应体时会先把设置的请求头调用servlet的api放到response中。

```java
// ServletServerHttpResponse.java
@Override
public OutputStream getBody() throws IOException {
    this.bodyUsed = true;
    // 写响应头到HttpServletResponse
    writeHeaders();
    // 调用HttpServletResponse api返回响应输出流
    return this.servletResponse.getOutputStream();
}
```

### Content-Type响应头以及编码

Content-Type响应头非常重要，告知了客户端响应体数据类型以及编码方式，在SpringMVC开发模式中正常情况下是不用开发者来设置的，但是如果通过Servlet API来设置这个响应头会出现一些诡异的现象而掉进坑里，我认为这里是Spring实现上的bug导致的。这里简要分析下源码以及bug产生原因。

`AbstractMessageConverterMethodProcessor.writeWithMessageConverters`方法中会获取用户设置的`Content-Type`头部。

```java
/**
 * 精简代码
 */
void writeWithMessageConverters() {
    MediaType contentType = outputMessage.getHeaders().getContentType();
}

```

HttpHeaders.java

```java
public MediaType getContentType() {
    // getFirst方法被重写了
    String value = getFirst(CONTENT_TYPE);
    return (StringUtils.hasLength(value) ? MediaType.parseMediaType(value) : null);
}
```

ServletServerHttpResponse.ServletResponseHttpHeaders.java

```java
@Override
@Nullable
public String getFirst(String headerName) {
    if (headerName.equalsIgnoreCase(CONTENT_TYPE)) {
        // Content-Type is written as an override so check super first
        // 如果通过HttpEntity方式设置的，这里可以拿到值
        String value = super.getFirst(headerName);
        // 通过Servlet Api设置的值
        return (value != null ? value : servletResponse.getHeader(headerName));
    }
    else {
        String value = servletResponse.getHeader(headerName);
        return (value != null ? value : super.getFirst(headerName));
    }
}
```

从代码中可以看到Spring是想过要从`HttpServletResponse`对象中获取`Content-Type`头的，但是不幸的是它的getHeader方法对于`Content-Type`头部获取是有限制的，必须等到把请求数据写出后才能获取到，在前面也说过设置`Content-Type`头部时其实调用的是`setContentType`方法，即便通过`setHeader`方法来设置时，内部也是调用的`setContentType`方法，需要使用`getContentType`方法才能正确获取到值。

看一个例子

```java
@GetMapping("/hello")
public void hello(HttpServletResponse response) throws IOException {
    response.setHeader(HttpHeaders.CONTENT_TYPE, MediaType.TEXT_HTML_VALUE);
    // null
    System.out.println(response.getHeader(HttpHeaders.CONTENT_TYPE));
    // 可以拿到值
    System.out.println(response.getContentType());
    response.getOutputStream().write("Hello Spring Boot 3".getBytes());
    // 将数据真实写出, 或者这里也可以调用close方法
    response.getOutputStream().flush();
    // 可以拿到值
    System.out.println(response.getHeader(HttpHeaders.CONTENT_TYPE));
}
```

**因此如果不是自己直接使用HttpServletResponse的输出流来输出数据，那么务必使用HttpEntity方式来设置该请求头。**

#### 原生Servlet写法

先看原生servlet写法，直接使用HttpServletResponse的响应流输出数据

```java
/**
 * 输出响应(字节流方式)
 * 字节流方式，由自己控制编码，必须保证和Content-Type中的charset保持一致，否则乱码
 */
@GetMapping("/printBody")
public void printBody(HttpServletResponse response) throws IOException {
    // text/html;charset=UTF-8
    response.setContentType(new MediaType(MediaType.TEXT_HTML, StandardCharsets.UTF_8).toString());
    response.getOutputStream().write("中文字符".getBytes(StandardCharsets.UTF_8));
    response.getOutputStream().flush();

    //response.getWriter().write("中文字符");
}

/**
 * 输出响应(字符流方式)
 * response内部写字符流时最终都会变成写字节流，会使用Content-Type中指定的编码，这样就不会乱码了
 */
@GetMapping("/printBody")
public void printBody(HttpServletResponse response) throws IOException {
    // text/html;charset=UTF-8
    response.setContentType(new MediaType(MediaType.TEXT_HTML, StandardCharsets.UTF_8).toString());
    response.getWriter().write("中文字符");
    response.getWriter().flush();
}
```

注1：response对象还有一个setCharacterEncoding("utf-8")，效果是一样的，也是设置编码，只不过setContentType方法同时设置了contentType、charset两个属性，response对象是有连个字段来存储的，contentType属性存储不带charset的部分。

注2：由于这种方式不会经过SpringMVC的handleReturnValue逻辑，因此直接通过response的api设置不会出现问题。

注3：什么情况下，不会经过SpringMVC的handleReturnValue逻辑呢？

handleReturnValue主要逻辑就是根据Hander(Controller中的映射方法)的返回值，去找最匹配的HttpMessageConverter，将这个数据调用ServletHttpResponse的输出流将数据响应给客户端，同时会设置对应的Content-Type。

```java
/**
 * 关键代码
 * ServletInvocableHandlerMethod.java
 */
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {
    // 执行hander方法获取返回值
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    setResponseStatus(webRequest);

    // 如何返回值为null(前提条件)
    if (returnValue == null) {
        // 存在响应状态码或者mavContainer.isRequestHandled()
        if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
            disableContentCachingIfNecessary(webRequest);
            mavContainer.setRequestHandled(true);
            return;
        }
    }
    else if (StringUtils.hasText(getResponseStatusReason())) {
        mavContainer.setRequestHandled(true);
        return;
    }

    mavContainer.setRequestHandled(false);
    Assert.state(this.returnValueHandlers != null, "No return value handlers");
    try {
        // 处理返回值，响应数据给客户端
        this.returnValueHandlers.handleReturnValue(
                returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    }
    catch (Exception ex) {
        if (logger.isTraceEnabled()) {
            logger.trace(formatErrorForReturnValue(returnValue), ex);
        }
        throw ex;
    }
}
```

**当Controller方法返回null时(或者方法返回类型为void也是返回null)，此为前提条件。**

第一种情况，响应状态码有值，即存在@ResponseStatus注解

```java
@GetMapping("/printBody")
@ResponseStatus(HttpStatus.OK)
public String printBody1() throws IOException {
    return null;
}

@GetMapping("/printBody")
@ResponseStatus(HttpStatus.OK)
public void printBody1() throws IOException {
    
}
```

第二种情况，mavContainer.isRequestHandled() = true

当Controller方法参数中存在HttpServletResponse参数时，会将requestHandlerd设置成true

```java
@GetMapping("/printBody")
public void printBody(HttpServletResponse response) throws IOException {
    
}
```

其它情况不去细究了。

#### SpringMVC处理响应体

现在大部分情况下写法都是由SpringMVC自己来帮我们处理响应体的，尤其是application/json形式，默认编码就是utf-8。目前想到的就是下载情况了，由我们设置Content-Type，调用响应流来输出数据，即上面那种写法。

那么`handleReturnValue`方法内部逻辑是如何来寻找最合适的Content-Type(MediaType)

具体逻辑位于`AbstractMessageConverterMethodProcessor.writeWithMessageConverters`，由`handleReturnValue`方法内部调用。

1. 如果通过HttpEntity(或者ResponseEntity)设置了Content-Type响应头，并且要是具体的，没有带通配符，那么直接结束就是它了。

2. 获取请求头Accept的值，返回一个MediaType列表，记为acceptableTypes

3. 获取服务端可以产生的MediaType列表，记为producibleMediaTypes

   3.1 从HttpServletRequest获取，如获取到则producibleMediaTypes就是它了

   ```java
   /**
    * 由Controller方法上的@RequestMapping中的produces属性指定
    * 如下面例子
    * @GetMapping(value = "/", produces = {MediaType.APPLICATION_JSON_VALUE, "text/html;charset=UTF-8"})
    */
   Set<MediaType> mediaTypes = (Set<MediaType>) request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
   ```

   3.2 根据响应体的类型，从一系列HttpMessageConverter中获取MediaType列表，根据canWrite方法来判断该HttpMessageConverter是否满足条件。

4. 使用acceptableTypes中的值来对producibleMediaTypes进行过滤，最后排序找到一个最佳的MediaType。
5. 如果这个MediaType中没有charset部分，使用最终处理的HttpMessageConverter里的charset组成一个新的MediaType。

```java
MediaType finalMediaType = new MediaType(mediaType, charset);
```

6. **终于找到合适的mediaType了，也就是Content-Type，然后设置Content-Type响应头，这一步会覆盖掉我们自己在HttpServletResponse中设置的Content-Type以及Charset。**

```java
@Override
public final void write(final T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException {

    final HttpHeaders headers = outputMessage.getHeaders();
    /**
     * 添加Content-Type响应头
     * 这里只是添加到SpringMVC自己的对象中，真正添加到HttpServletResponse中是真正写响应体时
     * 调用outputMessage.getBody方法触发
     */
    addDefaultHeaders(headers, t, contentType);

    // 省略分支逻辑
    // 真正写数据到输出流中
    writeInternal(t, outputMessage);
    outputMessage.getBody().flush();
}

protected void addDefaultHeaders(HttpHeaders headers, T t, @Nullable MediaType contentType) throws IOException {
    // 响应头中Content-Type为空，使用mediaType给它赋值
    if (headers.getContentType() == null) {
        MediaType contentTypeToUse = contentType;
        if (contentType == null || !contentType.isConcrete()) {
            contentTypeToUse = getDefaultContentType(t);
        }
        else if (MediaType.APPLICATION_OCTET_STREAM.equals(contentType)) {
            MediaType mediaType = getDefaultContentType(t);
            contentTypeToUse = (mediaType != null ? mediaType : contentTypeToUse);
        }
        if (contentTypeToUse != null) {
            // mediaType中charset为null，使用该HttpMessageConverter中的charset
            if (contentTypeToUse.getCharset() == null) {
                Charset defaultCharset = getDefaultCharset();
                if (defaultCharset != null) {
                    contentTypeToUse = new MediaType(contentTypeToUse, defaultCharset);
                }
            }
            // 设置Content-Type
            headers.setContentType(contentTypeToUse);
        }
    }
    if (headers.getContentLength() < 0 && !headers.containsKey(HttpHeaders.TRANSFER_ENCODING)) {
        Long contentLength = getContentLength(t, headers.getContentType());
        if (contentLength != null) {
            headers.setContentLength(contentLength);
        }
    }
}

```

将SpringMVC自己的响应头同步到HttpServletResponse中

响应数据肯定要获取到响应输出流，通过HttpOutputMessage的getBody来获取

```java
// ServletServerHttpResponse.java
@Override
public OutputStream getBody() throws IOException {
    this.bodyUsed = true;
    // 写响应头到HttpServletResponse
    writeHeaders();
    // 调用HttpServletResponse api返回响应输出流
    return this.servletResponse.getOutputStream();
}

/**
 * 可以看到会直接覆盖Content-Type以及charset，也就是自己设置的没有用
 */
private void writeHeaders() {
    if (!this.headersWritten) {
        getHeaders().forEach((headerName, headerValues) -> {
            for (String headerValue : headerValues) {
                this.servletResponse.addHeader(headerName, headerValue);
            }
        });
        // HttpServletResponse exposes some headers as properties: we should include those if not already present
        if (this.servletResponse.getContentType() == null && this.headers.getContentType() != null) {
            this.servletResponse.setContentType(this.headers.getContentType().toString());
        }
        if (this.servletResponse.getCharacterEncoding() == null && this.headers.getContentType() != null &&
                this.headers.getContentType().getCharset() != null) {
            this.servletResponse.setCharacterEncoding(this.headers.getContentType().getCharset().name());
        }
        long contentLength = getHeaders().getContentLength();
        if (contentLength != -1) {
            this.servletResponse.setContentLengthLong(contentLength);
        }
        this.headersWritten = true;
    }
}
```

因此如果我们响应字符串时，想要调整编码方式时，一定要使用HttpEntity方式或者指定@RequestMapping注解的produces属性。

看一些例子:

```java
@GetMapping("/charsetTest")
public String charsetTest(HttpServletResponse response)  {
    /**
     * 响应头设置无效
     * SpringMVC最终找到最符合的是MediaType.TEXT_HTML，且它没有charset部分
     * 而在SpringBoot中内置的StringHttpMessageConverter使用utf-8编码，因此不会乱码
     * 实际返回的Content-Type是text/html;charset=UTF-8
     */
    response.setContentType(MediaType.TEXT_HTML_VALUE);
    response.setCharacterEncoding(StandardCharsets.ISO_8859_1.name());
    return "hello，中国";
}
```

```java
/**
 * 乱码
 * 实际返回的Content-Type是text/html;charset=ISO-8859-1
 * ISO-8859-1不支持中文
 */
@GetMapping(value = "/charsetTest", produces = {"text/html;charset=ISO-8859-1"})
public String charsetTest()  {
    return "hello，中国";
}

/**
 * 乱码
 * 实际返回的Content-Type是text/html;charset=ISO-8859-1
 * ISO-8859-1不支持中文
 */
@GetMapping(value = "/charsetTest")
public HttpEntity<String> charsetTest()  {
    HttpHeaders headers = new HttpHeaders();
    // 设置响应头, 且带编码
    headers.setContentType(new MediaType(MediaType.TEXT_HTML, StandardCharsets.ISO_8859_1));
    return new HttpEntity<>("hello，中国", headers);
}
```

