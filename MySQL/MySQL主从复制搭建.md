### 前言
> MySQL 8.0.36

[官方文档](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-howto.html)

### docker部署

基于gtid模式搭建一主一从架构。

#### 配置

master配置文件内容如下(my.cnf)

```properties
[mysqld]
# 主从复制关键配置
server_id = 1
gtid_mode = ON
enforce_gtid_consistency = ON

# 事务隔离级别
transaction_isolation = READ-COMMITTED

# 开启慢查询日志
slow_query_log = 1
# 超过2秒算慢查询
long_query_time = 2
# 慢查询日志输出到文件
log_output = FILE

# 2G
innodb_buffer_pool_size = 2147483648
```

slave配置文件内容如下(my.cnf)

```properties
[mysqld]
# 主从复制关键配置
server_id = 2
gtid_mode = ON
enforce_gtid_consistency = ON
# 禁止写操作, 不过对root用户无效
read_only = 1

# 事务隔离级别
transaction_isolation = READ-COMMITTED

# 开启慢查询日志
slow_query_log = 1
# 超过2秒算慢查询
long_query_time = 2
# 慢查询日志输出到文件
log_output = FILE

# 2G
innodb_buffer_pool_size = 2147483648
```

* 主从复制模式是基于bin log实现的，因此一定要开启这个，不过8.0的版本默认是开启的，因此可以不用管这项了。
* server_id：唯一标识，每个MySQL实例必须不一样。
* gtid_mode：因为基于gtid模式搭建，所以必须开启。
* enforce_gtid_consistency：表示所有事务都不能违反gtid事务一致性。
* read_only：从库开启只读。

#### docker-compose.yml

```yaml
services:
  mysql-master:
    image: mysql:8.0.36
    container_name: mysql-master
    ports:
      - 4306:3306
    volumes:
      # /etc/mysql/conf.d目录中以cnf结尾的文件都会被加载为配置文件
      - ./master/etc:/etc/mysql/conf.d
      - ./master/data/:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=123456
    networks:
      mysql:
        aliases:
          - mysql-master

  mysql-slave:
    image: mysql:8.0.36
    container_name: mysql-slave
    ports:
      - 4307:3306
    volumes:
      - ./slave/etc:/etc/mysql/conf.d
      - ./slave/data/:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=123456
    depends_on:
      - mysql-master
    networks:
      mysql:
        aliases:
          - mysql-slave

networks:
  mysql:
    name: mysql_network
    driver: bridge
```

把上面的配置文件放到对应的master/etc以及slave/etc目录下。

使用下面命令运行容器

```bash
docker compose up -d
```

#### 创建账号

在主库上创建用于复制bin log的账号

```mysql
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
flush PRIVILEGES;
```

注意使用mysql_native_password而不是默认的caching_sha2_password，不然从库连接主库时需要安全认证，麻烦。

#### 从库连接主库

到目前为止，启动的两个实例还没有关联关系，需要使用`CHANGE REPLICATION SOURCE TO`建立关联关系。

注意：该命令要在从库上执行

```mysql
CHANGE REPLICATION SOURCE TO SOURCE_HOST = 'mysql-master', SOURCE_PORT = 3306, SOURCE_USER = 'repl', SOURCE_PASSWORD = '123456', SOURCE_AUTO_POSITION = 1;
```

* SOURCE_HOST：主库ip，由于docker部署设置了网络，用名称替代。
* SOURCE_PORT：主库端口。
* SOURCE_USER：用于复制bin log的账号，即上面创建的账号。
* SOURCE_PASSWORD：用于复制bin log的账号密码。
* SOURCE_AUTO_POSITION：使用基于gtid复制的自动定位功能，1开启 0关闭。

注意：字符串需要加引号，不然报语法错误。

#### 开启复制需要的线程

```mysql
START REPLICA;
```

会开启两个线程，一个io线程(用于从master复制bin-log，并存储到自己relay-log)，一个statement线程，用于从relay-log解析语句重放生成数据。

**注意：重新启动容器不需要再执行这个命令，会自动开启这两个线程。**

#### 验证

```mysql
-- 查看复制情况，注意不需要末尾的分号，需要在从库上执行
SHOW REPLICA STATUS \G
```

结果

```tex
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: mysql-master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000012
          Read_Master_Log_Pos: 1157
               Relay_Log_File: 6c90ab4ad2c4-relay-bin.000003
                Relay_Log_Pos: 1367
        Relay_Master_Log_File: binlog.000012
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1157
              Relay_Log_Space: 2544
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: b5ec3160-fc01-11ee-a27f-0242ac180002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: b5ec3160-fc01-11ee-a27f-0242ac180002:1-7
            Executed_Gtid_Set: b5ec3160-fc01-11ee-a27f-0242ac180002:1-7
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 0
            Network_Namespace: 
```

Slave_IO_Running: Yes
Slave_SQL_Running: Yes

表示两个线程启动完毕。

### 如何查看主从数据复制完毕

通过在**从库**上执行`SHOW REPLICA STATUS;`

* 查看Slave_SQL_Running_State的值是不是Replica has read all relay log; waiting for more updates，表明全部同步完毕。

* 查看Master_Log_File、Read_Master_Log_Pos，表示获取到的主库上的binlog文件以及位置；

  然后在**主库**上执行`SHOW MASTER STATUS`，可以看到主库上最新的binlog文件以及位置，比较是否相等即可。

