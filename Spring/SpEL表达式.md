### 基本原则

- 字符串需要使用单引号引起来，否则表示的是一个变量。
- 变量前面没有#符号，表示取根对象的属性值。**如果根对象没有该属性，则会抛异常。**
- 变量前加#符号，表示访问上下文中通过`setVariable`方法设置的变量，其中`#root`是一个特殊值，表示根对象。**如果找不到该变量，不会抛异常，而是返回null。**
- 变量前加@符号，表示从上下文中的`BeanResolver`获取变量值，一个默认实现`BeanFactoryResolver`是从Spring容器中获取bean。**若无法找到对应的bean，则会抛异常。**
- `EvaluationContext`上下文中可以设置根对象，还可以通过`setVariable`方法设置其它变量，这些变量可以通过#符合+变量名引用。特别的，如果有设置`BeanResolver`实例，可以从Spring容器中获取bean实例，通过@符合+beanName引用。

### 使用例子

#### 简单使用

```java
@Test
public void simple() {
    ExpressionParser expressionParser = new SpelExpressionParser();
    // 简单计算
    Expression expression = expressionParser.parseExpression("3 + 4");
    Integer value = expression.getValue(Integer.class);
    Assert.assertEquals(Integer.valueOf(7), value);
    
    // 字面量, 字符串需要使用''引起来
    expression = expressionParser.parseExpression("'hello'");
    String literal = expression.getValue(String.class);
    Assert.assertEquals("hello", literal);
}
```

#### 访问静态成员

使用T(完全限定名)形式可访问类，如T(java.lang.String)。

```java
public static class Util {
    public static final String NAME = "hello";

    public static String get() {
        return "world";
    }
}

@Test
public void accessStaticMemberOfClass() {
    ExpressionParser expressionParser = new SpelExpressionParser();
    // T(完全限定名)访问类
    Expression expression = expressionParser.parseExpression("T(com.wangtao.springboottest.SpelTest.Util)");
    Assert.assertEquals(Util.class, expression.getValue());

    // 静态变量
    expression = expressionParser.parseExpression("T(com.wangtao.springboottest.SpelTest.Util).NAME");
    Assert.assertEquals("hello", expression.getValue());

    // 静态方法
    expression = expressionParser.parseExpression("T(com.wangtao.springboottest.SpelTest.Util).get()");
    Assert.assertEquals("world", expression.getValue());
}
```

#### 访问根对象

**注意，若访问根对象中一个不存在的属性，会抛异常。**

使用的pojo对象
```java
@AllArgsConstructor
@Data
public class SpelObj {

    private String name;

    private Integer age;
}
```

api使用
```java
/**
 * getValue方法若不明确指定EvaluationContext参数, 
 * 则会创建一个空StandardEvaluationContext实例
 */
@Test
public void accessRoot() {
    SpelObj root = new SpelObj("hello", 20);
    ExpressionParser expressionParser = new SpelExpressionParser();

    // 访问根对象
    Expression expression = expressionParser.parseExpression("#root");
    Assert.assertSame(root, expression.getValue(root));

    // 访问根对象属性
    expression = expressionParser.parseExpression("#root.name");
    String name = expression.getValue(root, String.class);
    Assert.assertEquals(name, "hello");
    
    // 访问根对象属性(简写形式, 不加#符号)
    expression = expressionParser.parseExpression("name");
    name = expression.getValue(root, String.class);
    Assert.assertEquals(name, "hello");

    // 调用方法, 此处显式创建EvaluationContext上下文对象, 并设置根对象
    expression = expressionParser.parseExpression("name.concat(' world')");
    EvaluationContext evaluationContext = new StandardEvaluationContext(root);
    String result = expression.getValue(evaluationContext, String.class);
    Assert.assertEquals("hello world", result);
}
```

#### 使用#访问变量

除了默认的root(根对象)，上下文对象中可以自定义其他变量，通过#符合访问。访问一个不存在的变量返回null，不会抛异常。

