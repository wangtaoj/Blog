>RocketMQ 4.9.4

### 基本原理

**目前4.9.x版本中，广播模式((MessageModel.BROADCASTING))下不支持顺序消费消息，因此只讨论集群模式(MessageModel.CLUSTERING)。没试过最新的5.x版本支不支持。**

RocketMQ的topic可以有多个逻辑队列，在集群模式(MessageModel.CLUSTERING)下，一个逻辑队列只会被同一个消费者组下的一个消费者消费，这样可以起到负载均衡的作用，提升消费效率。

在RocketMQ中，顺序消费消息需要生产者和消费者互相合作才行。
- 生产者把需要顺序消费的消息发送到同一个逻辑队列，因为队列先进先出呀。如果发送到不同的逻辑队列，消费者是无法保证顺序消费的。
- 消费者从该逻辑队列消费消息时，依次取消息顺序消费即可。消费者内部是有一个线程池的，用于消费消息，顺序模式下，**只会有一个线程去消费同一个逻辑队列，这样可以保证顺序，同时避免线程间的锁竞争，浪费资源，如果有多个线程去消费，简单加锁无法控制哪个线程先执行哪个线程后执行来保证顺序消费。**

**注：若消息消费失败，后面的消息不会被消息，因为要保证顺序消费，直到当前消息重试成功或者超过最大重试次数后进入到死信队列后才会消费后续的消息。**

### 源码解析

先看下消息消费接口，省略了无关方法

```java
public interface ConsumeMessageService {
    
    /**
     * @param msgs 拉取到的消息列表, 都是在同一个逻辑队列中
     * @param processQueue 处理队列，一个逻辑队列对应一个processQueue
     * @param messageQueue 逻辑队列
     * @param dispathToConsume 是否提交任务给消费线程池
     */
    void submitConsumeRequest(
        final List<MessageExt> msgs,
        final ProcessQueue processQueue,
        final MessageQueue messageQueue,
        final boolean dispathToConsume);
}
```

这里重点说下参数`dispathToConsume`的作用
- 并发消费模式，该参数无意义
- 顺序消费模式，用于控制同一个逻辑队列只会由一个线程消费的关键参数，值为true时，才会提交任务给消费线程池。消费者拉取到消息列表后，先会把消息列表添加到`processQueue`对象中的`TreeMap`(顺序消费肯定是有序Map)中，然后返回个`boolean`值给`dispathToConsume`，会判断当前逻辑队列是否正在被消费，如果正在被消费，则返回false，不用往线程中添加一个任务，因为已经有线程在处理了。否则返回true，往线程池添加一个任务，等待线程池中的线程来消费。

具体逻辑来看源码，首先入口在消费者拉取消息的方法中
#### 拉取消息 & 添加消费任务到消费线程池

`DefaultMQPushConsumerImpl`类中的`pullMessage()`

```java
/**
 * 省略了很多无关代码
 */
PullCallback pullCallback = new PullCallback() {
    @Override
    public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
            pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                subscriptionData);

            switch (pullResult.getPullStatus()) {
                case FOUND:
                    long prevRequestOffset = pullRequest.getNextOffset();
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                    long pullRT = System.currentTimeMillis() - beginTimestamp;
                    DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                        pullRequest.getMessageQueue().getTopic(), pullRT);

                    long firstMsgOffset = Long.MAX_VALUE;
                    if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    } else {
                        firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                            pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());
                        // 将拉取到的消息列表添加到processQueue对象的TreeMap中
                        boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                        // 提交消费任务
                        DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                            pullResult.getMsgFoundList(),
                            processQueue,
                            pullRequest.getMessageQueue(),
                            dispatchToConsume);
                        // 继续拉取任务
                        if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                            DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                        } else {
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        }
                    }

                    if (pullResult.getNextBeginOffset() < prevRequestOffset
                        || firstMsgOffset < prevRequestOffset) {
                        log.warn(
                            "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                            pullResult.getNextBeginOffset(),
                            firstMsgOffset,
                            prevRequestOffset);
                    }

                    break;
                default:
                    break;
            }
        }
    }
}
```

看到调用了两个关键方法`putMessage`，`submitConsumeRequest`。

先看`ProcessQueue`类的`putMessage`方法，该方法不仅添加了消息，而且返回值就是`dispatchToConsume`参数的值。

