### Redis主从搭建

redis主从模式分为1个主节点，多个从节点。默认情况下主节点支持读写操作，从节点不支持写操作，主节点数据发生变化后会同步给从节点。

主从模式主要缺点是不能负载均衡，受限单机容量。并且如果主节点挂了不会自动从从节点中选举新的主节点。

由于机器有限，下面主要模拟一台机器搭建主从模式，一主一丛

新建6379 6380文件夹，复制redis.conf到这两个文件下进行修改

6379/redis.conf (master)

```properties
bind 0.0.0.0
port 6379
dbfilename dump_6379.rdb
requirepass 123456
```

6380/redis.conf (slave)

```reStructuredText
bind 0.0.0.0
port 6380
dbfilename dump_6380.rdb
requirepass 123456
# 指定主节点
slaveof 127.0.0.1 6379
masterauth 123456
```

启动与连接

```ba
./redis-server ../RedisMasterSlave/6379/redis.conf
./redis-server ../RedisMasterSlave/6380/redis.conf

./redis-cli -h 127.0.0.1 -p 6379
./redis-cli -h 127.0.0.1 -p 6380
```

### Spring Boot集成Redis