```java
@Test
public void accessVar() {
    ExpressionParser expressionParser = new SpelExpressionParser();

    // 使用#访问自定义变量
    Expression expression = expressionParser.parseExpression("#name.concat(':').concat(#age)");
    StandardEvaluationContext evaluationContext = new StandardEvaluationContext();
    // 添加自定义变量（非root变量）
    evaluationContext.setVariable("name", "zhangsan");
    evaluationContext.setVariable("age", 20);
    String result = expression.getValue(evaluationContext, String.class);
    Assert.assertEquals("zhangsan:20", result);
}
```

#### 使用@符合访问bean

需要往上下文中设置`BeanResolver`，用于获取bean。

```java
@Test
public void accessSpringBean() {
    // 创建容器
    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext();
    ac.registerBean("user", SpelObj.class, "user-bean", 20);
    ac.refresh();

    // 创建BeanResovler
    BeanResolver beanResolver = new BeanFactoryResolver(ac);

    ExpressionParser expressionParser = new SpelExpressionParser();
    // 通过@符号+beanName访问bean
    Expression expression = expressionParser.parseExpression("@user");
    StandardEvaluationContext evaluationContext = new StandardEvaluationContext(new SpelObj("hello", 20));
    // 设置BeanResovler
    evaluationContext.setBeanResolver(beanResolver);

    SpelObj spelObj = ac.getBean("user", SpelObj.class);
    Assert.assertSame(spelObj, expression.getValue(evaluationContext));
}
```

注: 如果bean是一个`FactoryBean`，需要访问这个FactoryBean本身，则使用&符合即可。

#### 模板表达式

在`@Value`注解中可以使用`@Value(#{name})`这种语法来给字段赋值，这其实用到了模板表达式，默认情况下，解析器会把给定的字符串都当做SpEL表达式来解析，而使用了模板之后，只会把模板部分当做SpEL表达式，其他部分只是一个普通字符串，原样返回。

要使用模板表达式，解析时需要使用以下方法。

```java
/**
 * @param expressionString 表达式字符串
 * @param context 解析上下文，指定模板前后缀
 */
Expression parseExpression(String expressionString, ParserContext context)
    throws ParseException;
```

一个通过子类实现为`TemplateParserContext`，可以自定义前后缀，默认为#{}。

使用例子如下

```java
@Test
public void template() {
    ExpressionParser expressionParser = new SpelExpressionParser();
    // 创建上下文
    StandardEvaluationContext evaluationContext = new StandardEvaluationContext();
    // 设置root变量
    evaluationContext.setRootObject(new SpelObj("lisi", 20));
    // 添加自定义变量（非root变量）
    evaluationContext.setVariable("name", "zhangsan");
    evaluationContext.setVariable("age", 24);

    // 取字面量(非模板部分字符串无需再加引号)
    Expression expression = expressionParser.parseExpression("hello #{'world'}", new TemplateParserContext());
    String result = expression.getValue(evaluationContext, String.class);
    Assert.assertEquals("hello world", result);

    // 取根对象属性
    expression = expressionParser.parseExpression("hello #{name}", new TemplateParserContext());
    result = expression.getValue(evaluationContext, String.class);
    Assert.assertEquals("hello lisi", result);

    // 取自定义变量属性
    expression = expressionParser.parseExpression("hello #{#name}", new TemplateParserContext());
    result = expression.getValue(evaluationContext, String.class);
    Assert.assertEquals("hello zhangsan", result);

    // 非模板部分字符串原样返回
    expression = expressionParser.parseExpression("#age #{#age}", new TemplateParserContext());
    result = expression.getValue(evaluationContext, String.class);
    Assert.assertEquals("#age 24", result);
}
```
### MethodBasedEvaluationContext上下文对象

`MethodBasedEvaluationContext`是`StandardEvaluationContext`的一个子类。它主要是把方法参数也加到了变量中，使得用户可以直接通过#+参数名来获取值。常常用于解析注解中的SpEL表达式。如Cache模块中`@Cacheable`注解中的key属性就支持SpEL表达式。