```java
public boolean putMessage(final List<MessageExt> msgs) {
    // 默认为false
    boolean dispatchToConsume = false;
    try {
        // 加写锁，只能由一个线程来操作msgTreeMap
        this.treeMapLock.writeLock().lockInterruptibly();
        try {
            int validMsgCnt = 0;
            for (MessageExt msg : msgs) {
                /*
                 * 将消息添加到msgTreeMap, key为该消息在逻辑队列中的偏移量(下标，从0开始)
                 * 偏移量越小，说明消息最先进入该逻辑队列，也意味着最先消费
                 */
                MessageExt old = msgTreeMap.put(msg.getQueueOffset(), msg);
                if (null == old) {
                    validMsgCnt++;
                    this.queueOffsetMax = msg.getQueueOffset();
                    msgSize.addAndGet(msg.getBody().length);
                }
            }
            msgCount.addAndGet(validMsgCnt);

            /*
             * consuming: 初始化时为false, 消费完了之后将消息删除后msgTreeMap没有消息了也会设置成false
             * 存在消息，并且没有线程正在消费该逻辑队列，说明需要添加任务到线程池中，由一个新的线程来消费消息
             * 否则，由正在消费的线程来消费新加入的消息
             */
            if (!msgTreeMap.isEmpty() && !this.consuming) {
                dispatchToConsume = true;
                this.consuming = true;
            }

            if (!msgs.isEmpty()) {
                MessageExt messageExt = msgs.get(msgs.size() - 1);
                String property = messageExt.getProperty(MessageConst.PROPERTY_MAX_OFFSET);
                if (property != null) {
                    long accTotal = Long.parseLong(property) - messageExt.getQueueOffset();
                    if (accTotal > 0) {
                        this.msgAccCnt = accTotal;
                    }
                }
            }
        } finally {
            this.treeMapLock.writeLock().unlock();
        }
    } catch (InterruptedException e) {
        log.error("putMessage exception", e);
    }

    return dispatchToConsume;
}
```

再看`ConsumeMessageOrderlyService`类(顺序消费的实现类)中的`submitConsumeRequest`方法。

```java
@Override
public void submitConsumeRequest(
    final List<MessageExt> msgs,
    final ProcessQueue processQueue,
    final MessageQueue messageQueue,
    final boolean dispathToConsume) {
    // 根据dispathToConsume的值来决定是否要往线程池中添加消费任务
    if (dispathToConsume) {
        ConsumeRequest consumeRequest = new ConsumeRequest(processQueue, messageQueue);
        this.consumeExecutor.submit(consumeRequest);
    }
}
```

#### 消息消费逻辑

当把消费任务添加到消费线程池后，具体的消费逻辑便是由消费线程池中的线程来执行。话不多说，肯定看`ConsumeRequest`任务类的`run`方法。

