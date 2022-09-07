### 声明

> 源码基于Spring Boot 2.3.12.RELEASE、Spring Framework 5.2.15.RELEASE

### Servlet3.0 文件上传

Servlet 3.0对于`HttpServletRequest`接口增加了`getParts`方法，从而不用再借助`apache commons-fileupload`组件来获取文件相关信息。

```java
/**
 * 获取所有参数
 */
Collection<Part> getParts();

/**
 * 根据参数名获取
 */
Part getPart(String name) throws IOException, ServletException;
```

对于文件上传提交，即`content-type=multipart/form-data`，还需要使用以下注解搭配才能使用上面的方法获取到信息。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MultipartConfig {

    /**
     * 文件上传时的临时路径
     */
    String location() default "";

    /**
     * 单个文件最大值
     * 默认-1，不限制
     */
    long maxFileSize() default -1L;

    /**
     * 整个请求的数据最大值
     * 默认-1，不限制
     */
    long maxRequestSize() default -1L;

    /**
     * 每次写入磁盘的阈值
     */
    int fileSizeThreshold() default 0;
}
```

下面看下使用例子

假设前台文件上传请求如下

| key   | value  |
| ----- | ------ |
| file1 | in.txt |
| file2 | in.txt |
| count | 2      |

```java
@MultipartConfig
@WebServlet(urlPatterns = "/uploadFile")
public class UploadServlet extends HttpServlet {
    private static final long serialVersionUID = 318064779855484536L;

    /**
     * @MultipartConfig注解必须，否则下面方法获取不到任何数据
     * 即使是count参数
     */
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // {count=[2]}
        Map<String, String[]> parameterMap = req.getParameterMap();
        System.out.println(parameterMap);
        // 2
        System.out.println(req.getParameter("count"));
        // 集合数目为3条，普通参数也会包含，只不过普通参数只能获取到name属性，其它的都是null
        Collection<Part> parts = req.getParts();
        for (Part part : parts) {
            // 参数名字
            System.out.println(part.getName());
            // 文件类型, 如文本(text/plain), 图片(image/png)
            System.out.println(part.getContentType());
            // 文件名
            System.out.println(part.getSubmittedFileName());
            // 文件流
            if (part.getName().equals("file1") || part.getName().equals("file2")) {
                System.out.println(IOUtils.toString(part.getInputStream(), StandardCharsets.UTF_8));
            }
            System.out.println("=======================");
        }
    }
}
```

MultipartConfig代码配置

```java
// 注册Servlet以及初始化配置
ServletRegistration.Dynamic registration = servletContext.addServlet(
    "uploadServlet",
    new UploadServlet()
);
registration.addMapping("/uploadFile");
// 全部使用默认值
registration.setMultipartConfig(new MultipartConfigElement(""));
```

### SpringMVC文件上传

使用SpringMVC进行文件上传，我们需要给`DispatcherServlet`赋值一个`multipartResolver`。只需要往容器中注入一个id为**`multipartResolver`**的bean即可，`DispatcherServlet`初始化时会自动从容器中搜索id为`multipartResolver`进行赋值。

目前SpringMVC中给`MultipartResolver`组件提供了2种实现，如下所示

* `StandardServletMultipartResolver`，依赖于`Servlet 3.0 API`，上面章节所述，Spring Boot项目默认方式。
* `CommonsMultipartResolver`，依赖于`apache commons-fileupload`

只需往容器中注入即可(**两者使用其一**)

```java
@Configuration
public class WebMvcConfig {
    
    /**
     * 使用该方式时不要忘记给DispatcherServlet设置MultipartConfig
     * 参考MultipartConfig代码配置
     */
    @Bean
    public MultipartResolver multipartResolver() {
        return new StandardServletMultipartResolver();
    }
    
    @Bean
    public MultipartResolver multipartResolver() {
        final long _1M = 1024 * 1024;
        CommonsMultipartResolver multipartResolver = new CommonsMultipartResolver();
        // 一次请求数据最大值
        multipartResolver.setMaxUploadSize(_1M * 20);
        // 编码
        multipartResolver.setDefaultEncoding("UTF-8");
        // 单个文件最大值
        multipartResolver.setMaxUploadSizePerFile(_1M * 10);
        return multipartResolver;
    }
}
```

Controller

```java
@RestController
public class UploadController {

