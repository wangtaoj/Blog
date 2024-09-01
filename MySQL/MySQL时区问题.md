### 时间类型

* TIMESTAMP：占用4个字节(有符号整数)，即最大值为2,147,483,647 (1 << 31 - 1)，转成时间就是2038-01-19 03:14:07。这个时间是UTC时区。存储的是该时间距离1970-01-01 00:00:00的**秒数**。

* DATETIME：占用8个字节，与时区无关，不会进行转换。

**两个重点**

**TIMESTAMP存储单位是秒，代表UTC时区时间到1970-01-01 00:00:00的秒数**

注: TIMESTAMP后面增加了精确位，加了小数位，相应占用的字节会变多，用来存储小数秒，这样可以精确到毫秒、微秒。

| 类型                         | 存储字节数 |
| ---------------------------- | ---------- |
| TIMESTAMP(默认，无小数）     | 4 字节     |
| TIMESTAMP(1) 或 TIMESTAMP(2) | 5 字节     |
| TIMESTAMP(3) 或 TIMESTAMP(4) | 6 字节     |
| TIMESTAMP(5) 或 TIMESTAMP(6) | 7 字节     |

### 系统参数

* time_zone：该变量用于控制TIMESTAMP列的时区转换，当然也会影响一些时间函数，比如now()。默认值为SYSTEM，此时取决于系统变量`system_time_zone`的值。代表服务端所在的时区，作用域分为全局和会话级别，可以由客户端动态修改。
* system_time_zone：MySQL服务器所在时区，MySQL服务启动时继承于机器中的时区，国内默认是CST。只有全局作用域

time_zone参数对TIMESTAMP字段的影响

比如当前time_zone设置为UTC+8，即北京时间东八区。

插入时，将时间转换成UTC时间，即减去8小时，然后存储UTC对应时间到1970年的秒数。

查询时，将秒数转换成UTC时间，然后根据时区进行转换，即加上8小时得到结果。

**注意: 这部分转换动作都是MySQL服务端发生的，和JDBC驱动或者客户端没有关系，依赖于会话的time_zone系统变量，JDBC驱动查询时实际得到的结果是MySQL服务端已经转换完成后的时间，并且这个结果不是秒数，而是时间字符串。这里容易误解成JDBC驱动或者客户端干的转换。与客户端有关的是可以设置time_zone时区，告诉服务端如何转换。**

关于CST时区的说明

**CST** 是一个多义的时区缩写，可能代表不同的时间区域，因此在使用时需要明确指明具体的时区。以下是几个不同的 CST 的解释：

1. **China Standard Time (CST, UTC+8)**：即中国标准时间，常用在中国大陆、台湾、香港、澳门地区，等同于北京时间（UTC+8）。
2. **Central Standard Time (CST, UTC-6)**：即美国和加拿大中部标准时间，主要用于美国中部时间区，标准时间为 UTC-6，夏令时为 CDT (Central Daylight Time, UTC-5)。
3. **Cuba Standard Time (CST, UTC-5)**：古巴标准时间，与东部时间相同，标准时间为 UTC-5，夏令时为 CDT (Cuba Daylight Time, UTC-4)。
4. **Central Standard Time (Australia) (CST, UTC+9:30)**：澳大利亚中部标准时间（适用于南澳大利亚州、北领地等），标准时间为 UTC+9:30，夏令时为 CDT (UTC+10:30)。

由于 CST 可能表示不同的时间区域，因此在使用 CST 时，应根据上下文清楚说明是哪个国家或地区的 CST。

**在MySQL中CST代表中国标准时间(UTC+8)。**

### MySQL驱动

JDBC驱动有3个重要参数

**首先明确一点的是，对于TIMESTAMP类型，MySQL服务端发送给JDBC驱动的是一个已经根据time_zone系统变量转换后的时间，而不是到1970-01-01 00:00:00的秒数。**

* connectionTimeZone，时区，如果不设置，将会获取当前会话的`time_zone`变量值。格式为ZoneId，形如+8:00，或者名字Asia/Shanghai，别名serverTimezone。
* forceConnectionTimeZoneToSession，默认为false，是否需要设置会话的`time_zone`变量。
* preserveInstants，默认为true，保留时区信息，这个是jdbc驱动自己的行为。

当`forceConnectionTimeZoneToSession`=false且`preserveInstants`=false时，设置`connectionTimeZone`参数没有任何作用。

当`forceConnectionTimeZoneToSession`=false时，对于TIMESTAMP类型不会产生任何时区影响，因为不会设置`time_zone`变量的值，`time_zone`取决于服务端设置的全局作用域，不受`connectionTimeZone`影响。并且时区转换动作由MySQL服务端自动完成的，然后再发给JDBC驱动。

当preserveInstants=true时的行为

**存储时**

若列的类型为TIMESTAMP，Java类型是`OffsetDateTime`、`ZonedDateTime`、`java.sql.Timestamp`则会进行时区转换。

其中前面两个好理解因为它们都是带有时区信息的，而`java.sql.Timestamp`本质上是一个时间戳(即TIMESTAMP类型语义，只不多精确度变成了纳秒)，代表的是到1970-01-01 00:00:00纳秒数，因此可以配合时区转换成一个对应时区的时间。`java.util.Date`在JDBC中都是转换成`java.sql.Timestamp`或者`java.sql.Date`或者`java.sql.Time`来处理的。

比如OffsetDateTime字段的值为2024-09-01 17:06:00 UTC+8，connectionTimeZone为UTC+9，则发送给MySQL服务端的是

2024-09-01 18:06:00，加一个小时。

**检索时**

若列的类型是TIMESTAMP、DATETIME、字符串类型，Java类型是`OffsetDateTime`、`ZonedDateTime`、`java.sql.Timestamp`，

则会保留时区信息。

MySQL发送给JDBC驱动的时间是2024-09-01 18:06:00，JDBC驱动再根据connectionTimeZone时区信息合成最终的结果。

即2024-09-01 18:06:00 UTC+9

若preserveInstants=false，则上面存储转换时使用的时区TimeZone.getDefault()，检索时保留的时区也是TimeZone.getDefault()

### 最佳实践

connectionTimeZone务必要和mysql服务端的time_zone系统变量一致，否则会导致时间可能不一致。

forceConnectionTimeZoneToSession设置成false，默认值就是false，不要去修改会话的time_zone系统变量。

preserveInstants设置成true，默认值就是true。

基于最佳实践参数的例子说明:

假设MySQL服务器time_zone系统变量是UTC+9

客户端JVM所在时区为UTC+8

**对于列是TIMESTAMP类型，Java类型是ZonedDateTime**

保存时

时间是2024-09-01 17:06:00 UTC+8

由于preserveInstants=true，会将这个时间转成connectionTimeZone指定时区的时间2024-09-01 18:06:00 UTC+9

然后发送给MySQL服务端，发送的值为2024-09-01 18:06:00，是没有时区信息的，需要结合MySQL的time_zone系统变量，然后time_zone也是UTC+9，相当于MySQL存的就是2024-09-01 18:06:00 UTC+9

查询时

MySQL发送给JDBC驱动的时间是2024-09-01 18:06:00，由于preserveInstants=true，会保留时区(connectionTimeZone)UTC+9，而不是用JVM所在的默认时区UTC+8，因此最终得到值便是2024-09-01 18:06:00 UTC+9，相当于2024-09-01 17:06:00 UTC+8。时间是一致的。

**因此务必要保证connectionTimeZone和mysql服务端的time_zone系统变量一致**

**对于列是TIMESTAMP类型，Java类型是LocalDateTime。**

由于保存时，对于LocalDateTime不会进行转换，因为LocalDateTime没有时区信息，发送给MySQL的值为2024-09-01 17:06:00 ，L相当于保存的是2024-09-01 17:06:00 UTC+9

查询时，MySQL发送给JDBC驱动的时间是2024-09-01 17:06:00，驱动对于LocalDateTime类型不会进行转换，这就是最终结果。时间是一致的。

### 源码参考

保存：`ValueEncoder`、`ZonedDateTimeValueEncoder`、`LocalDateTimeValueEncoder`、`SqlTimestampValueEncoder`

检索：`MysqlTextValueDecoder`、`LocalDateTimeValueFactory`、`ZonedDateTimeValueFactory`、`SqlTimestampValueFactory`