```java
@Override
public void run() {
    if (this.processQueue.isDropped()) {
        log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
        return;
    }

    /* 
     * 根据逻辑队列加消费全局锁，经过前面分析，正常情况下是不会有别的线程来竞争锁的
     * 特殊情况: 本线程已经将processQueue中积压的消息全部消费完了，此时拉取线程刚好拉取到消息，那么会提交消费任务到
     *          线程池中，最坏情况下可能会有两个线程来消费线程，因此需要加一个全局锁的
     */
    final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
    synchronized (objLock) {
        if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
            || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {
            final long beginTime = System.currentTimeMillis();
            // 循环消费消息
            for (boolean continueConsume = true; continueConsume; ) {
                if (this.processQueue.isDropped()) {
                    log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                    break;
                }

                if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    && !this.processQueue.isLocked()) {
                    log.warn("the message queue not locked, so consume later, {}", this.messageQueue);
                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                    break;
                }

                if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    && this.processQueue.isLockExpired()) {
                    log.warn("the message queue lock expired, so consume later, {}", this.messageQueue);
                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                    break;
                }

                // 该线程最大的消费持续时间，结束当前线程，同时添加一个消费任务给线程池，dispathToConsume永远传true
                long interval = System.currentTimeMillis() - beginTime;
                if (interval > MAX_TIME_CONSUME_CONTINUOUSLY) {
                    ConsumeMessageOrderlyService.this.submitConsumeRequestLater(processQueue, messageQueue, 10);
                    break;
                }

                // 一次消费的条数，默认为1
                final int consumeBatchSize =
                    ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
                // 从processQueue对象中的TreeMap获取消息(顺序获取)
                List<MessageExt> msgs = this.processQueue.takeMessages(consumeBatchSize);
                defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());
                if (!msgs.isEmpty()) {
                    final ConsumeOrderlyContext context = new ConsumeOrderlyContext(this.messageQueue);

                    ConsumeOrderlyStatus status = null;

                    ConsumeMessageContext consumeMessageContext = null;
                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        consumeMessageContext = new ConsumeMessageContext();
                        consumeMessageContext
                            .setConsumerGroup(ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumerGroup());
                        consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
                        consumeMessageContext.setMq(messageQueue);
                        consumeMessageContext.setMsgList(msgs);
                        consumeMessageContext.setSuccess(false);
                        // init the consume context type
                        consumeMessageContext.setProps(new HashMap<String, String>());
                        ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
                    }

                    long beginTimestamp = System.currentTimeMillis();
                    ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
                    boolean hasException = false;
                    try {
                        this.processQueue.getConsumeLock().lock();
                        if (this.processQueue.isDropped()) {
                            log.warn("consumeMessage, the message queue not be able to consume, because it's dropped. {}",
                                this.messageQueue);
                            break;
                        }
                        // 执行具体的消费逻辑
                        status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);
                    } catch (Throwable e) {
                        log.warn(String.format("consumeMessage exception: %s Group: %s Msgs: %s MQ: %s",
                            RemotingHelper.exceptionSimpleDesc(e),
                            ConsumeMessageOrderlyService.this.consumerGroup,
                            msgs,
                            messageQueue), e);
                        hasException = true;
                    } finally {
                        this.processQueue.getConsumeLock().unlock();
                    }

                    if (null == status
                        || ConsumeOrderlyStatus.ROLLBACK == status
                        || ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                        log.warn("consumeMessage Orderly return not OK, Group: {} Msgs: {} MQ: {}",
                            ConsumeMessageOrderlyService.this.consumerGroup,
                            msgs,
                            messageQueue);
                    }

                    long consumeRT = System.currentTimeMillis() - beginTimestamp;
                    if (null == status) {
                        if (hasException) {
                            returnType = ConsumeReturnType.EXCEPTION;
                        } else {
                            returnType = ConsumeReturnType.RETURNNULL;
                        }
                    } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
                        returnType = ConsumeReturnType.TIME_OUT;
                    } else if (ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                        returnType = ConsumeReturnType.FAILED;
                    } else if (ConsumeOrderlyStatus.SUCCESS == status) {
                        returnType = ConsumeReturnType.SUCCESS;
                    }

                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
                    }

                    if (null == status) {
                        status = ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                    }

                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        consumeMessageContext.setStatus(status.toString());
                        consumeMessageContext
                            .setSuccess(ConsumeOrderlyStatus.SUCCESS == status || ConsumeOrderlyStatus.COMMIT == status);
                        consumeMessageContext.setAccessChannel(defaultMQPushConsumer.getAccessChannel());
                        ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
                    }

                    ConsumeMessageOrderlyService.this.getConsumerStatsManager()
                        .incConsumeRT(ConsumeMessageOrderlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

                    continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
                } else {
                    continueConsume = false;
                }
            }
        } else {
            if (this.processQueue.isDropped()) {
                log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                return;
            }

            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
        }
    }
}
```

#### 消费之后的结果处理

执行具体消费逻辑时，可能消费成功，或者消费失败。根据消费结果也会有不一样的处理行为，比如消息重试，消费者位点提交等。

目前消费状态仅有两个枚举，其它的值已经过时了，不要使用

```java
public enum ConsumeOrderlyStatus {
    /**
     * 成功状态
     */
    SUCCESS,

    /**
     * 过时
     */
    @Deprecated
    ROLLBACK,

    /**
     * 过时
     */
    @Deprecated
    COMMIT,
    
    /**
     * 需要重试
     */
    SUSPEND_CURRENT_QUEUE_A_MOMENT;
}
```

如果消费消息时发生异常，则结果默认为`SUSPEND_CURRENT_QUEUE_A_MOMENT`。

