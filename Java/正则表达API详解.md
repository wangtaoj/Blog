### 1. Pattern类

```java
public class PatternExample {
    /**
     * public static String quote(String s)
     * 返回指定字符串的字面值模式, 也就是说字符串序列中的元字符和转义序列将不具有特殊含义.
     * 会使用\Q \E包裹, \Q \E中间的字符是字面意思, 不具有特殊含义.
     * 
     * public static Pattern compile(String regex, int flag)
     * 编译给定正则表达式
     * flag: 匹配标志, 常用的如下解释
     * CASE_INSENSITIVE: 匹配时大小写不敏感.
     * MULTILINE: 启用多行模式, ^ $匹配行的开头和结尾, 而不是整个输入序列的的开头和结尾.
     * UNIX_LINES: 启用UNIX换行符, 在多行模式中使用^ $时只有\n被识别成终止符.
     * DOTALL: 在此模式中元字符.可以匹配任意字符, 包括换行符.
     * LITERAL: 启用模式的字面值意思, 模式中的元字符、转义字符不再具有特殊含义.
     * 
     * public static boolean matches(String regex, String s)
     * 判断整个输入序列是否与给定的模式匹配.
     * 底层调用Matcher实例的matchers方法
     */
    @Test
    public void quote() {
        String regex1 = Pattern.quote(".java");
        Assert.assertEquals("\\Q.java\\E", regex1);
        Matcher matcher = Pattern.compile(regex1).matcher("Effect.java");
        Assert.assertTrue(matcher.find());

        String regex2 = Pattern.quote("\\.java");
        Assert.assertEquals("\\Q\\.java\\E", regex2);
        Assert.assertTrue(Pattern.compile(regex2).matcher("Effect\\.java").find());
        Assert.assertFalse(Pattern.compile(regex2).matcher("Effect.java").find());
    }

    @Test
    public void matchesVsFind() {
        String s = "satat";
        String regex = "at";
        Assert.assertFalse(Pattern.matches(regex, s));
        Assert.assertTrue(Pattern.compile(regex).matcher(s).find());
        Assert.assertFalse(Pattern.compile("^at").matcher(s).find());
    }
```



### 2. Matcher

