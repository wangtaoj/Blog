>RocketMQ 4.9.4

### 重置功能

在RocketMQ Dashboard的主题界面，提供了两个功能
- 重置消费位点，按照时间戳重置指定消费者组的消费位点，会按照时间戳找到最接近的消息位点，RocketMQ在存储消息时会记录消息的存入时间。
- 跳过堆积，直接跳到最大消费位点。

### 使用限制

* 只适合集群消费模式，并且消费者要在线
* 不能指定消费某一条消息。

### 基本原理

无论是重置消费位点还是跳过堆积功能，原理都是一样的，只是重新设置的消费位点不一样。
**集群消费模式下，broker会存储消费者的消费进度。**
大致流程如下
- 找到topic所在的**所有broker**(一般都是master)，循环请求每一个broker重置消费位点。
- broker端根据时间戳找到该主题下每一个逻辑队列的消费位点，用于进行重置。然后找到订阅该主题的所有消费者，发送请求给消费者，通知消费者用这个新的消费位点去更新。**注意此时broker端是不会主动更新自己存储的消费者组的消费位点的。**
- 每一个消费者更新**自己负责**的逻辑队列的消费位点，**并且同步到broker中**，然后删除对应的ProcessQueue。这里会挂起消费者，不会去拉取新的消息。
- 由于ProcessQueue被删除，消费者下一次重新平衡时会检测到，于是为这个逻辑队列(MessageQueue)创建一个新的ProcessQueue，这样该MessageQueue的消费位点会从broker中获取到我们重置的消费位点。

注:
- 为什么是topic所在的所有broker，因为需要重置该topic所有的逻辑队列。
- broker集群部署时，多个主从组成一个集群，比如一个集群包含broker-a、broker-b。而broker-a又可以分为主从结构(一主多从，brokerName相同，brokerId不同)，broker-a和broker-b是这个集群的一个分片。

### 源码简要分析

#### RocketMQ-Dashboard
入口肯定是在RocketMQ Dashboard的Controller中，点下按钮F12便可以找到请求路径了。分别对应`ConsumerController`类中`resetOffset`和`skipAccumulate`方法，一个是重置消费位点，一个是跳过堆积。

```java
@Controller
@RequestMapping("/consumer")
@Permission
public class ConsumerController {

    /**
     * 重置消费位点
     */
    @RequestMapping(value = "/resetOffset.do", method = {RequestMethod.POST})
    @ResponseBody
    public Object resetOffset(@RequestBody ResetOffsetRequest resetOffsetRequest) {
        logger.info("op=look resetOffsetRequest={}", JsonUtil.obj2String(resetOffsetRequest));
        return consumerService.resetOffset(resetOffsetRequest);
    }

    /**
     * 跳过堆积
     */
    @RequestMapping(value = "/skipAccumulate.do", method = {RequestMethod.POST})
    @ResponseBody
    public Object skipAccumulate(@RequestBody ResetOffsetRequest resetOffsetRequest) {
        logger.info("op=look resetOffsetRequest={}", JsonUtil.obj2String(resetOffsetRequest));
        return consumerService.resetOffset(resetOffsetRequest);
    }

}
```

这两个方法唯一的区别就是前台传参不同，跳过堆积前台`resetTime`传的是-1，代表着最大消息位点。

最终会调用`DefaultMQAdminExtImpl`类`resetOffsetByTimestamp`方法。

```java
public Map<MessageQueue, Long> resetOffsetByTimestamp(String topic, String group, long timestamp, boolean isForce,
        boolean isC) throws RemotingException, MQBrokerException, InterruptedException, MQClientException {
    // 获取主题的路由信息
    TopicRouteData topicRouteData = this.examineTopicRouteInfo(topic);
    // 拿到主题所在的所有broker
    List<BrokerData> brokerDatas = topicRouteData.getBrokerDatas();
    Map<MessageQueue, Long> allOffsetTable = new HashMap<>();
    if (brokerDatas != null) {
        // 循环向每一个broker发送请求
        for (BrokerData brokerData : brokerDatas) {
            // 获取broker的主节点ip(主从结构只需要主节点就可以了)
            String addr = brokerData.selectBrokerAddr();
            if (addr != null) {
                // 委托broker执行重置操作
                Map<MessageQueue, Long> offsetTable = this.mqClientInstance.getMQClientAPIImpl().invokeBrokerToResetOffset(addr, topic, group, timestamp, isForce, timeoutMillis, isC);
                if (offsetTable != null) {
                    allOffsetTable.putAll(offsetTable);
                }
            }
        }
    }
    return allOffsetTable;
}
```

#### Broker
上述方法请求Code: `RequestCode.INVOKE_BROKER_TO_RESET_OFFSET`。

对应broker中`AdminBrokerProcessor`类`resetOffset`方法。

