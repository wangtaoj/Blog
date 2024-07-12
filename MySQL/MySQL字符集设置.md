> MySQL v8.0

从MySQL 8.0开始，默认的字符集为utf8mb4，排序规则为utf8mb4_0900_ai_ci。

**大前提: **

* 字符集只针对字符串类型，varchar、char、text等。
* 数据库以及表的字符集最终都会落实到字段中，如果字段没有指定字符集，则会取表的默认字符集和排序规则。
* 如果在DDL语句中只指定了字符集，那么排序规则便是该字符集默认的那个排序规则。
* 如果在DDL语句中只指定了排序规则，那么字符集便是该排序规则所属的那个字符集。

### 1. 查询MySQL可用的字符集

```mysql
-- 列出所有可用的字符集, 并且可以看到每个字符集默认的排序规则
show character set;
-- 带查询条件
show character set where charset = 'utf8mb4';
```

### 2. 查询MySQL可用的排序规则

```mysql
-- 列出所有的排序规则
show collation;
-- 查询指定字符集下的所有排序规则
show collation where charset = 'utf8mb4';
```

**注意排序规则是和字符集相关联的。**

### 3. 创建数据库时指定库的默认字符集和排序规则

```mysql
-- default可省略
create database testdb default character set utf8mb4 collate utf8mb4_bin;
```

### 4. 表的默认字符集和排序规则

创建表时指定，若不指定则会使用数据库的字符集和排序规则

```mysql
create table t(
  id bigint unsigned not null auto_increment primary key comment '主键',
  name varchar(20) comment '名字'
) engine=InnoDB default character set utf8mb4 collate utf8mb4_bin comment '测试表';
```

修改已存在表的默认字符集和排序规则

```mysql
-- 会修改所有类型为字符串的列的字符集
alter table t convert to character set utf8mb4 collate utf8mb4_0900_ai_ci;

-- 只会影响后续新增的字段，不会影响已有的字段
alter table t character set utf8mb4 collate utf8mb4_0900_ai_ci;
```

查看表的字符集和排序规则

```mysql
-- 查看建表语句
show create table t;
```

### 5. 字段的字符集和排序规则

修改字段的字符集和排序规则

```mysql
-- 使用修改表字段语句，但是字段定义得写全
alter table t modify column name varchar(20) character set utf8mb4 collate utf8mb4_bin comment '名字';
```

查看字段的字符集和排序规则

```mysql
show full column from t;
```

### 6.关于排序规则的说明

已经有了字符集为啥还需要排序规则，排序规则会影响字段的比较以及排序等。

拿utf8mb4_0900_ai_ci来说，它是大小写不敏感的，不区分音调的。

比如表t中有个name字段，有一条zhangsan的记录。

那么`select * from t where name = 'zhangsan' `或者`select * from t where name = 'ZHANGSAN'`都是可以查到这条记录的。

而utf8mb4_bin是二进制形式，比较时需要内容一模一样才行。

**另外比较重要的是，两表join时，字段的排序规则不一致时会导致索引失效。**

ai表示accent insensitivity，也就是不区分音调

as表示accent sensitivity，即区分音调

ci表示case insensitivity，也就是不区分大小写

cs表示case sensitivity，即区分大小写