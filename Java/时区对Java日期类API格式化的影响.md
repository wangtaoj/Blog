### 时区基本概念

时区（Time Zone）是指地球上的一个地区与格林尼治标准时间（GMT）或协调世界时（UTC）之间的时间差异。由于地球自转的原因，不同的地理位置会有不同的时间。时区的划分使得世界各地能够更合理地安排时间，保持同步。

**UTC（协调世界时）**: UTC 是一种标准时间，它没有受到地球自转速度变化影响，是全世界时间标准的基础。所有的时区都是相对于 UTC 来定义的，例如 UTC+8 表示比 UTC 早 8 个小时的时间。

**GMT（格林尼治标准时间）**: GMT 是一种基于地球自转和太阳位置的时间标准，虽然它和 UTC 时间非常接近，但由于 GMT 是一种天文时间，存在微小的变化。

**时区偏移（Offset）**: 时区通常会以 UTC 偏移的方式表示，例如 UTC+0:00、UTC+05:30、UTC-8 :00等等。UTC+8:00 的时区意味着这个时区比 UTC 早 8 个小时，比如中国的北京时间。

**夏令时（Daylight Saving Time, DST）**: 某些国家和地区会使用夏令时，在夏季时将时间拨快一小时，以充分利用日照时间。夏令时的使用会导致某些地方的时间偏移在一年中有变化。

### Java中的时区API

#### JDK8之前的TimeZone

```java
/**
 * 获取所有可用的zoneId
 */
public static synchronized String[] getAvailableIDs();

/**
 * 获取所在地默认时区，国内便是Asia/Shanghai，东八区GMT+08:00
 */
public static TimeZone getDefault();

/**
 * 根据ID获取时区，注意如果是一个不支持的ID，则会降级到GMT时区，而不是报错，因此有一个坑。
 * 格式:
 * 标准的地区名如Asia/Shanghai
 * UTC、GMT
 * 偏移量: GMT+08:00 (注意不支持UTC+08:00), 然后偏移量只支持到分钟级别
 * 这个API支持单独的UTC，但是不支持UTC开头的偏移量
 * 因此如果你使用了UTC+08:00，则会降级到GMT时区，产生诡异现象。
 */
public static synchronized TimeZone getTimeZone(String ID);
```

**以地区命名的id和偏移量区别**

比如: Asia/Shanghai和GMT+08:00，对于大部分情况而言是对等的，然而由于夏令时的存在，使得时钟拨快了一个小时，使用Asia/Shanghai的话，会根据规则判断当前时间是不是在夏令时区间，从而计算出的偏移量是+09:00，而使用GMT+08:00则是固定的。因此使用Asia/Shanghai这个地区命名的不用用户考虑这种繁琐的夏令时规则。

#### JDK8新增的ZoneId、ZoneOffset

ZoneId

```java
/**
 * 获取所有可用的zoneId
 */
public static Set<String> getAvailableZoneIds();

/**
 * 获取所在地默认时区，国内便是Asia/Shanghai，东八区UTC+08:00
 */
public static ZoneId systemDefault();

/**
 * 根据ID获取时区，这个方法做了改进，不支持的参数会直接报错而不是降级到UTC，非常棒。
 * 格式:
 * 标准的地区名如Asia/Shanghai
 * Z、UTC、GMT。这三个等价，相当于UTC+0
 * 偏移量: UTC+08:00 GMT+08:00
 * 这个API同时支持UTC和GMT前缀
 */
public static ZoneId of(String zoneId);
```

ZoneOffset

代表UTC时区的偏移量，没有了地区名写法。ZoneOffset继承了ZoneId

```java
/**
 * 覆盖了ZoneId的of方法
 * 格式: 不能写前缀，只能写偏移量
 * +08:00
 * -08:00
 * 特别的这个偏移量可以精确到秒
 * +08:00:30
 */
public static ZoneOffset of(String offsetId);

/**
 * 根据小时的偏移量
 * ofHours(8)相当于+08:00
 * ofHours(-8)相当于-08:00
 */
public static ZoneOffset ofHours(int hours);
```

ZoneOffset不仅仅提供了ofHours方法，诸如ofHoursMinutes等方法，一眼就懂。

#### 新老时区API互转

TimeZone提供了两个方法

```java
/**
 * 静态方法
 */
public static TimeZone getTimeZone(ZoneId zoneId);

/**
 * 成员方法
 */
public ZoneId toZoneId();
```

### 纪元毫秒(epochMillis)

**纪元毫秒**是从计算机的纪元时间（Epoch time）开始经过的毫秒数。纪元时间通常被定义为 **1970年1月1日 00:00:00 UTC**。纪元毫秒通常用来表示计算机系统中日期和时间的时间戳。

**注意是UTC时区的1970-01-01 00:00:00**

给定一个毫秒数，它代表非UTC时区的具体哪个时间，必须要指定时区才有意义。

比如1000这个纪元毫秒数，即1秒

等同于UTC时区的1970-01-01 00:00:01