```java
public RemotingCommand resetOffset(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
    final ResetOffsetRequestHeader requestHeader =
        (ResetOffsetRequestHeader) request.decodeCommandCustomHeader(ResetOffsetRequestHeader.class);
    LOGGER.info("[reset-offset] reset offset started by {}. topic={}, group={}, timestamp={}, isForce={}",
        RemotingHelper.parseChannelRemoteAddr(ctx.channel()), requestHeader.getTopic(), requestHeader.getGroup(),
        requestHeader.getTimestamp(), requestHeader.isForce());

    // 默认为false，走下面的逻辑
    if (this.brokerController.getBrokerConfig().isUseServerSideResetOffset()) {
        String topic = requestHeader.getTopic();
        String group = requestHeader.getGroup();
        int queueId = requestHeader.getQueueId();
        long timestamp = requestHeader.getTimestamp();
        Long offset = requestHeader.getOffset();
        return resetOffsetInner(topic, group, queueId, timestamp, offset);
    }

    boolean isC = false;
    LanguageCode language = request.getLanguage();
    switch (language) {
        case CPP:
            isC = true;
            break;
    }
    // 委托给Broker2Client
    return this.brokerController.getBroker2Client().resetOffset(requestHeader.getTopic(), requestHeader.getGroup(),
        requestHeader.getTimestamp(), requestHeader.isForce(), isC);
}
```

再看Broker2Client

```java
public RemotingCommand resetOffset(String topic, String group, long timeStamp, boolean isForce,
        boolean isC) {
        final RemotingCommand response = RemotingCommand.createResponseCommand(null);

        TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(topic);
        // 找不到topic信息，报错
        if (null == topicConfig) {
            log.error("[reset-offset] reset offset failed, no topic in this broker. topic={}", topic);
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark("[reset-offset] reset offset failed, no topic in this broker. topic=" + topic);
            return response;
        }

        Map<MessageQueue, Long> offsetTable = new HashMap<>();

        // 遍历所有的逻辑队列
        for (int i = 0; i < topicConfig.getWriteQueueNums(); i++) {
            MessageQueue mq = new MessageQueue();
            mq.setBrokerName(this.brokerController.getBrokerConfig().getBrokerName());
            mq.setTopic(topic);
            mq.setQueueId(i);

            long consumerOffset =
                this.brokerController.getConsumerOffsetManager().queryOffset(group, topic, i);
            // 从这里可知广播模式不支持重置, 只有集群模式才会把消费位点存储到broker端
            if (-1 == consumerOffset) {
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark(String.format("THe consumer group <%s> not exist", group));
                return response;
            }

            long timeStampOffset;
            // 使用最大消息位点，跳过堆积传的就是-1
            if (timeStamp == -1) {

                timeStampOffset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, i);
            } else {
                // 根据时间戳寻找一个最近的消息位点
                timeStampOffset = this.brokerController.getMessageStore().getOffsetInQueueByTime(topic, i, timeStamp);
            }

            if (timeStampOffset < 0) {
                log.warn("reset offset is invalid. topic={}, queueId={}, timeStampOffset={}", topic, i, timeStampOffset);
                timeStampOffset = 0;
            }
            // 前台传的是true，所以是可以回退的
            if (isForce || timeStampOffset < consumerOffset) {
                offsetTable.put(mq, timeStampOffset);
            } else {
                offsetTable.put(mq, consumerOffset);
            }
        }

        ResetOffsetRequestHeader requestHeader = new ResetOffsetRequestHeader();
        requestHeader.setTopic(topic);
        requestHeader.setGroup(group);
        requestHeader.setTimestamp(timeStamp);
        // 请求消费者的请求代码
        RemotingCommand request =
            RemotingCommand.createRequestCommand(RequestCode.RESET_CONSUMER_CLIENT_OFFSET, requestHeader);
        if (isC) {
            // c++ language
            ResetOffsetBodyForC body = new ResetOffsetBodyForC();
            List<MessageQueueForC> offsetList = convertOffsetTable2OffsetList(offsetTable);
            body.setOffsetTable(offsetList);
            request.setBody(body.encode());
        } else {
            // other language
            ResetOffsetBody body = new ResetOffsetBody();
            body.setOffsetTable(offsetTable);
            request.setBody(body.encode());
        }

        // 获取消费者组信息
        ConsumerGroupInfo consumerGroupInfo =
            this.brokerController.getConsumerManager().getConsumerGroupInfo(group);

        if (consumerGroupInfo != null && !consumerGroupInfo.getAllChannel().isEmpty()) {
            ConcurrentMap<Channel, ClientChannelInfo> channelInfoTable =
                consumerGroupInfo.getChannelInfoTable();
            // 给消费者组下的每一个消费者发送请求
            for (Map.Entry<Channel, ClientChannelInfo> entry : channelInfoTable.entrySet()) {
                int version = entry.getValue().getVersion();
                if (version >= MQVersion.Version.V3_0_7_SNAPSHOT.ordinal()) {
                    try {
                        this.brokerController.getRemotingServer().invokeOneway(entry.getKey(), request, 5000);
                        log.info("[reset-offset] reset offset success. topic={}, group={}, clientId={}",
                            topic, group, entry.getValue().getClientId());
                    } catch (Exception e) {
                        log.error("[reset-offset] reset offset exception. topic={}, group={} ,error={}",
                            topic, group, e.toString());
                    }
                } else {
                    response.setCode(ResponseCode.SYSTEM_ERROR);
                    response.setRemark("the client does not support this feature. version="
                        + MQVersion.getVersionDesc(version));
                    log.warn("[reset-offset] the client does not support this feature. channel={}, version={}",
                        RemotingHelper.parseChannelRemoteAddr(entry.getKey()), MQVersion.getVersionDesc(version));
                    return response;
                }
            }
        } else {
            String errorInfo =
                String.format("Consumer not online, so can not reset offset, Group: %s Topic: %s Timestamp: %d",
                    requestHeader.getGroup(),
                    requestHeader.getTopic(),
                    requestHeader.getTimestamp());
            log.error(errorInfo);
            response.setCode(ResponseCode.CONSUMER_NOT_ONLINE);
            response.setRemark(errorInfo);
            return response;
        }
        response.setCode(ResponseCode.SUCCESS);
        ResetOffsetBody resBody = new ResetOffsetBody();
        resBody.setOffsetTable(offsetTable);
        response.setBody(resBody.encode());
        return response;
    }
```