    @PostMapping("/upload")
    public ResponseEntity<String> upload(MultipartFile file1, MultipartFile file2, Integer count) throws IOException {
        // 文件名
        System.out.println(file1.getOriginalFilename());
        // 文件类型 如文本(text/plain), 图片(image/png)
        System.out.println(file1.getContentType());
        String string1 = IOUtils.toString(file1.getInputStream(), StandardCharsets.UTF_8);
        System.out.println(string1);
        String string2 = IOUtils.toString(file2.getInputStream(), StandardCharsets.UTF_8);
        System.out.println(string2);
        System.out.println(count);
        return ResponseEntity.of(Optional.of("success"));
    }
}
```

### SpringMVC 文件上传原理

SpringMVC将所有请求交给`DispatcherServlet`处理，对于文件上传请求，会使用`MultipartResolver`组件包装请求。

1. 先看`MultipartResolver`组件初始逻辑

`DispatcherServlet.java`

```java
@Override
protected void onRefresh(ApplicationContext context) {
    // 初始各大组件
    initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
	// 初始MultipartResolver组件
    initMultipartResolver(context);
    ....
}

private void initMultipartResolver(ApplicationContext context) {
    try {
        // 从容器中获取id为multipartResolver的bean
        this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.multipartResolver);
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.multipartResolver.getClass().getSimpleName());
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        // Default is no multipart resolver.
        this.multipartResolver = null;
        if (logger.isTraceEnabled()) {
            logger.trace("No MultipartResolver '" + MULTIPART_RESOLVER_BEAN_NAME + "' declared");
        }
    }
}
```

2. 借助`MultipartResolver`包装Request

`DispatcherServlet.java`

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // 判断是不是文件上传
            processedRequest = checkMultipart(request);
            ...
        }
    }
}

protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
    if (this.multipartResolver != null &&
        this.multipartResolver.isMultipart(request)) {
        // 包装请求
        return this.multipartResolver.resolveMultipart(request);
    }
    return request;
}
```

3. `StandardServletMultipartResolver`

```java
public class StandardServletMultipartResolver implements MultipartResolver {

	private boolean resolveLazily = false;

	public void setResolveLazily(boolean resolveLazily) {
		this.resolveLazily = resolveLazily;
	}

    /**
     * 判断是否文件上传
     */
	@Override
	public boolean isMultipart(HttpServletRequest request) {
		return StringUtils.startsWithIgnoreCase(request.getContentType(), "multipart/");
	}

    /**
     * 包装请求为MultipartHttpServletRequest
     * 该接口存在MultipartFile对象的方法
     */
	@Override
	public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException {
		return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
	}

    /**
     * 清理操作
     */
	@Override
	public void cleanupMultipart(MultipartHttpServletRequest request) {
		if (!(request instanceof AbstractMultipartHttpServletRequest) ||
				((AbstractMultipartHttpServletRequest) request).isResolved()) {
			// To be on the safe side: explicitly delete the parts,
			// but only actual file parts (for Resin compatibility)
			try {
				for (Part part : request.getParts()) {
					if (request.getFile(part.getName()) != null) {
						part.delete();
					}
				}
			}
			catch (Throwable ex) {
				LogFactory.getLog(getClass()).warn("Failed to perform cleanup of multipart items", ex);
			}
		}
	}

}
```

主要逻辑还是在`StandardMultipartHttpServletRequest`类中

4. `StandardMultipartHttpServletRequest`