```java
public boolean processConsumeResult(
    final List<MessageExt> msgs,
    final ConsumeOrderlyStatus status,
    final ConsumeOrderlyContext context,
    final ConsumeRequest consumeRequest
) {
    boolean continueConsume = true;
    long commitOffset = -1L;
    if (context.isAutoCommit()) {
        switch (status) {
            case SUCCESS:
                // 消费成功，获取消费最大的位点+1
                commitOffset = consumeRequest.getProcessQueue().commit();
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                break;
            case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                // 消息重试
                if (checkReconsumeTimes(msgs)) {
                    consumeRequest.getProcessQueue().makeMessageToConsumeAgain(msgs);
                    this.submitConsumeRequestLater(
                        consumeRequest.getProcessQueue(),
                        consumeRequest.getMessageQueue(),
                        context.getSuspendCurrentQueueTimeMillis());
                    continueConsume = false;
                } else {
                    // 超过最大重试次数, 获取消费最大的位点+1
                    commitOffset = consumeRequest.getProcessQueue().commit();
                }
                break;
            default:
                break;
        }
    } else {
        switch (status) {
            case SUCCESS:
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                break;
            case COMMIT:
                commitOffset = consumeRequest.getProcessQueue().commit();
                break;
            case ROLLBACK:
                consumeRequest.getProcessQueue().rollback();
                this.submitConsumeRequestLater(
                    consumeRequest.getProcessQueue(),
                    consumeRequest.getMessageQueue(),
                    context.getSuspendCurrentQueueTimeMillis());
                continueConsume = false;
                break;
            case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                if (checkReconsumeTimes(msgs)) {
                    consumeRequest.getProcessQueue().makeMessageToConsumeAgain(msgs);
                    this.submitConsumeRequestLater(
                        consumeRequest.getProcessQueue(),
                        consumeRequest.getMessageQueue(),
                        context.getSuspendCurrentQueueTimeMillis());
                    continueConsume = false;
                }
                break;
            default:
                break;
        }
    }
    // 更新消费位点
    if (commitOffset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
        this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), commitOffset, false);
    }

    return continueConsume;
}
```

#### 消费位点提交

与并发消费模式不一样，只会在消费成功后或者超过最大重试次数进入到死信队列后才会进行消费位点的更新。

* 并发消费，不管是否消费成功，都会更新位点，并且使用`ProcessQueue`中`TreeMap`里最小的偏移量进行更新。
* 顺序消费，消费成功后或者超过最大重试次数进入到死信队列后才会更新消费位点，并且使用最大的偏移位点加+1。很好理解，因为顺序消费，只会有一个线程消费消息，不会出现小的偏移量消费失败而大的偏移量消费成功的场景。(进入死信队列除外)

具体看上面`processConsumeResult`方法

ProcessQueue.java
```java
public long commit() {
    try {
        this.treeMapLock.writeLock().lockInterruptibly();
        try {
            /* 获取最大的偏移量
             * consumingMsgOrderlyTreeMap中的元素在takeMessages方法中添加的
             * 也就是当前消费的消息列表
             */
            Long offset = this.consumingMsgOrderlyTreeMap.lastKey();
            msgCount.addAndGet(0 - this.consumingMsgOrderlyTreeMap.size());
            for (MessageExt msg : this.consumingMsgOrderlyTreeMap.values()) {
                msgSize.addAndGet(0 - msg.getBody().length);
            }
            // 清空缓冲区
            this.consumingMsgOrderlyTreeMap.clear();
            if (offset != null) {
                // 返回下一次的消费位点
                return offset + 1;
            }
        } finally {
            this.treeMapLock.writeLock().unlock();
        }
    } catch (InterruptedException e) {
        log.error("commit exception", e);
    }

    return -1;
}
```

#### 重试机制

并发消费时，是通过将消息发送到消费者组中的重试队列来实现的，默认重试16次，而且每次消费间隔呈阶梯式增长。

顺序消费，因为要考虑顺序，是直接把消费失败的消息列表重新添加到`ProcessQueue`对象中的`TreeMap`中，消费时，线程直接从`ProcessQueue`对象中的`TreeMap`拿消息进行消费的。默认重试次数为`Integer.MAX_VALUE`。超过最大重试次数后，将这些失败的消息发送到死信队列，然后正常消费后续消息。

消费间隔为固定的值，默认为1s，可以全局设置，也可以针对当前消息进行设置。

