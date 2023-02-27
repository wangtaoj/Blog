> seata版本: 1.6.1

### 官网

[官方文档](https://seata.io/zh-cn/index.html)

[下载链接](https://seata.io/zh-cn/blog/download.html)

### seata server安装事项

seata server即seata术语中的TC(事务协调者)，用于维护全局和分支事务的状态，驱动全局事务提交或回滚。

搭建seata server总体上注意下面这些点

* 注册中心，registry.type，支持file(本地文件，默认)、redis、nacos、euraka、consul等注册中心
* 配置中心，config.type，支持file(本地文件，默认)、redis、nacos、apollo、consul等配置中心
* 事务会话信息存储方式，store.mode，支持file、db、redis，一般都会使用db方式，需要初始化一些表

搭建时可以参考官网文档中运维指南->部署章节，[资源目录介绍](https://seata.io/zh-cn/docs/ops/deploy-guide-beginner.html)

在下载的资源包中script中存在以下目录

- client

  存放client端sql脚本 (包含 undo_log表) ，参数配置，1.6.1版本没看到该目录

- config-center

  各个配置中心参数导入脚本，config.txt(包含server和client，原名nacos-config.txt)为通用参数文件

- server

  server端数据库脚本 (包含 lock_table、branch_table 与 global_table) 及各个容器配置

### seata server启动

存储模式使用db，注册中心、配置中心都使用nacos

1. 建表

   将server目录下的数据库文件执行，主要是global_table、branch_table、lock_table、distributed_lock四张表。

   在mysql中先创建seatadb数据库，然后建表。**该版本的建表语句需要使用mysql 8.0以上版本**

2. 修改存储模式store.mode为db

   seata server就是一个spring boot应用，配置可在application.yml文件进行修改。启动脚本修改了程序的application.yml文件位置，不需要到jar包内部去改，路径为conf/application.yml，此外同目录下application.example.yml中附带额外配置，可以进行复制。修改配置如下所示

   ```yaml
   seata:
     store:
       # support: file 、 db 、 redis
       mode: db
       # 这些属性可以从nacos配置中心读取(可不写)
       db:
         datasource: druid
         db-type: mysql
         driver-class-name: com.mysql.cj.jdbc.Driver
         # 数据库名使用第一步的数据库
         url: jdbc:mysql://127.0.0.1:3306/seatadb?rewriteBatchedStatements=true&serverTimezone=Asia/Shanghai
         user: root
         password: 123456
         min-conn: 10
         max-conn: 100
         global-table: global_table
         branch-table: branch_table
         lock-table: lock_table
         distributed-lock-table: distributed_lock
         query-limit: 1000
         max-wait: 5000
   ```

3. 修改注册中心registry.type为nacos

   配置文件同上面

   ```yaml
   seata:
     registry:
       # support: nacos, eureka, redis, zk, consul, etcd3, sofa
       type: nacos
       nacos:
         application: seata-server
         server-addr: 127.0.0.1:8848
         group: SEATA_GROUP
         namespace: c7123a1e-d438-412c-af72-cdf3ab18a547
         cluster: default
   ```
   
4. 修改配置中心config.type为nacos

   配置文件同上面

   ```yaml
   seata:
     config:
       # support: nacos, consul, apollo, zk, etcd3
       type: nacos
       nacos:
         server-addr: 127.0.0.1:8848
         namespace: c7123a1e-d438-412c-af72-cdf3ab18a547
         group: SEATA_GROUP
         data-id: seata.properties
   ```

5. 同步配置到nacos配置中心

   上面将配置中心修改成了nacos，因此要将配置推送到nacos配置中心去，下载的资源包已经提供了同步脚本，路径是/script/config-center，配置文件为config.txt。进入nacos目录，执行脚本即可。

   修改文件中store.db相关属性，如上面第二点所示，修改后的如下所示

   ```properties
   #For details about configuration items, see https://seata.io/zh-cn/docs/user/configurations.html
   #Transport configuration, for client and server
   transport.type=TCP
   transport.server=NIO
   transport.heartbeat=true
   transport.enableTmClientBatchSendRequest=false
   transport.enableRmClientBatchSendRequest=true
   transport.enableTcServerBatchSendResponse=false
   transport.rpcRmRequestTimeout=30000
   transport.rpcTmRequestTimeout=30000
   transport.rpcTcRequestTimeout=30000
   transport.threadFactory.bossThreadPrefix=NettyBoss
   transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
   transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
   transport.threadFactory.shareBossWorker=false
   transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
   transport.threadFactory.clientSelectorThreadSize=1
   transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
   transport.threadFactory.bossThreadSize=1
   transport.threadFactory.workerThreadSize=default
   transport.shutdown.wait=3
   transport.serialization=seata
   transport.compressor=none
   
   #Transaction routing rules configuration, only for the client
   service.vgroupMapping.default_tx_group=default
   
   service.enableDegrade=false
   service.disableGlobalTransaction=false
   
   #Transaction rule configuration, only for the client
   client.rm.asyncCommitBufferLimit=10000
   client.rm.lock.retryInterval=10
   client.rm.lock.retryTimes=30
   client.rm.lock.retryPolicyBranchRollbackOnConflict=true
   client.rm.reportRetryCount=5
   client.rm.tableMetaCheckEnable=true
   client.rm.tableMetaCheckerInterval=60000
   client.rm.sqlParserType=druid
   client.rm.reportSuccessEnable=false
   client.rm.sagaBranchRegisterEnable=false
   client.rm.sagaJsonParser=fastjson
   client.rm.tccActionInterceptorOrder=-2147482648
   client.tm.commitRetryCount=5
   client.tm.rollbackRetryCount=5
   client.tm.defaultGlobalTransactionTimeout=60000
   client.tm.degradeCheck=false
   client.tm.degradeCheckAllowTimes=10
   client.tm.degradeCheckPeriod=2000
   client.tm.interceptorOrder=-2147482648
   client.undo.dataValidation=true
   client.undo.logSerialization=jackson
   client.undo.onlyCareUpdateColumns=true
   server.undo.logSaveDays=7
   server.undo.logDeletePeriod=86400000
   client.undo.logTable=undo_log
   client.undo.compress.enable=true
   client.undo.compress.type=zip
   client.undo.compress.threshold=64k
   #For TCC transaction mode
   tcc.fence.logTableName=tcc_fence_log
   tcc.fence.cleanPeriod=1h
   
   #Log rule configuration, for client and server
   log.exceptionRate=100
   
   #Transaction storage configuration, only for the server. The file, db, and redis configuration values are optional.
   store.mode=db
   #Used for password encryption
   #store.publicKey=
   
   
   #These configurations are required if the `store mode` is `db`. If `store.mode,store.lock.mode,store.session.mode` are not equal to `db`, you can remove the configuration block.
   store.db.datasource=druid
   store.db.dbType=mysql
   store.db.driverClassName=com.mysql.cj.jdbc.Driver
   store.db.url=jdbc:mysql://127.0.0.1:3306/seatadb?rewriteBatchedStatements=true&serverTimezone=Asia/Shanghai
   store.db.user=root
   store.db.password=123456
   store.db.minConn=5
   store.db.maxConn=30
   store.db.globalTable=global_table
   store.db.branchTable=branch_table
   store.db.distributedLockTable=distributed_lock
   store.db.queryLimit=100
   store.db.lockTable=lock_table
   store.db.maxWait=5000
   
   
   #Transaction rule configuration, only for the server
   server.recovery.committingRetryPeriod=1000
   server.recovery.asynCommittingRetryPeriod=1000
   server.recovery.rollbackingRetryPeriod=1000
   server.recovery.timeoutRetryPeriod=1000
   server.maxCommitRetryTimeout=-1
   server.maxRollbackRetryTimeout=-1
   server.rollbackRetryTimeoutUnlockEnable=false
   server.distributedLockExpireTime=10000
   server.xaerNotaRetryTimeout=60000
   server.session.branchAsyncQueueSize=5000
   server.session.enableBranchAsyncRemove=false
   server.enableParallelRequestHandle=false
   
   #Metrics configuration, only for the server
   metrics.enabled=false
   metrics.registryType=compact
   metrics.exporterList=prometheus
   metrics.exporterPrometheusPort=9898
   ```

   执行脚本

   ```bash
   sh nacos-config.sh -h localhost -p 8848 -g SEATA_GROUP -t c7123a1e-d438-412c-af72-cdf3ab18a547
   ```

   执行完发现在nacos配置中心，每一项属性key作为一个dataId存在，但是1.6.1版本的seata已经支持从一个dataId中读取所有属性了，因此手动在nacos中新建一个`data-id=seata.properties`、`group=SEATA_GROUP`、`namespace=c7123a1e-d438-412c-af72-cdf3ab18a547`的配置文件，然后将config.txt文件里的内容复制进去就好了。

6. 启动seata server

   ```bash
   # -h 指定IP，该IP将会被注册到注册中心, 多网卡机器可以使用此参数指定确切的IP
   # -p seata server监听端口, 默认为seata应用程序启动端口 + 100, 应用程序启动的端口在application.yml中可修改, 默认7091
   # seata应用程序启动的端口7091是seata控制台页面使用的端口
   # 本例子nacos注册的IP和端口为192.168.2.102:8091
   sh seata-server.sh -h 192.168.2.102 -p 8091
   ```

​       启动日志在seata安装目录/logs下

### seata server docker部署

[官方文档](https://seata.io/zh-cn/docs/ops/deploy-by-docker-compose.html)

使用docker部署和手动部署没啥区别

1. 下载镜像

   ```bash
   docker pull seataio/seata-server:1.6.1
   ```

2. 获取启动的默认配置文件，用于修改，启动一个临时的容器

   ```bash
   # 启动临时容器
   docker run --name seata-server -p 8091:8091 -p 7091:7091 seataio/seata-server:1.6.1
   # 拷贝默认配置文件(将resources目录拷贝到宿主机指定目录下)
   docker cp seata-server:/seata-server/resources /Users/wangtao/Developer/docker-compose/seata-1.6.1
   # 删除临时容器(先停止)
   docker rm seata-server
   ```

3. 修改resources目录下的application.yml文件（同手动部署）

   主要注意下nacos的ip以及namespace，store.mode=db。docker容器内可以直接根据宿主机的ip来访问宿主机的应用。

   宿主机无网络环境下可通过虚拟域名形式访问`host.docker.internal`

   ```yaml
   server:
     port: 7091
   
   spring:
     application:
       name: seata-server
   
   logging:
     config: classpath:logback-spring.xml
     file:
       path: ${user.home}/logs/seata
     extend:
       logstash-appender:
         destination: 127.0.0.1:4560
       kafka-appender:
         bootstrap-servers: 127.0.0.1:9092
         topic: logback_to_logstash
   
   console:
     user:
       username: seata
       password: seata
   
   seata:
     config:
       # support: nacos, consul, apollo, zk, etcd3
       type: nacos
       nacos:
         server-addr: 192.168.2.102:8848
         namespace: 1a309f28-39f8-4229-97ee-5bbdd446c3f4
         group: SEATA_GROUP
         data-id: seata.properties
     registry:
       # support: nacos, eureka, redis, zk, consul, etcd3, sofa
       type: nacos
       nacos:
         application: seata-server
         server-addr: 192.168.2.102:8848
         group: SEATA_GROUP
         namespace: 1a309f28-39f8-4229-97ee-5bbdd446c3f4
         cluster: default
     store:
       # support: file 、 db 、 redis
       mode: db
   #  server:
   #    service-port: 8091 #If not configured, the default is '${server.port} + 1000'
     security:
       secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
       tokenValidityInMilliseconds: 1800000
       ignore:
         urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login
   ```

4. 同步配置到nacos配置中心，namespace、group、dataId得一致

   主要修改store.db相关内容

   ```properties
   #For details about configuration items, see https://seata.io/zh-cn/docs/user/configurations.html
   #Transport configuration, for client and server
   transport.type=TCP
   transport.server=NIO
   transport.heartbeat=true
   transport.enableTmClientBatchSendRequest=false
   transport.enableRmClientBatchSendRequest=true
   transport.enableTcServerBatchSendResponse=false
   transport.rpcRmRequestTimeout=30000
   transport.rpcTmRequestTimeout=30000
   transport.rpcTcRequestTimeout=30000
   transport.threadFactory.bossThreadPrefix=NettyBoss
   transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
   transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
   transport.threadFactory.shareBossWorker=false
   transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
   transport.threadFactory.clientSelectorThreadSize=1
   transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
   transport.threadFactory.bossThreadSize=1
   transport.threadFactory.workerThreadSize=default
   transport.shutdown.wait=3
   transport.serialization=seata
   transport.compressor=none
   
   #Transaction routing rules configuration, only for the client
   service.vgroupMapping.default_tx_group=default
   
   service.enableDegrade=false
   service.disableGlobalTransaction=false
   
   #Transaction rule configuration, only for the client
   client.rm.asyncCommitBufferLimit=10000
   client.rm.lock.retryInterval=10
   client.rm.lock.retryTimes=30
   client.rm.lock.retryPolicyBranchRollbackOnConflict=true
   client.rm.reportRetryCount=5
   client.rm.tableMetaCheckEnable=true
   client.rm.tableMetaCheckerInterval=60000
   client.rm.sqlParserType=druid
   client.rm.reportSuccessEnable=false
   client.rm.sagaBranchRegisterEnable=false
   client.rm.sagaJsonParser=fastjson
   client.rm.tccActionInterceptorOrder=-2147482648
   client.tm.commitRetryCount=5
   client.tm.rollbackRetryCount=5
   client.tm.defaultGlobalTransactionTimeout=60000
   client.tm.degradeCheck=false
   client.tm.degradeCheckAllowTimes=10
   client.tm.degradeCheckPeriod=2000
   client.tm.interceptorOrder=-2147482648
   client.undo.dataValidation=true
   client.undo.logSerialization=jackson
   client.undo.onlyCareUpdateColumns=true
   server.undo.logSaveDays=7
   server.undo.logDeletePeriod=86400000
   client.undo.logTable=undo_log
   client.undo.compress.enable=true
   client.undo.compress.type=zip
   client.undo.compress.threshold=64k
   #For TCC transaction mode
   tcc.fence.logTableName=tcc_fence_log
   tcc.fence.cleanPeriod=1h
   
   #Log rule configuration, for client and server
   log.exceptionRate=100
   
   #Transaction storage configuration, only for the server. The file, db, and redis configuration values are optional.
   store.mode=db
   #Used for password encryption
   #store.publicKey=
   
   #These configurations are required if the `store mode` is `db`. If `store.mode,store.lock.mode,store.session.mode` are not equal to `db`, you can remove the configuration block.
   store.db.datasource=druid
   store.db.dbType=mysql
   store.db.driverClassName=com.mysql.cj.jdbc.Driver
   store.db.url=jdbc:mysql://192.168.2.102:3306/seatadb?rewriteBatchedStatements=true&serverTimezone=Asia/Shanghai
   store.db.user=root
   store.db.password=123456
   store.db.minConn=5
   store.db.maxConn=30
   store.db.globalTable=global_table
   store.db.branchTable=branch_table
   store.db.distributedLockTable=distributed_lock
   store.db.queryLimit=100
   store.db.lockTable=lock_table
   store.db.maxWait=5000
   
   #Transaction rule configuration, only for the server
   server.recovery.committingRetryPeriod=1000
   server.recovery.asynCommittingRetryPeriod=1000
   server.recovery.rollbackingRetryPeriod=1000
   server.recovery.timeoutRetryPeriod=1000
   server.maxCommitRetryTimeout=-1
   server.maxRollbackRetryTimeout=-1
   server.rollbackRetryTimeoutUnlockEnable=false
   server.distributedLockExpireTime=10000
   server.xaerNotaRetryTimeout=60000
   server.session.branchAsyncQueueSize=5000
   server.session.enableBranchAsyncRemove=false
   server.enableParallelRequestHandle=false
   
   #Metrics configuration, only for the server
   metrics.enabled=false
   metrics.registryType=compact
   metrics.exporterList=prometheus
   metrics.exporterPrometheusPort=9898
   ```

5. 建表(与手工部署一致)

6. docker command or docker-compose.yml

   ```bash
   docker run -d --name seata-server \
           -p 8091:8091 \
           -p 7091:7091 \
           -e SEATA_IP=192.168.2.102 \
           -e EATA_PORT=8091 \
           -e STORE_MODE=db \
           -v /Users/wangtao/Developer/docker-compose/seata-1.6.1/resources:/seata-server/resources  \
           seataio/seata-server:1.6.1
   ```

   ```yaml
   services:
     seata-server:
       image: seataio/seata-server:1.6.1
       container_name: seata-server
       ports:
         - "7091:7091"
         - "8091:8091"
       environment:
         # 可不用, 会从配置中心读取, 加上这个万一配置中心配置有问题加载不到配置时会报错可提前发现问题
         - STORE_MODE=db
         # 以SEATA_IP作为host注册到注册中心，使用宿主机ip
         - SEATA_IP=192.168.2.102
         - SEATA_PORT=8091
       volumes:
         - ./resources:/seata-server/resources
   ```

   

### 客户端整合

1. **每个**客户端数据库新建以下表

   ```sql
   CREATE TABLE `undo_log`
   (
       `id`            bigint       NOT NULL AUTO_INCREMENT,
       `branch_id`     bigint       NOT NULL,
       `xid`           varchar(100) NOT NULL,
       `context`       varchar(128) NOT NULL,
       `rollback_info` longblob     NOT NULL,
       `log_status`    int          NOT NULL,
       `log_created`   datetime     NOT NULL,
       `log_modified`  datetime     NOT NULL,
       `ext`           varchar(100) DEFAULT NULL,
       PRIMARY KEY (`id`),
       UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
   ) ENGINE = InnoDB;
   ```

2. **每个**客户端应用引入依赖

   ```xml
   <!-- 此版本和seata服务端保持一致 -->
   <dependency>
     <groupId>io.seata</groupId>
     <artifactId>seata-spring-boot-starter</artifactId>
     <version>1.6.1</version>
   </dependency>
   
   <!-- 和Spring-Cloud-Alibaba版本保持一致即可 -->
   <dependency>
     <groupId>com.alibaba.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
     <version>${spring-cloud-alibaba.version}</version>
     <exclusions>
       <exclusion>
         <groupId>io.seata</groupId>
         <artifactId>seata-spring-boot-starter</artifactId>
       </exclusion>
     </exclusions>
   </dependency>
   ```

3. **每一**个客户端bootstrap.yml文件增加seata配置

   ```yaml
   # seata服务注册信息, 用于下面引用, 避免重复粘贴
   meta:
     seata:
       nacos:
         server-addr: 127.0.0.1:8848
         namespace: c7123a1e-d438-412c-af72-cdf3ab18a547
         group: SEATA_GROUP
   
   seata:
     enabled: true
     application-id: ${spring.application.name}
     # 事务分组, 后面细讲
     tx-service-group: default_tx_group
     registry:
       type: nacos
       nacos:
         # 应与seata-server实际注册的服务名一致
         application: seata-server
         server-addr: ${meta.seata.nacos.server-addr}
         namespace: ${meta.seata.nacos.namespace}
         group: ${meta.seata.nacos.group}
     config:
       type: nacos
       nacos:
         server-addr: ${meta.seata.nacos.server-addr}
         namespace: ${meta.seata.nacos.namespace}
         group: ${meta.seata.nacos.group}
         data-id: seata.properties
   ```

4. 使用

   只需要在需要分布式事务的方法前加上注解即可`@GlobalTransactional`

### 事务分组概念

参照[官方文档事务分组](https://seata.io/zh-cn/docs/user/txgroup/transaction-group.html)

这里简单描述下，为了方便理解，详细可以看官方文档

seata-server(即事物协调者TC)可以向nacos注册中心注册多个实例，当然服务名肯定是一样，但是有一个cluster属性可以不同，相同cluster的是一个集群。即多个实例可以构成多个集群，但是服务名是一样的。那么相应的客户端便是事务分组这么一个概念，一个事务分组可以访问一个集群，客户端的事务分组和TC集群建立一个映射关系。

举例: 

TC有两个集群clusterA、clusterB，服务名为seata-server

客户端应用有A、B、C 3个微服务，事务分组分别为group1、group2

然后事务分组与TC集群映射关系为group1 -> clusterA、group2 -> clusterB

那么微服务A、B只能拿到TC clusterA集群下的节点信息，微服务C只能拿到TC clusterB集群下的节点信息。

* TC cluster注册配置(服务端注册中心配置)

```yaml
seata:
  registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: nacos
      nacos:
        application: seata-server
        server-addr: 127.0.0.1:8848
        group: SEATA_GROUP
        namespace: c7123a1e-d438-412c-af72-cdf3ab18a547
        # 集群名称
        cluster: clusterA
```

* 客户端事务分组指定

```yaml
seata:
  tx-service-group: group1
```

* 事务分组与TC cluster映射关系

  通过用户配置的配置中心去寻找service.vgroupMapping .[*事务分组*名]=[TC cluster name]或者直接通过bootstrap.yml文件配置seata.service.vgroup-mapping.事务分组名=集群名称

```properties
service.vgroupMapping.group1=clusterA
```

### 客户端代码

分为3个微服务seata-business、seata-order、seata-storage。

其中seata-order用于下单，seata-storage用于扣减库存、seata-business通过open feign调用这两个服务完成购买商品过程。

为了简单起见只用了一个数据库(seatadb)。

[项目地址](https://github.com/wangtaoj/SpringCloudAlibaba/tree/master/seata-example)