也等同于UTC+8:00时区的1970-01-01 08:00:01

**想反的，给定一个时间格式字符串，必须指定时区，才能转成纪元毫秒数**

### java.util.Date

`Date`类本身存储的就是纪元毫秒数，`getTime`方法就可以获取到这个存储的纪元毫秒数。

但是`Date`类中的其它方法，诸如`getYear`、`getDate`、`getSeconds`都是配合默认时区(中国是Asia/Shanghai)的一个时间来获取的。

这些getXXX方法都已经被标注过时了。

`Date`类中的`toString`方法也一样，作为显示用，内部调用了这些getXXX方法。

使用DateFormat格式化成字符时，得到的结果具体是哪个时间，取决于DateFormat中设置的timeZone，若没有指定，则默认值`TimeZone.getDefault()`。

**总结: Date类存储的值是纪元毫秒数，与时区无关，不过内部中的一些方法返回值使用了默认时区来计算。**

### java.sql.Timestamp

`java.sql.Timestamp`继承了`java.util.Date`，然后多加了一个`nanos`属性用来存储纳秒，值为[0 , 999,999,999]中的一个。也就是说增加了精度，其余和`java.util.Date`一样。

### Instant

`Instant`为JDK8新增的一个类，可以用来代替`java.util.Date`，内部存储的是纪元纳秒数，精度会更高。可以翻译为时刻。

内部有两个属性

**seconds**：距离纪元时间的秒数

**nanos**：纳秒数，[0, 999,999,999]，用于提高精度。

创建方法

```java
/**
 * 根据毫秒数创建
 */
public static Instant ofEpochMilli(long epochMilli);

/**
 * 一个参数是秒
 * 第二个参数是纳秒数，可以为正也可以为负，用来调整最终的一个结果
 * 注意这两个参数不等同于Instant中的连个成员变量seconds、nanos
 */
public static Instant ofEpochSecond(long epochSecond, long nanoAdjustment);

/**
 * 当前时间对应的时刻
 */
public static Instant now();
```

基本使用

```java
@Test
public void testInstant() {
    // 该毫秒数对应UTC时间为2024-09-02 13:45:49
    long millis = 1725284749641L;
    Instant instant = Instant.ofEpochMilli(millis);
    // 2024-09-02T13:45:49.641Z
    System.out.println(instant);
    instant = Instant.parse("2024-09-02T13:45:49.641Z");
    // 1725284749641
    System.out.println(instant.toEpochMilli());
    // 配合时区转成ZonedDateTime, 2024-09-02T21:45:49.641+08:00[Asia/Shanghai]，可以看到时间加了8小时
    System.out.println(instant.atZone(ZoneId.of("Asia/Shanghai")));
    // 配合时区转成OffsetDateTime，2024-09-02T21:45:49.641+08:00
    System.out.println(instant.atOffset(ZoneOffset.ofHours(8)));
}
```

### java.util.Date格式化

```java
@Test
public void testFormatDate() {
    // 该毫秒数对应UTC时间为2024-09-02 13:45:49
    long millis = 1725284749641L;
    Date date = new Date(millis);

    String pattern = "yyyy-MM-dd HH:mm:ss";
    /*
     * SimpleDateFormat线程不安全, 方法会修改自身
     * timeZone默认为TimeZone.getDefault()，国内即Asia/Shanghai
     */
    DateFormat dateFormat = new SimpleDateFormat(pattern);

    DateFormat dateFormatGmt9 = new SimpleDateFormat(pattern);
    dateFormatGmt9.setTimeZone(TimeZone.getTimeZone("GMT+09:00"));

    // 东八区: 2024-09-02 21:45:49
    System.out.println(dateFormat.format(date));
    // 东九区: 2024-09-02 22:45:49
    System.out.println(dateFormatGmt9.format(date));
}
```

### JSR310时间类格式化