可以使用如下方式来访问方法参数
- 直接通过#+参数名，如#name
- 通过#+内置参数名a，#a0访问第一个参数，#a1访问第二个参数
- 通过#+内置参数名p，#p0访问第一个参数，#p1访问第二个参数

**其中a0和p0是等价的，只是设置两个参数前缀而已。而通过参数名称来访问需要`ParameterNameDiscoverer`的支持，默认情况下，java编译后通过反射是拿不到真实的方法参数名称的，需要带上-parameters参数编译才行，不过Spring还另外基于ASM的方式解析字节码文件，获取字节码的本地方法表来获取方法真实参数。`DefaultParameterNameDiscoverer`实现类同时使用上面所说的两种方式来获取方法参数名。**

下面来看下使用例子

```java
interface Samer {
    boolean isSame(String name, Integer age);
}

@Test
public void accessMethodArg() {
    // 该对象可以重复使用并且线程安全
    ExpressionParser expressionParser = new SpelExpressionParser();

    Samer proxy = (Samer) Proxy.newProxyInstance(
            SpelTest.class.getClassLoader(),
            new Class<?>[]{Samer.class},
            new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) {
                    // root对象
                    Object root = new Object();
                    // 创建EvaluationContext
                    MethodBasedEvaluationContext evaluationContext = new MethodBasedEvaluationContext(
                            root,
                            method,
                            args,
                            new DefaultParameterNameDiscoverer()
                    );
                    // 通过#+方法参数名
                    Expression expression = expressionParser.parseExpression("#name");
                    Assert.assertEquals("hello", expression.getValue(evaluationContext));

                    // 通过# + 内置的变量名+下标a0
                    expression = expressionParser.parseExpression("#a0");
                    Assert.assertEquals("hello", expression.getValue(evaluationContext));

                    // 通过# + 内置的变量名+下标p0
                    expression = expressionParser.parseExpression("#p1");
                    Assert.assertEquals(20, expression.getValue(evaluationContext));

                    if (method.getName().equals("isSame")) {
                        return Objects.equals("hello", args[0]) && Objects.equals(20, args[1]);
                    }
                    throw new UnsupportedOperationException(method.getName());
                }
            }
    );
    Assert.assertTrue(proxy.isSame("hello", 20));
}
```

该类的实现原理也非常简单，只是重写了`lookupVariable`方法，即寻找自定义变量的逻辑。

```java
@Override
@Nullable
public Object lookupVariable(String name) {
    // 先查找下变量存不存在
    Object variable = super.lookupVariable(name);
    if (variable != null) {
        return variable;
    }
    if (!this.argumentsLoaded) {
        // 把方法参数放到variables变量表中
        lazyLoadArguments();
        this.argumentsLoaded = true;
        // 再次获取
        variable = super.lookupVariable(name);
    }
    return variable;
}
```

```java
protected void lazyLoadArguments() {
    // Shortcut if no args need to be loaded
    if (ObjectUtils.isEmpty(this.arguments)) {
        return;
    }

    // 获取参数名
    String[] paramNames = this.parameterNameDiscoverer.getParameterNames(this.method);
    int paramCount = (paramNames != null ? paramNames.length : this.method.getParameterCount());
    int argsCount = this.arguments.length;

    for (int i = 0; i < paramCount; i++) {
        Object value = null;
        if (argsCount > paramCount && i == paramCount - 1) {
            // Expose remaining arguments as vararg array for last parameter
            value = Arrays.copyOfRange(this.arguments, i, argsCount);
        }
        else if (argsCount > i) {
            // Actual argument found - otherwise left as null
            value = this.arguments[i];
        }
        // a0、a1等
        setVariable("a" + i, value);
        // p0、p1等
        setVariable("p" + i, value);
        // 参数名
        if (paramNames != null && paramNames[i] != null) {
            setVariable(paramNames[i], value);
        }
    }
}
```