#### Consumer

broker请求代码为`RequestCode.RESET_CONSUMER_CLIENT_OFFSET`，消费者客户端对应的处理逻辑为`ClientRemotingProcessor`类中的`resetOffset`方法。

```java
public RemotingCommand resetOffset(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
    final ResetOffsetRequestHeader requestHeader =
        (ResetOffsetRequestHeader) request.decodeCommandCustomHeader(ResetOffsetRequestHeader.class);
    log.info("invoke reset offset operation from broker. brokerAddr={}, topic={}, group={}, timestamp={}",
        RemotingHelper.parseChannelRemoteAddr(ctx.channel()), requestHeader.getTopic(), requestHeader.getGroup(),
        requestHeader.getTimestamp());
    Map<MessageQueue, Long> offsetTable = new HashMap<MessageQueue, Long>();
    if (request.getBody() != null) {
        ResetOffsetBody body = ResetOffsetBody.decode(request.getBody(), ResetOffsetBody.class);
        offsetTable = body.getOffsetTable();
    }
    // 委托给MQClientInstance
    this.mqClientFactory.resetOffset(requestHeader.getTopic(), requestHeader.getGroup(), offsetTable);
    return null;
}
```

再看MQClientInstance

```java
public synchronized void resetOffset(String topic, String group, Map<MessageQueue, Long> offsetTable) {
        DefaultMQPushConsumerImpl consumer = null;
    try {
        MQConsumerInner impl = this.consumerTable.get(group);
        if (impl != null && impl instanceof DefaultMQPushConsumerImpl) {
            consumer = (DefaultMQPushConsumerImpl) impl;
        } else {
            log.info("[reset-offset] consumer dose not exist. group={}", group);
            return;
        }
        // 挂起消费者, 拉取消息线程会判断该状态, 如果挂起则会等待下一次定时任务再拉消息, 也就是挂起期间不会拉取新的消息
        consumer.suspend();

        ConcurrentMap<MessageQueue, ProcessQueue> processQueueTable = consumer.getRebalanceImpl().getProcessQueueTable();
        for (Map.Entry<MessageQueue, ProcessQueue> entry : processQueueTable.entrySet()) {
            MessageQueue mq = entry.getKey();
            if (topic.equals(mq.getTopic()) && offsetTable.containsKey(mq)) {
                ProcessQueue pq = entry.getValue();
                // 设置为true, 拉取线程将会直接结束, 消费者没有重新平衡期间, 该逻辑队列将不会有新的消息过来
                pq.setDropped(true);
                pq.clear();
            }
        }

        // 下面单独解析
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
        }

        Iterator<MessageQueue> iterator = processQueueTable.keySet().iterator();
        while (iterator.hasNext()) {
            MessageQueue mq = iterator.next();
            Long offset = offsetTable.get(mq);
            if (topic.equals(mq.getTopic()) && offset != null) {
                try {
                    // 更新消费位点
                    consumer.updateConsumeOffset(mq, offset);
                    // 持久化消费位点, 同步最新消费位点给broker，然后删除消费者内存中的消费位点
                    consumer.getRebalanceImpl().removeUnnecessaryMessageQueue(mq, processQueueTable.get(mq));
                    // 删除ProcessQueue，以便重平衡时创建新的ProcessQueu，从broker端获取新的消费位点
                    iterator.remove();
                } catch (Exception e) {
                    log.warn("reset offset failed. group={}, {}", group, mq, e);
                }
            }
        }
    } finally {
        if (consumer != null) {
            // 恢复消费者
            consumer.resume();
        }
    }
}
```

中间有一段休眠10s的代码，我的思考是此时可能存在逻辑队列正在消费消息，虽然上面挂起了消费者，也让对应的ProcessQueue无效，只是不会有新的消息过来，所以这里是想等待当前的消息消费完，不过这个时间感觉不大好把握啊。**如果消费者消费很慢，超过了10s，然后消费成功，这里也会提交消费位点，同时定时任务刚好把这次提交同步给了broker，默认情况下，消费者通过定时任务将消费者内存的消费位点发送给broker，5s一次。岂不是会把这个方法更新好的消费位点覆盖掉了，导致重置消费位点失败。**