```java
@Test
public void testFormatJsr310Date() {
    /*
     * 下面4个都是代表东八区的2024-09-02 12:45:49
     * LocalDateTime没有时区概念
     * Instant + 东八区便能得到时间024-09-02 12:45:49
     */
    LocalDateTime localDateTime = LocalDateTime.of(2024,9, 2, 21, 45, 49);
    ZonedDateTime zonedDateTime = ZonedDateTime.of(localDateTime, ZoneId.of("Asia/Shanghai"));
    OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, ZoneOffset.ofHours(8));
    Instant instant = Instant.ofEpochMilli(1725284749641L);

    // 2024-09-02T21:45:49
    System.out.println(localDateTime);
    // 2024-09-02T21:45:49+08:00[Asia/Shanghai]
    System.out.println(zonedDateTime);
    // 2024-09-02T21:45:49+08:00
    System.out.println(offsetDateTime);
    /*
     * 2024-09-02T13:45:49.641Z
     * Instant toString方法展示的是UTC时区的时间, 末尾的Z代表UTC
     * 那么对应于东八区的时间就是2024-09-02T21:45:49.64
     */
    System.out.println(instant);

    /*
     * DateTimeFormatter线程安全，和String一样，为不可变对象
     * 与SimpleDateFormat不同，时区字段默认是null，没有默认值
     */
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    // 2024-09-02 21:45:49
    System.out.println(formatter.format(localDateTime));
    // 2024-09-02 21:45:49
    System.out.println(formatter.format(zonedDateTime));
    // 2024-09-02 21:45:49
    System.out.println(formatter.format(offsetDateTime));
    // 会报错, 格式化成年月日这样形式, Formatter必须要指定时区
    Assertions.assertThrowsExactly(UnsupportedTemporalTypeException.class, () -> formatter.format(instant));

    // 指定东九区时区
    DateTimeFormatter utc9Formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")
            .withZone(ZoneOffset.ofHours(9));

    // 2024-09-02 21:45:49
    System.out.println(utc9Formatter.format(localDateTime));
    // 2024-09-02 22:45:49
    System.out.println(utc9Formatter.format(zonedDateTime));
    // 2024-09-02 22:45:49
    System.out.println(utc9Formatter.format(offsetDateTime));
    // 2024-09-02 22:45:49
    System.out.println(utc9Formatter.format(instant));
}
```

总结:

`DateTimeFormatter`是和`String`一样的不可变对象，线程安全。因此修改时一定记得要返回，否则无效哦。

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
formatter = formatter.withZone(ZoneOffset.ofHours(9));
```

`DateTimeFormatter`中的时区属性默认值为null，而不是系统所在时区，和`SimpleDateFormat`不同。

`LocalDateTime`格式化时不会进行时区转换，因为本身没有时区概念。

`ZonedDateTime`、`OffsetDateTime`格式化时，只有当`DateTimeFormatter`明确指定了时区才会进行转换。`ZonedDateTime`、`OffsetDateTime`携带的时区为源，`DateTimeFormatter`中的时区为目标时区。

`Instant`格式化时，`DateTimeFormatter`必须指定时区，否则会报错。毕竟纪元毫秒数(纳秒数)搭配时区才能转成时间。

### 时间新旧API互转

两套API互转的桥梁便是`Instant`，因为`Instant`和`Date`存储的都是到纪元时间的毫秒数(纳秒数)。

当然了由于精度不同，`Instant`转`Date`会丢失一点精度，因为`Date`只能存储到毫秒级别。若不想丢精度，可以转成`java.sql.Timsstamp`，它也是可以精确到纳秒的。

java.util.Date.java

```java
/**
 * Instant -> Date
 */
public static Date from(Instant instant) {
    try {
        return new Date(instant.toEpochMilli());
    } catch (ArithmeticException ex) {
        throw new IllegalArgumentException(ex);
    }
}

/**
 * Date -> Instant
 */
public Instant toInstant() {
    return Instant.ofEpochMilli(getTime());
}
```

有了`Instant`之后，那么`Instant` + `ZoneId` = `ZonedDateTime`，`Instant` + `ZoneOffset` = `OffsetDateTime`。

Instant.java

```java
/**
 * Instant -> OffsetDateTime
 */
public OffsetDateTime atOffset(ZoneOffset offset) {
    return OffsetDateTime.ofInstant(this, offset);
}

/**
 * Instant -> ZonedDateTime
 */
public ZonedDateTime atZone(ZoneId zone) {
    return ZonedDateTime.ofInstant(this, zone);
}
```

而`OffsetDateTime`或者`ZonedDateTime`转`Instant`也很简单，调用它们的`toInstant`方法即可

**一句话总结：Instant+时区便可以转化成对应时区的时间。**

`OfffsetDateTime`或者`ZoneDateTime`去掉时区便是`LocalDateTime`了，相反`LocalDateTime`加上时区就是`OfffsetDateTime`或者`ZoneDateTime`。这里就是单纯的去掉时区或者加上时区，不会发生任何时间转换，因为`LocalDateTime`没有时区概念，就是本地时间。

```java
// ZoneDateTime -> LocalDateTime
LocalDateTime localDateTime = zonedDateTime.toLocalDateTime();
// OffsetDateTime -> LocalDateTime
LocalDateTime localDateTime = offsetDateTime.toLocalDateTime();
// LocalDateTime -> ZoneDateTime
ZoneDateTime zoneDateTime = localDateTime.atZone(ZoneId.of("UTC"));
// LocalDateTime -> OffsetDateTime
OffsetDateTime offsetDateTime = localDateTime.atOffset(ZoneOffset.UTC);
```

最后来一个`java.util.Date` -> `LocalDateTime`，`LocalDateTime` -> `java.util.Date`

```java
// 先转成东八区时间，然后去掉时区换成本地时间
LocalDateTime localDateTime = date.toInstant().atZone(ZoneId.systemDefault()).toLocalDateTime();
// 先将本地时间+时区，然后转成Instant，再到Date
Date date = Date.from(localDateTime.atZone(ZoneId.systemDefault()).toInstant())
```