```java
public boolean processConsumeResult(
        final List<MessageExt> msgs,
        final ConsumeOrderlyStatus status,
        final ConsumeOrderlyContext context,
        final ConsumeRequest consumeRequest
    ) {
    boolean continueConsume = true;
    long commitOffset = -1L;
    if (context.isAutoCommit()) {
        switch (status) {
            case COMMIT:
            case ROLLBACK:
                log.warn("the message queue consume result is illegal, we think you want to ack these message {}",
                    consumeRequest.getMessageQueue());
            case SUCCESS:
                commitOffset = consumeRequest.getProcessQueue().commit();
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                break;
            case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                // 重试, 检查是否超过最大重试次数
                if (checkReconsumeTimes(msgs)) {
                    // 将消息重新添加到ProcessQueue中
                consumeRequest.getProcessQueue().makeMessageToConsumeAgain(msgs);
                    // 添加消费任务到线程池中, 注意第三个参数
                    this.submitConsumeRequestLater(
                        consumeRequest.getProcessQueue(),
                        consumeRequest.getMessageQueue(),
                        context.getSuspendCurrentQueueTimeMillis());
                    continueConsume = false;
                } else {
                    // 进入死信队列后，获取消费位点，用于下面代码提交消费位点
                    commitOffset = consumeRequest.getProcessQueue().commit();
                }
                break;
            default:
                break;
        }
    }
}
```

```java
private boolean checkReconsumeTimes(List<MessageExt> msgs) {
    boolean suspend = false;
    if (msgs != null && !msgs.isEmpty()) {
        for (MessageExt msg : msgs) {
            // 是否超过最大重试次数, 默认为Integer.MAX_VALUE
            if (msg.getReconsumeTimes() >= getMaxReconsumeTimes()) {
                MessageAccessor.setReconsumeTime(msg, String.valueOf(msg.getReconsumeTimes()));
                // 将消息发送到死信队列中, 发送成功将会返回true
                if (!sendMessageBack(msg)) {
                    suspend = true;
                    msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                }
            } else {
                // 重试次数+1
                suspend = true;
                msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
            }
        }
    }
    return suspend;
}

private int getMaxReconsumeTimes() {
    // default reconsume times: Integer.MAX_VALUE
    if (this.defaultMQPushConsumer.getMaxReconsumeTimes() == -1) {
        return Integer.MAX_VALUE;
    } else {
        return this.defaultMQPushConsumer.getMaxReconsumeTimes();
    }
}
```

`ProcessQueue.java`
```java
public void makeMessageToConsumeAgain(List<MessageExt> msgs) {
    try {
        this.treeMapLock.writeLock().lockInterruptibly();
        try {
            for (MessageExt msg : msgs) {
                // 删除临时缓冲区的消息
                this.consumingMsgOrderlyTreeMap.remove(msg.getQueueOffset());
                // 重新放回
                this.msgTreeMap.put(msg.getQueueOffset(), msg);
            }
        } finally {
            this.treeMapLock.writeLock().unlock();
        }
    } catch (InterruptedException e) {
        log.error("makeMessageToCosumeAgain exception", e);
    }
}
```

`ConsumeMessageOrderlyService.java`
```java
private void submitConsumeRequestLater(
    final ProcessQueue processQueue,
    final MessageQueue messageQueue,
    final long suspendTimeMillis
) {
    // 上面调用时给的是context.getSuspendCurrentQueueTimeMillis()，默认为-1，我们也可以在消费时给它设置一个值
    long timeMillis = suspendTimeMillis;
    if (timeMillis == -1) {
        // 全局参数，默认为1秒
        timeMillis = this.defaultMQPushConsumer.getSuspendCurrentQueueTimeMillis();
    }

    // 10ms <= 间隔时间 <= 30s
    if (timeMillis < 10) {
        timeMillis = 10;
    } else if (timeMillis > 30000) {
        timeMillis = 30000;
    }

    // 定时任务提交消费任务
    this.scheduledExecutorService.schedule(new Runnable() {

        @Override
        public void run() {
            ConsumeMessageOrderlyService.this.submitConsumeRequest(null, processQueue, messageQueue, true);
        }
    }, timeMillis, TimeUnit.MILLISECONDS);
}
```

由上面代码可知

设置全局重试间隔时间

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_name");  
// 全局重试间隔时间为3s
consumer.setSuspendCurrentQueueTimeMillis(3000);
```

设置当前消息列表重试间隔时间

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_name");
consumer.registerMessageListener(new MessageListenerOrderly() {
            
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
        // 设置当前消息列表重试间隔时间, context对象只是当前消费逻辑的上下文，非全局上下文
        context.setSuspendCurrentQueueTimeMillis(3000);
        try {
            // 消费消息
            doSomething(msgs);
            return ConsumeOrderlyStatus.SUCCESS;
        } catch (Exception e) {
            return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
        }
    }
});

```