> MySQL v8.0

mysqldump是MySQL的逻辑备份工具，将数据库中的数据转成SQL形式。

### 使用形式

可使用`mysqldump --help`来查看文档说明

基础选项

* -h 指定MySQL服务端ip
* -P 指定服务端端口
* -u 指定登录用户
* -p 指定登录密码

### 备份库

```bash
# 备份testdb库
mysqldump -h 127.0.0.1 -P 3306 -u root -p 123456 --single-transaction --set-gtid-purged=OFF --routines --triggers --add-drop-database --databases testdb > testdb.sql
```

* --single-transaction，开启一致性视图，其实就是在一个可重复图事务中执行dump操作，因为使用事务，因此只能针对InnoDB引擎。此时数据库可以正常进行读写操作，但是会阻塞DDL语句，因为DDL语句需要申请mdl写锁，而dump操作时存在mdl读锁，并且mdl锁在事务结束时才会释放，因此备份时数据库中接下来的DDL语句会被阻塞。
* --set-gtid-purged，可选的值为ON、OFF。当值为ON时，会在导出文件中记录GTID(全局事务ID，每个事务都会有一个GTID)，如果是用在备库中恢复数据，那么需要开启，保证主库和从库的GTID是一致的，如果不是建议关闭，恢复数据时由MySQL服务端重新生成。**注意GTID只有在服务端开启GTID模式才会存在，如果没有开启，使用ON选项会报错。**
* --routines，备份库中的存储过程以及函数。
* --triggers，备份表的触发器。
* --add-drop-database，在备份文件中添加drop database if exists testdb语句。
* --databases，后面跟要备份的数据库，多个库使用空格隔开。
* --all-databases，备份所有库。

### 备份表

```bash
# 备份testdb数据库中t1、t2表
mysqldump -h 127.0.0.1 -P 3306 -u root -p 123456 --single-transaction --set-gtid-purged=OFF testdb t1 t2 > bak.sql
```

备份表时不会输出create database语句

* --no-create-info，不会输出表的创建语句。
* --no-data，不会输出表数据。

### 更详细的说明可参考

[https://www.cnblogs.com/digdeep/p/4898622.html](https://www.cnblogs.com/digdeep/p/4898622.html)