```java	
public class MatcherExample {

    /**
     * 匹配操作
     * public boolean lookingAt()
     * 尝试将整个输入序列的开始处与模式匹配, 对于输入序列的结尾处不做要求
     * 也就是说从左(必须是序列的开头)到右有一处与模式匹配, 便返回true.
     * 
     * public boolean matches()
     * 尝试将整个输入序列与模式匹配, 只有从头到尾完全与模式匹配才返回true.
     * 
     * public boolean find()
     * 只要整个输入序列或者子序列有一个与模式匹配, 便返回true.
     * find方法可用来寻找输入序列中所有与模式匹配的子序列.
     * 
     * 上述三个方法返回true时, 可以使用start, end, group方法获取详细信息
     * 返回false时或者没有调用过匹配方法, 调用start, end, group会抛出异常"No match available"
     * 
     * public int start()
     * 返回匹配序列在输入序列中的初始索引.
     * public int start(int group)
     * 返回给定组捕获的匹配序列在输入序列中的初始索引, 如果给定模式匹配成功,
     * 但是模式中的指定组没有匹配返回-1.
     * 如果模式中没有指定的组将抛出异常IndexOutOfBoundsException: No group ${group}
     * 
     * public int end()
     * 返回匹配序列中最后一个字符在输入序列中的索引+1.
     * public int end(int group)
     * 返回给定组捕获的匹配序列中最后一个字符在输入序列中索引+1,
     * 如果给定模式匹配成功, 但是模式中的指定组没有匹配返回-1.
     * 如果模式中没有指定的组将抛出异常IndexOutOfBoundsException: No group ${group}
     * 
     * public String group()
     * 返回匹配序列
     * public String group(int group)
     * 返回给定组捕获的匹配序列.
     * 如果给定模式匹配成功, 但是模式中的指定组没有匹配返回null.
     * 如果模式中没有指定的组将抛出异常IndexOutOfBoundsException: No group ${group}
     */
    @Test
    public void match() {
        String goal = "at sat cat mat";
        String regex = ".?at(a)?";
        Matcher matcher = Pattern.compile(regex).matcher(goal);
        // 从开始处匹配
        Assert.assertTrue(matcher.lookingAt());
        Assert.assertEquals(0, matcher.start());
        Assert.assertEquals(2, matcher.end());
        Assert.assertEquals("at", goal.substring(matcher.start(), matcher.end()));
        Assert.assertEquals("at", matcher.group());
        // 组1(a)没有匹配到, 返回-1
        Assert.assertEquals(-1, matcher.start(1));
        Assert.assertEquals(-1, matcher.end(1));
        Assert.assertNull(matcher.group(1));
    }

    /**
     * 修改或读取当前模式匹配输入序列的范围(区域), 默认是全部区域.
     * 查询时包头不包尾(subString()方法一样的含义)
     */
    @Test
    public void region() {
        String goal = "33abcd55";
        String regex = "abcd";
        Matcher matcher = Pattern.compile(regex).matcher(goal);
        // 查询开始索引
        Assert.assertEquals(0, matcher.regionStart());
        // 查询结尾索引
        Assert.assertEquals(goal.length(), matcher.regionEnd());
        Assert.assertFalse(matcher.matches());
        matcher.reset();
        // 调整区域, 相当于截取${goal}的abcd来匹配.
        matcher.region(2, 6);
        Assert.assertTrue(matcher.matches());
        Assert.assertEquals(2, matcher.regionStart());
        Assert.assertEquals(6, matcher.regionEnd());
    }

    @Test
    public void group() {
        String goal = "pig dog cat snake horse dog cat tiger monkey";
        String regex = "(dog)\\s*(cat)\\s*?";
        Matcher matcher = Pattern.compile(regex).matcher(goal);
        // 查找所有与模式匹配的串
        while (matcher.find()) {
            // result: dog cat
            System.out.println(matcher.group());
            // result: [dog, cat]
            for (int i = 1; i <= matcher.groupCount(); i++) {
                if (i == 1) {
                    System.out.print("[" + matcher.group(i));
                } else if (i == matcher.groupCount()) {
                    System.out.println(", " + matcher.group(i) + "]");
                } else {
                    System.out.print(", " + matcher.group(i));
                }
            }
            System.out.println("-----------------");
        }
    }

    /**
     * public Matcher appendReplacement(StringBuffer sb, String replacement)
     * 将目标字符串与模式匹配的部分替换成replacement, 将结果放到sb中
     * 此方法只会替换一处匹配的地方, 并且目标字符串的后续部分不会存放到sb中
     * 其中replacement可以使用反向引用来获取捕获组的内容
     * 反向引用规则(仅适用于Java):
     * 1. 使用捕获组编号,  $n  其中0 <= n <=groupCount()
     * 2. 使用捕获组名称,  ${name}  name以非数字开头
     * replacement只会涉及到两个字符的转义: 1. $  ->  \\$   2. \   ->   \\
     * 例子:
     * String gaol = "I like music", regex = "like, replacement = "love";
     * 那么sb = I love
     * 
     * StringBuffer appendTail(String sb)
     * 与appendReplacement搭配工作, 目标字符串剩下的内容添加到sb中
     * 接着上述例子调用appendTail方法, 那么sb = I love music
     * 
     * 基于以上两个方法便能实现replaceFirst, replaceAll两个方法
     */
    @Test
    public void replaceFist() {
        String goal = "I like music";
        String regex = "like";
        String relpacement = "love";
        Matcher matcher = Pattern.compile(regex).matcher(goal);
        StringBuffer sb = new StringBuffer();
        if (matcher.find()) {
            matcher.appendReplacement(sb, relpacement);
            Assert.assertEquals("I love", sb.toString());
            matcher.appendTail(sb);
            Assert.assertEquals("I love music", sb.toString());
        }
    }

    /**
     * 去掉dog 和 cat
     */
    @Test
    public void replaceAll() {
        String goal = "pig dog snake cat horse dog cat tiger monkey";
        String regex = "(dog\\s?)|(cat\\s?)";
        Matcher matcher = Pattern.compile(regex).matcher(goal);
        StringBuffer sb = new StringBuffer();
        boolean flag = false;
        while (matcher.find()) {
            matcher.appendReplacement(sb, "");
            flag = true;
        }
        if(flag) {
            sb = matcher.appendTail(sb);
            Assert.assertEquals("pig snake horse tiger monkey", sb.toString());
        }
    }

    /**
     * 去重
     */
    @Test
    public void removeDuplicate() {
        String goal = "aabbcddeefgg";
        String regex = "(\\w)\\1+";
        Matcher matcher = Pattern.compile(regex).matcher(goal);
        String result = matcher.replaceAll("$1");
        Assert.assertEquals("abcdefg", result);
    }
}

```



