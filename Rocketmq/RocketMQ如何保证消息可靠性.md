需要从Producer、Broker、Consumer这三个方面来阐述。RocketMQ并不是说能100%保证消息不会丢失，而是通过各种手段把丢失概率降到极低。

### Producer

依赖broker响应机制并且失败会重试

| **发送方式** | **是否可靠** |
| ------------ | ------------ |
| oneWay       | 最差         |
| async        | 中等         |
| sync         | 最可靠       |

同步发送消息时，只有收到 Broker 成功响应(SEND_OK)，才会认为消息发送成功，当然异步方式也有回调。但是异步方式失败了不会有重试逻辑。

同步发送时，若broker没有响应，或者响应不是SEND_OK(这个需要打开参数才行)，会重试2次。**并且重试时会选择另外的broker来进行发送**，毕竟还是发送给失败的这个broker，可能还是会失败。

重试逻辑: DefaultMQProducerImpl.sendDefaultImpl()

挑选队列: DefaultMQProducerImpl.selectOneMessageQueue()，会轮训队列进行发送，若发送失败，轮训下一个队列时还会判断该队列是否分布在这个发送失败的broker上，若是则继续轮训下一个队列。若没有另外一个broker可选，则回退默认的轮训策略，不会再判断broker了。

若有响应，但是响应不是SEND_OK时，需要设置`retryAnotherBrokerWhenNotStoreOK`参数才会进行重试逻辑，默认为false。

**只有同步发送，并且是普通消息(无序)才会有重试逻辑，因为普通消息可以随意选择发往不同的队列**

```java
/**
 * 有重试逻辑
 */
public SendResult send(Message msg);
public SendResult send(Message msg,long timeout);

/**
 * 无重试逻辑
 * 因为有指定要发送的队列
 */
public SendResult send(Message msg, MessageQueueSelector selector, Object arg)；
public SendResult send(Message msg, MessageQueue mq);
```

### Broker

#### 两种刷盘模式

同步刷盘 SYNC_FLUSH，真正落盘后才返回成功，即使Broker突然宕机，消息也已经持久化到磁盘上了。

异步刷盘 ASYNC_FLUSH（默认），此时消息只是写入到操作系统的PageCache，就返回成功了，还没有真正持久化到磁盘中，后台线程异步刷盘，性能会高很多，若Producer已经收到成功的响应，此时Broker突然宕机，异步线程还没有刷盘，那消息就丢失了。

#### 两种复制方式

异步复制(ASYNC_MASTER)，本地写成功就ACK，然后异步同步 Slave。风险: Master宕机Slave还没同步完成。

同步复制(SYNC_MASTER)，master和slave都成功，才ACK。可靠性极高，但是性能下降明显。

**因此最可高的方式是同步发送+同步刷盘+同步复制**

### Consumer

消费者也有ack机制，只有消费成功后才提交消费进度。并且消费失败有重试机制，超过最大重试次数后进入死信队列。**消费端一定要最好幂等性设计**，有太多的情况会导致消息重复投递，最典型的是消费进度没有提交成功。

