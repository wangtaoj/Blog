### 时间类型

* TIMESTAMP：占用4个字节(有符号整数)，即最大值为2,147,483,647 (1 << 31 - 1)，转成时间就是2038-01-19 03:14:07

​        这个时间是UTC时区。存储的是该时间距离1970-01-01 00:00:01的**秒数**。

​        **两个重点: 存储单位是秒，时区固定是UTC，比如东八区的时间，先转成UTC时区，即减去8个小时后再存储。**

​        **查询时，先根据秒数转换成UTC时区的时间，再根据客户端设置的时区转换，比如东八区，UTC->东八区，加上8小时。**

* DATETIME：占用8个字节，与时区无关，不会进行转换。

注: TIMESTAMP后面增加了精确位，加了小数位，相应占用的字节会变多，用来存储小数秒，这样可以精确到毫秒、微秒。

| 类型                         | 存储字节数 |
| ---------------------------- | ---------- |
| TIMESTAMP(默认，无小数）     | 4 字节     |
| TIMESTAMP(1) 或 TIMESTAMP(2) | 5 字节     |
| TIMESTAMP(3) 或 TIMESTAMP(4) | 6 字节     |
| TIMESTAMP(5) 或 TIMESTAMP(6) | 7 字节     |

### 系统参数

* time_zone：该变量用于初始化每个连接的客户端的时区，若客户端没有显示指定时区的话，默认值为SYSTEM，此时取决于系统变量`system_time_zone`的值。作用域分为全局和会话级别
* system_time_zone：MySQL服务器所在时区，MySQL服务启动时继承于机器中的时区，国内默认是CST。

time_zone参数对TIMESTAMP字段的影响

比如当前time_zone设置为UTC+8，即北京时间东八区。

插入时，将时间转换成UTC时间，减去8小时，然后存储UTC对应时间到1970年的秒数。

查询时，将秒数转换成UTC时间，然后发现时区是UTC+8，则再加上8小时得到结果。

**注意: 这部分转换动作都是MySQL服务端发生的，和JDBC驱动没关系，依赖于会话的time_zone系统变量，JDBC驱动查询时实际得到的结果是时间而不是秒数，这里容易误解成JDBC驱动干的转换。**

关于CST时区的说明

**CST** 是一个多义的时区缩写，可能代表不同的时间区域，因此在使用时需要明确指明具体的时区。以下是几个不同的 CST 的解释：

1. **China Standard Time (CST, UTC+8)**：即中国标准时间，常用在中国大陆、台湾、香港、澳门地区，等同于北京时间（UTC+8）。
2. **Central Standard Time (CST, UTC-6)**：即美国和加拿大中部标准时间，主要用于美国中部时间区，标准时间为 UTC-6，夏令时为 CDT (Central Daylight Time, UTC-5)。
3. **Cuba Standard Time (CST, UTC-5)**：古巴标准时间，与东部时间相同，标准时间为 UTC-5，夏令时为 CDT (Cuba Daylight Time, UTC-4)。
4. **Central Standard Time (Australia) (CST, UTC+9:30)**：澳大利亚中部标准时间（适用于南澳大利亚州、北领地等），标准时间为 UTC+9:30，夏令时为 CDT (UTC+10:30)。

由于 CST 可能表示不同的时间区域，因此在使用 CST 时，应根据上下文清楚说明是哪个国家或地区的 CST。

**在MySQL中CST代表中国标准时间(UTC+8)。**

### MySQL驱动

JDBC驱动也可以明确设置时区，这样子就会设置每个会话的时区，覆盖掉`time_zone`的全局设置，**time_zone**作用域分为会话级别和全局级别。

```java
String url = "jdbc:mysql://localhost:3306/testdb?connectionTimeZone=Asia/Shanghai&forceConnectionTimeZoneToSession=true";
```

* connectionTimeZone，时区，如果不设置，将会获取当前会话的`time_zone`变量值。格式为ZoneId，形如+8:00，或者名字Asia/Shanghai，别名serverTimezone。
* forceConnectionTimeZoneToSession，默认为false，是否需要设置会话的`time_zone`变量。
* preserveInstants，默认为true，转换时区信息，这个是jdbc驱动自己的行为。

当`forceConnectionTimeZoneToSession`=false且`preserveInstants`=false时，设置`connectionTimeZone`参数没有任何作用。

当`forceConnectionTimeZoneToSession`=false时，对于TIMESTAMP类型不会产生任何时区影响，因为不会设置`time_zone`变量的值，`time_zone`取决于服务端设置的全局作用域，不受`connectionTimeZone`影响。并且时区转换动作由MySQL服务端自动完成的，然后再发给JDBC驱动。

当`preserveInstants` =true时，如果Java中的类型是`OffsetDateTime`或者`ZonedDateTime`，即带有时区的类型，会将MySQL发过来的时间(若列的类型是TIMESTAMP，已经是根据time_zone转换完的时间，没有时区信息了)带上connectionTimeZone时区，也就相当于发生了时区转换即time_zone时区 -》connectionTimeZone时区。若`preserveInstants` =false时，时区使用JVM的系统时区`TimeZone.default()`

preserveInstants=true的生效条件

存储时，列的类型必须为TIMESTAMP。

检索时，列的类型是TIMESTAMP、DATETIME、字符串类型并且接收的Java类型是`OffsetDateTime`、`ZonedDateTime`、`java.sql.Timestamp`，需要两个条件同时满足。

其它情况通通使用JVM的系统时区`TimeZone.default()`

