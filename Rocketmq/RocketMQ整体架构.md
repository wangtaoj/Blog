> RocketMQ 4.9.4

### 基础概念

集群的形成主要分为标识、注册和同步三个步骤

1. **统一标识**：每个Broker启动时，都会读取其配置文件中的`brokerClusterName`（集群名称）。所有拥有相同`brokerClusterName`的Broker，在逻辑上就属于同一个RocketMQ集群。
2. **注册发现**：一个集群由多个独立的**Broker Group**组合而成。每个Group通过**相同的Broker名称（`brokerName`）** 和**不同的角色ID（`brokerId`）** 来区分主从，形成内部的高可用单元。例如，`brokerId=0`为Master，`>0`为Slave。属于同一个Group的节点上，存储着同样的数据分片。
3. **数据同步**：`brokerClusterName` 是Broker集群的身份标识。Broker启动后，会通过 `brokerClusterName` 与NameServer交互，实现**动态注册与发现**。Broker启动后会与所有NameServer节点建立长连接，并定时（默认30秒）发送心跳，上报自身信息。NameServer会维护`clusterAddrTable`，并在120秒内未收到心跳时，剔除该Broker。

### 集群中的主要角色

#### NameServer

NameServer是一个简单的 Topic 路由注册中心，支持 Topic、Broker 的动态注册与发现。

主要包括两个功能：

- **Broker管理**，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；
- **路由信息管理**，每个NameServer将保存关于 Broker 集群的整个路由信息和用于客户端查询的队列信息。Producer和Consumer通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。

NameServer通常会有多个实例部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，客户端仍然可以向其它NameServer获取路由信息。

#### Broker

Broker主要负责消息的存储、投递和查询以及服务高可用保证。

在 Master-Slave 架构中，Broker 分为 Master 与 Slave。一个Master可以对应多个Slave，但是一个Slave只能对应一个Master。Master 与 Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId 来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个，`brokerClusterName`相同时则共同组成一个集群。

#### Producer

发布消息的角色。Producer通过 MQ 的负载均衡模块选择相应的 Broker 集群队列进行消息投递。

#### Consumer

消息消费的角色。

- 支持以推（push），拉（pull）两种模式对消息进行消费。
- 同时也支持**集群方式**和广播方式的消费。
- 提供实时消息订阅机制，可以满足大多数用户的需求。

### 长连接关系

- 每个 **Broker** 与 **NameServer** 集群中的所有节点建立长连接，定时注册 Topic 信息到所有 NameServer。
- **Producer** 与 **NameServer** 集群中的其中一个节点建立长连接，定期从 NameServer 获取Topic路由信息。并向提供 Topic 服务的 Master 建立长连接，且定时向 Master 发送心跳。Producer 完全无状态。(注意并不是broker集群中的所有master节点，比如Producer目前只发送topic1主题消息，而topic1只分布在broker1节点上，broker2、broker3没有topic1主题，则只会与broker1建立长连接)。
- **Consumer** 与 **NameServer** 集群中的其中一个节点建立长连接，定期从 NameServer 获取 Topic 路由信息。并向提供 Topic 服务的 Master、Slave 建立长连接，且定时向 Master、Slave发送心跳。Consumer 既可以从 Master 订阅消息，也可以从Slave订阅消息。(注意不是broker集群中的所有节点，看消费者当前订阅了哪些主题，然后这些主题分布在哪些broker上，就与哪些broker建立长连接)。

### 长连接相关源码提示

建立长连接是在`NettyRemotingClient.getAndCreateChannel()`这个方法，根据参数传入的地址建立长连接，若已经建立则直接返回。

生产者或者消费者调用broker服务端的api时，都会先调用`getAndCreateChannel`方法获取到`Channel`对象来通信。

#### 维持心跳

无论是生产者还是消费，和broker发送心跳的逻辑都是在`MQClientInstance.sendHeartbeatToAllBroker()`方法中。

别看名字是所有的broker，保存broker地址的brokerAddrTable这个map对象是根据生产者要发送的topic或者消费者订阅的topic从NameServer中获取的路由信息构建而成的，取决于这些topic分布在哪些broker上。

```java
/**
 * key: brokerName
 * value: Map对象， key=brokerId, value=brokerIp
 */
private final ConcurrentMap<String, HashMap<Long, String/>> brokerAddrTable =
       new ConcurrentHashMap<String, HashMap<Long, String>>();
```

构建brokerAddrTable变量是在`MQClientInstance.updateTopicRouteInfoFromNameServer()`这个方法，注意看无参的那个方法。

### 创建主题

生产者发送消息时必须先有主题，有一个参数可以控制broker是否可以自动创建主题(无主题时)，但是不推荐这样做，自动创建的主题只有4个队列。

相关参数如下

autoCreateTopicEnable：控制broker是否可以自动创建主题

defaultTopicQueueNums：自动创建服务器不存在的topic，默认创建的队列数

本质上是复用这个内部topic(TBW102)的配置信息，autoCreateTopicEnable用于控制broker启动时是否主动创建TBW102这个主题。

#### 手动创建主题的命令

集群模式 (`-c`)

```bash
./mqadmin updateTopic -n <NameServer地址>:9876 -c DefaultCluster -t <你的Topic名> -r 9 -w 9
```

**`-r` / `-w` (读/写队列数)**：通常设为相同值且为Master数量的整数倍，以均匀分布负载。在3主架构下，常见设置为 6, 9, 12

特点: 自动在集群内所有Master Broker上均匀分布队列

Broker模式 (`-b`)

该模式可以逐个指定 Broker，精确控制分布，可实现非均匀、定制化的队列分布。需要多次执行命令，**每次针对一个Master Broker**。

假设3个Master Broker地址和端口如下：

- `broker-a`：192.168.1.10:10911
- `broker-b`：192.168.1.20:10911
- `broker-c`：192.168.1.30:10911

若要为 `order_topic` 创建共9个写队列的不均匀分布（如 `broker-a` 放4个，`broker-b` 放3个，`broker-c` 放2个），可以依次执行：

```bash
# 为 broker-a 创建4个队列
./mqadmin updateTopic -n 192.168.1.10:9876 -b 192.168.1.10:10911 -t order_topic -r 4 -w 4

# 为 broker-b 创建3个队列
./mqadmin updateTopic -n 192.168.1.20:9876 -b 192.168.1.20:10911 -t order_topic -r 3 -w 3

# 为 broker-c 创建2个队列
./mqadmin updateTopic -n 192.168.1.30:9876 -b 192.168.1.30:10911 -t order_topic -r 2 -w 2
```