```java
public StandardMultipartHttpServletRequest(HttpServletRequest request, 
    boolean lazyParsing) throws MultipartException {
    super(request);
    if (!lazyParsing) {
        // 解析请求
        parseRequest(request);
    }
}

private void parseRequest(HttpServletRequest request) {
    try {
        // 使用Servlet 3.0API
        Collection<Part> parts = request.getParts();
        this.multipartParameterNames = new LinkedHashSet<>(parts.size());
        MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<>(parts.size());
        for (Part part : parts) {
            String headerValue = part.getHeader(HttpHeaders.CONTENT_DISPOSITION);
            ContentDisposition disposition = ContentDisposition.parse(headerValue);
            String filename = disposition.getFilename();
            // 根据文件名是否有值来区分是文件参数还是普通参数
            if (filename != null) {
                if (filename.startsWith("=?") && filename.endsWith("?=")) {
                    filename = MimeDelegate.decode(filename);
                }
                // 文件参数
                files.add(part.getName(), new StandardMultipartFile(part, filename));
            }
            else {
                // 普通参数
                this.multipartParameterNames.add(part.getName());
            }
        }
        // 赋值
        setMultipartFiles(files);
    }
    catch (Throwable ex) {
        handleParseFailure(ex);
    }
}

/**
 * 根据参数名获取MultipartFile
 */
@Override
public MultipartFile getFile(String name) {
    return getMultipartFiles().getFirst(name);
}

/**
 * 根据参数名获取MultipartFile列表
 */
@Override
public List<MultipartFile> getFiles(String name) {
    List<MultipartFile> multipartFiles = getMultipartFiles().get(name);
    if (multipartFiles != null) {
        return multipartFiles;
    }
    else {
        return Collections.emptyList();
    }
}
```

5. SpringMVC如何给Controller中方法的`MultipartFile`对象赋值呢？

SpringMVC给Controller方法中的参数赋值是借助`HandlerMethodArgumentResolver`组件实现的

```java
public interface HandlerMethodArgumentResolver {

	/**
	 * 是否支持该参数
	 */
	boolean supportsParameter(MethodParameter parameter);

    /**
     * 解析参数值
     */
	@Nullable
	Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;

}
```

而负责给`MultipartFile`对象赋值的接口实现为`RequestParamMethodArgumentResolver`

```java
@Override
public boolean supportsParameter(MethodParameter parameter) {
    // 忽略该方法其它逻辑
    if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
        return true;
    }
}

/**
 * 如果参数类型是MultipartFile或者MultipartFile[]
 * 或者List<MultipartFile>、Collection<MultipartFile>
 */
public static boolean isMultipartArgument(MethodParameter parameter) {
    Class<?> paramType = parameter.getNestedParameterType();
    return (MultipartFile.class == paramType ||
            isMultipartFileCollection(parameter) || isMultipartFileArray(parameter) ||
            (Part.class == paramType || isPartCollection(parameter) || isPartArray(parameter)));
}
```

再看解析参数值逻辑

```java
/**
 * 该方法在父类AbstractNamedValueMethodArgumentResolver的resolveArgument方法调用
 */
@Override
@Nullable
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
    HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

    if (servletRequest != null) {
        // 从请求参数中获取值
        Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
        if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
            return mpArg;
        }
    }

    Object arg = null;
    // 如果还没获取到，再次尝试获取
    MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
    if (multipartRequest != null) {
        List<MultipartFile> files = multipartRequest.getFiles(name);
        if (!files.isEmpty()) {
            arg = (files.size() == 1 ? files.get(0) : files);
        }
    }
    if (arg == null) {
        String[] paramValues = request.getParameterValues(name);
        if (paramValues != null) {
            arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
        }
    }
    return arg;
}
```

```java
/**
 * 从封装好的MultipartHttpServletRequest对象中获取参数
 * 该方法还会有一个兜底的处理逻辑，即便容器中没有配置MultipartResolver组件，也能成功获取到参数
 */
@Nullable
public static Object resolveMultipartArgument(String name, MethodParameter parameter, HttpServletRequest request)
    throws Exception {
	// 获取经过MultipartResolver包装好的MultipartHttpServletRequest请求对象
    MultipartHttpServletRequest multipartRequest =
        WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class);
    /*
     * 判断是否是文件上传
     * 如果是multipartRequest == null，则没有经过MultipartResolver处理
     * isMultipartContent方法通过content-type是否以multipart/开头
     */
    boolean isMultipart = (multipartRequest != null || isMultipartContent(request));

    if (MultipartFile.class == parameter.getNestedParameterType()) {
        // 参数类型是MultipartFile
        if (multipartRequest == null && isMultipart) {
            // 如果是文件上传请求且multipartRequest == null，手动构造
            multipartRequest = new StandardMultipartHttpServletRequest(request);
        }
        return (multipartRequest != null ? multipartRequest.getFile(name) : null);
    }
    else if (isMultipartFileCollection(parameter)) {
        // 参数类型是Collection<MultipartFile>或者List<MultipartFile>
        if (multipartRequest == null && isMultipart) {
            // 如果是文件上传请求且multipartRequest == null，手动构造
            multipartRequest = new StandardMultipartHttpServletRequest(request);
        }
        return (multipartRequest != null ? multipartRequest.getFiles(name) : null);
    }
    else if (isMultipartFileArray(parameter)) {
        // 参数类型是MultipartFile[]
        if (multipartRequest == null && isMultipart) {
            // 如果是文件上传请求且multipartRequest == null，手动构造
            multipartRequest = new StandardMultipartHttpServletRequest(request);
        }
        if (multipartRequest != null) {
            List<MultipartFile> multipartFiles = multipartRequest.getFiles(name);
            return multipartFiles.toArray(new MultipartFile[0]);
        }
        else {
            return null;
        }
    }
    else if (Part.class == parameter.getNestedParameterType()) {
        return (isMultipart ? request.getPart(name): null);
    }
    else if (isPartCollection(parameter)) {
        return (isMultipart ? resolvePartList(request, name) : null);
    }
    else if (isPartArray(parameter)) {
        return (isMultipart ? resolvePartList(request, name).toArray(new Part[0]) : null);
    }
    else {
        return UNRESOLVABLE;
    }
}
```

