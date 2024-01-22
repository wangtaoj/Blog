> canal-1.1.7

### 简介

canal主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费。主要原理是基于MySQL的主从同步机制，将自己伪装成MySQL slave，从而可以接收到MySQL master推送的binary log，进而解析二进制日志捕获到变更数据。

### 搭建canal服务

#### 数据库准备

首先MySQL需要开启binlog，并配置binlog_format为ROW模式。

查看binlog是否开启

```mysql
-- log_bin = ON 表示已开启, OFF表示关闭
show variables like '%log_bin%';
-- 查看二进制log格式
show variables like '%binlog_format%';
-- 查看server_id, canal需要和master的不一样
show variables like '%server_id%';
```

若没有开启，则在my.cnf配置文件中**[mysqld]**部分加入如下配置

```properties
# 开启 binlog
log-bin=mysql-bin
# 选择 ROW 模式
binlog-format=ROW
# 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
server_id=1
```

创建一个拥有MySQL slave权限的用户

```mysql
# 使用mysql_native_password模式, 不然启动报错
CREATE USER 'canal'@'%' IDENTIFIED WITH mysql_native_password BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

#### docker canal安装

1. docker-compose.yml

   ```yaml
   services:
     canal-server:
       image: canal/canal-server:v1.1.7
       container_name: canal-server
       ports:
         - "11111:11111"
       volumes:
         - ./conf/example:/home/admin/canal-server/conf/example
         - ./conf/canal.properties:/home/admin/canal-server/conf/canal.properties
         - ./logs:/home/admin/canal-server/logs
   ```
   
2. 修改配置文件conf/example/instance.properties

   ```properties
   # MySQL master地址, host.docker.internal为docker容器所在的宿主机, 当然也可以明确指定IP
   canal.instance.master.address=host.docker.internal:3306
   # 拥有MySQL slave权限的用户
   canal.instance.dbUsername=canal
   canal.instance.dbPassword=canal
   # 监听canaldb数据库的所有表, 多个规则用逗号分割
   canal.instance.filter.regex=canaldb\\..*
   
   # 投递bin log消息到MQ的主题名称
   canal.mq.topic=canal-topic
   ```

3. 修改配置文件conf/canal.properties

   ```properties
   # tcp, kafka, rocketMQ, rabbitMQ, pulsarMQ, 默认tcp
   canal.serverMode = rocketMQ
   
   # 投递bin log消息到mq会进行序列化
   canal.mq.flatMessage = true
   # 生产者组
   rocketmq.producer.group = canal-test
   rocketmq.enable.message.trace = false
   rocketmq.customized.trace.topic =
   rocketmq.namespace =
   # rocketmq namesrv
   rocketmq.namesrv.addr = host.docker.internal:9876
   rocketmq.retry.times.when.send.failed = 0
   rocketmq.vip.channel.enabled = false
   rocketmq.tag = 
   ```

经过上述配置，canal服务端会监听canaldb数据库的所有表，每当监听到binlog日志时，会投递到RocketMQ中，并且消息的主题是canal-topic。

### 客户端例子

该例子需要配置`canal.serverMode = tcp`

1. 引入依赖

   ```xml
   <dependency>
     <groupId>com.alibaba.otter</groupId>
     <artifactId>canal.client</artifactId>
     <version>1.1.7</version>
   </dependency>
   
   <dependency>
     <groupId>com.alibaba.otter</groupId>
     <artifactId>canal.protocol</artifactId>
     <version>1.1.7</version>
   </dependency>
   ```

2. 代码示例(官网)

   ```java
   package com.wangtao.canal;
   
   import com.alibaba.otter.canal.client.CanalConnector;
   import com.alibaba.otter.canal.client.CanalConnectors;
   import com.alibaba.otter.canal.common.utils.AddressUtils;
   import com.alibaba.otter.canal.protocol.CanalEntry;
   import com.alibaba.otter.canal.protocol.Message;
   
   import java.net.InetSocketAddress;
   import java.util.List;
   
   import static com.alibaba.otter.canal.protocol.CanalEntry.*;
   
   public class SimpleCanalClient {
   
       public static void main(String[] args) {
           // 创建链接
           CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress(AddressUtils.getHostIp(),
                   11111), "example", "", "");
           int batchSize = 1000;
           int emptyCount = 0;
           try {
               connector.connect();
               connector.subscribe("canaldb\\..*");
               connector.rollback();
               int totalEmptyCount = 120;
               while (emptyCount < totalEmptyCount) {
                   Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据
                   long batchId = message.getId();
                   int size = message.getEntries().size();
                   if (batchId == -1 || size == 0) {
                       emptyCount++;
                       System.out.println("empty count : " + emptyCount);
                       try {
                           Thread.sleep(3000);
                       } catch (InterruptedException e) {
                       }
                   } else {
                       emptyCount = 0;
                       // System.out.printf("message[batchId=%s,size=%s] \n", batchId, size);
                       printEntry(message.getEntries());
                   }
   
                   connector.ack(batchId); // 提交确认
                   // connector.rollback(batchId); // 处理失败, 回滚数据
               }
   
               System.out.println("empty too many times, exit");
           } finally {
               connector.disconnect();
           }
       }
   
       private static void printEntry(List<Entry> entrys) {
           for (Entry entry : entrys) {
               if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {
                   continue;
               }
   
               RowChange rowChage = null;
               try {
                   rowChage = RowChange.parseFrom(entry.getStoreValue());
               } catch (Exception e) {
                   throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),
                           e);
               }
   
               EventType eventType = rowChage.getEventType();
               System.out.println(String.format("================&gt; binlog[%s:%s] , name[%s,%s] , eventType : %s",
                       entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),
                       entry.getHeader().getSchemaName(), entry.getHeader().getTableName(),
                       eventType));
   
               for (RowData rowData : rowChage.getRowDatasList()) {
                   if (eventType == CanalEntry.EventType.DELETE) {
                       printColumn(rowData.getBeforeColumnsList());
                   } else if (eventType == EventType.INSERT) {
                       printColumn(rowData.getAfterColumnsList());
                   } else {
                       System.out.println("-------&gt; before");
                       printColumn(rowData.getBeforeColumnsList());
                       System.out.println("-------&gt; after");
                       printColumn(rowData.getAfterColumnsList());
                   }
               }
           }
       }
   
       private static void printColumn(List<Column> columns) {
           for (Column column : columns) {
               System.out.println(column.getName() + " : " + column.getValue() + "    update=" + column.getUpdated());
           }
       }
   }
   
   ```

   ### RocketMQ消费者例子
   
   该例子主要实现的功能是同步canaldb数据库的user表的数据到ES中。
   
   [代码位置](https://github.com/wangtaoj/canal-learning)
   
   一些说明
   
   * 由于配置了`canal.mq.flatMessage = true`，发送到mq中的消息格式如下所示
   
   ```json
   {
     "data": [
       {
         "id": "1",
         "username": "wt1"
       }
     ],
     "database": "canaldb",
     "es": 1680267742000,
     "id": 2,
     "isDdl": false,
     "mysqlType": {
       "id": "int",
       "username": "varchar(255)"
     },
     "old": [
       {
         "username": "wt2"
       }
     ],
     "pkNames": [
       "id"
     ],
     "sql": "",
     "sqlType": {
       "id": 4,
       "username": 12
     },
     "table": "user",
     "ts": 1680267742743,
     "type": "UPDATE"
   }
   ```
   
   * 消息主题为canal-topic
   * 消息消费主入口类`CanalListener`
   * 同步数据到ES逻辑类`UserCanalProcessor`