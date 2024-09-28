> skywalking-oap 10.0.1、skywalking-ui 10,0,1、skywalking-agent 9.3.0

### 简要介绍

skywalking主要包含下面三部分

* OAP：skywalking服务端、存储介质可选用关系型数据、ES、Hbase等。
* UI：用于展示各项数据的一个web页面。
* Agent：客户端应用使用，用于收集客户端的各项指标数据上报到OAP中。

### 服务端搭建

compose.yml

```yaml
services:
  skywalking-oap-server:
    image: apache/skywalking-oap-server:10.0.1-java21
    container_name: skywalking-oap-server
    ports:
      - "11800:11800"
      - "12800:12800"
    environment:
      TZ: Asia/Shanghai
      SW_STORAGE: elasticsearch
      SW_STORAGE_ES_CLUSTER_NODES: host.docker.internal:9200
    networks:
      skywalking:
        aliases:
          - skywalking-oap-server

  skywalking-ui:
    image: apache/skywalking-ui:10.0.1-java21
    container_name: skywalking-ui
    ports:
      - "9080:8080"
    environment:
      TZ: Asia/Shanghai
      SW_OAP_ADDRESS: http://skywalking-oap-server:12800
    depends_on:
      - skywalking-oap-server
    networks:
      skywalking:
        aliases:
          - skywalking-ui

networks:
  skywalking:
    name: skywalking
    driver: bridge
```

使用ES作为存储介质，高版本skywalking的环境变量SW_STORAGE不再需要明确指定是elasticsearch6还是elasticsearch7了，并且还支持elasticsearch 8.x的版本。

OAP开放了两个端口，其中11800给agent使用，12800给ui使用。

启动完毕后访问`127.0.0.1:9080`即可看到web可视化界面。

### 关于ES存储的说明

[官方文档说明](https://skywalking.apache.org/docs/main/v10.0.1/en/setup/backend/storages/elasticsearch/)

数据基本都存储在以sw开头的索引名中，并且这些索引名称都是滚动的，后面会跟一个日期，也就是说滚动步长到期后，会创建一个新的索引来存储数据。

滚动步长参数如下所示

```yaml
storage:
  elasticsearch:
	dayStep: ${SW_STORAGE_DAY_STEP:1}
```

默认值为1，所以每天都会重新创建一个索引来保存当天的数据，可以通过环境变量`SW_STORAGE_DAY_STEP`指定值。

比如指定值为7，即每个索引存储一周的数据，OAP服务端与2024-09-01启动。

那么[20240901, 20240907]的数据都会保存到sw_log-20240901(sw-log是用于存储上报的应用日志索引，拿这个索引举例)中。

[20240908, 20240914]的数据会保存到sw_log-20240908中。

### 关于数据存储的过期时间

[官方文档说明](https://skywalking.apache.org/docs/main/v10.0.1/en/setup/backend/ttl/)

Skywalking OAP会删除掉历史数据，可以通过参数指定保留多少天的数据。 

recordDataTTL用于指定记录数据、比如trace、log

metricsDataTTL用于指定指标数据，指标它不是一个实时数据，是一个统计聚合分析数据。包括了metrics for service, instance, endpoint, and topology map。

```yaml
recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3} # Unit is day
metricsDataTTL: ${SW_CORE_METRICS_DATA_TTL:7} # Unit is day
```

**值的注意的细节**

当滚动步长参数不是1时，拿ES来说，dayStep设置为7，想要保留15天的数据，要设置ttl为22。

因为删除的时候是删的整个索引，而这个索引包函了7天的数据。

sw_log-20240901包含了01-07这7天的数据，如果ttl设置成15，那么到20240916这天时，就会把sw_log-20240901索引删掉，那么02-07的数据都没了，但是这些数据距离20240916还没有超过15天，是不应该删除的。