**通过该方法可知，即便没有给`DispatcherServlet`配置`MultipartResolver`组件，也能正确获取值。**

### Spring Boot文件上传默认配置

理解了SpringMVC中文件上传的原理后，在Spring Boot中那便很简单了，因为Spring Boot也只不过是做了一些自动配置而已。Spring Boot对于文件上传的自动配置类为`MultipartAutoConfiguration`

```java
/**
 * 该配置在Servlet 3.0以上环境生效，因为MultipartConfigElement类是3.0版本才有
 * 可以通过spring.servlet.multipart.enabled=false关闭
 *
 * 该配置类非常简单，就是注册了Servlet3.0方式所需的两个类
 */
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Servlet.class, StandardServletMultipartResolver.class, MultipartConfigElement.class })
@ConditionalOnProperty(prefix = "spring.servlet.multipart", name = "enabled", matchIfMissing = true)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(MultipartProperties.class)
public class MultipartAutoConfiguration {

    private final MultipartProperties multipartProperties;

    public MultipartAutoConfiguration(MultipartProperties multipartProperties) {
        this.multipartProperties = multipartProperties;
    }

    /**
     * 如果容器中没有MultipartConfigElement类型的bean
     * 且没有CommonsMultipartResolver类型的bean(使用apache file upload)
     * 就不需要MultipartConfigElement了
     */
    @Bean
    @ConditionalOnMissingBean({ MultipartConfigElement.class, CommonsMultipartResolver.class })
    public MultipartConfigElement multipartConfigElement() {
        return this.multipartProperties.createMultipartConfig();
    }

    /**
     * 如果容器中没有MultipartResolver类型的bean
     */
    @Bean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
    @ConditionalOnMissingBean(MultipartResolver.class)
    public StandardServletMultipartResolver multipartResolver() {
        StandardServletMultipartResolver multipartResolver = new StandardServletMultipartResolver();
        multipartResolver.setResolveLazily(this.multipartProperties.isResolveLazily());
        return multipartResolver;
    }

}
```

```java
/**
 * 属性配置
 */
@ConfigurationProperties(prefix = "spring.servlet.multipart", ignoreUnknownFields = false)
public class MultipartProperties {

    /**
	 * 是否启用自动配置
	 */
    private boolean enabled = true;

    /**
	 * 临时路径，为null，则使用tomcat默认路径
	 */
    private String location;

    /**
	 * 默认1M
	 */
    private DataSize maxFileSize = DataSize.ofMegabytes(1);

    /**
	 * 默认10M
	 */
    private DataSize maxRequestSize = DataSize.ofMegabytes(10);

    /**
	 * 默认值，与MultipartConfigElement fileSizeThreshold默认值也是0
	 */
    private DataSize fileSizeThreshold = DataSize.ofBytes(0);

    public MultipartConfigElement createMultipartConfig() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
        map.from(this.fileSizeThreshold).to(factory::setFileSizeThreshold);
        map.from(this.location).whenHasText().to(factory::setLocation);
        map.from(this.maxRequestSize).to(factory::setMaxRequestSize);
        map.from(this.maxFileSize).to(factory::setMaxFileSize);
        return factory.createMultipartConfig();
    }
}
```

