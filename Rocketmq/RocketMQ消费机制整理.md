> 本文以 Apache RocketMQ 4.9.4 的 Java 客户端为准。

本文围绕消费者侧最常遇到的几个问题整理：一条消息由谁消费、以什么并发模型消费、失败后如何重试、进度何时提交，以及 Broker 何时真正删除历史消息。

先建立三个基本认识：

1. `MessageQueue` 是消费分配、顺序保证和 offset 记录的基本单位，不是单条消息。
2. RocketMQ 的消费语义是**至少一次**。重试、重平衡、进度尚未持久化等情况都可能造成重复投递，业务必须幂等。
3. `DefaultMQPushConsumer` 名为 Push，但底层仍是消费者主动发起 Pull 请求；Broker 的长轮询让它表现得像“推送”。

## 消费模式：集群消费与广播消费

通过 `DefaultMQPushConsumer#setMessageModel` 配置，取值为 `MessageModel.CLUSTERING`（默认）或 `MessageModel.BROADCASTING`。

| 维度 | 集群消费（`CLUSTERING`） | 广播消费（`BROADCASTING`） |
| --- | --- | --- |
| 同组多个实例 | 同一 `MessageQueue` 同一时刻只分给其中一个实例 | 每个在线实例都消费全部队列 |
| 扩容效果 | 队列在组内重新分摊，提高吞吐 | 每个实例工作量都增加，不能分摊 |
| 消费进度 | 以 `consumerGroup + topic + queueId` 保存到 Broker | 每个客户端本地保存自己的进度 |
| 典型场景 | 订单处理、异步任务、削峰 | 配置刷新、缓存失效、通知类消息 |
| 离线实例 | 恢复后按已保存的组进度继续 | 通常不会补偿离线期间错过的消息 |

### 集群消费的分配原理

同一消费者组的每个客户端都会从 Broker 查询该组在线的 `clientId` 列表，并基于相同的队列集合与分配策略，在本地计算自己应负责的队列。默认策略是 `AllocateMessageQueueAveragely`：将排序后的队列尽量平均分给排序后的消费者。

```text
Topic: 8 个 MessageQueue，消费者组实例: C1、C2、C3
              ↓ 默认平均分配
C1: Q0、Q1、Q2    C2: Q3、Q4、Q5    C3: Q6、Q7
```

因此，消费并行度主要受队列数限制。8 个队列即使启动 20 个同组消费者，也最多只有 8 个消费者能实际分到队列。消费者上线、下线、Topic 队列数变化时会触发重平衡；交接期间可能重复消费。

关键参数与入口：

| 项目 | 说明 |
| --- | --- |
| `messageModel` | 消费模式，默认 `CLUSTERING`。同一组必须保持一致。 |
| `allocateMessageQueueStrategy` | 队列分配策略，默认平均分配；同组所有实例必须使用同一确定性策略。 |
| `RebalanceService` | 客户端周期性重平衡服务，4.9.x 默认每 20 秒执行一次，也可被事件唤醒。 |
| `RebalanceImpl.rebalanceByTopic` | 计算队列分配的核心入口。 |
| `RebalancePushImpl.computePullFromWhereWithException` | 新接管队列时，根据已提交 offset 与 `consumeFromWhere` 计算拉取起点。 |

## 消费方式：并发消费与顺序消费

消费方式由注册监听器的方法决定：

```java
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) ->
    ConsumeConcurrentlyStatus.CONSUME_SUCCESS);

consumer.registerMessageListener((MessageListenerOrderly) (msgs, context) ->
    ConsumeOrderlyStatus.SUCCESS);
```

两种方式都先把从同一逻辑队列拉到的消息放入其本地 `ProcessQueue`；差异主要在“一个队列中的消息如何交给消费线程”以及“失败如何处理”。

### 并发消费

并发消费对应 `MessageListenerConcurrently` 与 `ConsumeMessageConcurrentlyService`。同一个 `MessageQueue` 的多批消息可以被多个消费线程并行处理，因此吞吐高，但**不保证同一队列内的完成顺序**。

拉取的消息暂存在`ConsumeRequest`对象中，实现了`Runnable`接口，消费线程每次从线程池的阻塞队列中获取一个`ConsumeRequest`对象，然后进行消费。

消费结果：

- `CONSUME_SUCCESS`：本批消息成功；
- `RECONSUME_LATER`：失败消息将被发送至重试队列，稍后再投递；
- `ConsumeConcurrentlyContext#setAckIndex`：一批中前半部分成功、后半部分失败时，指定最后一条成功消息的下标。未设置时，失败结果默认整批重试。

常用参数：

| 参数 | 默认值（4.9.x） | 作用 |
| --- | ---: | --- |
| `consumeThreadMin` / `consumeThreadMax` | 20 / 20 | 并发消费线程池的核心/最大线程数。增大它不突破队列数限制，但可提高单队列多批消息的并发度。 |
| `consumeMessageBatchMaxSize` | 1 | 单次交给监听器的最大消息数。批量处理需自行处理部分失败语义。 |
| `consumeConcurrentlyMaxSpan` | 2000 | 同一 `ProcessQueue` 内最大 offset 跨度；超过时暂停继续拉取，防止慢消息造成过多内存积压。 |
| `pullThresholdForQueue` | 1000 | 每队列本地缓存消息数阈值，超过后暂缓拉取。 |
| `pullThresholdSizeForQueue` | 100 MiB | 每队列本地缓存消息体积阈值，超过后暂缓拉取。 |

源码入口：`ConsumeMessageConcurrentlyService.submitConsumeRequest`、`ConsumeMessageConcurrentlyService.ConsumeRequest.run`。

### 顺序消费

**注: 广播消费模式不支持顺序消息**

顺序消费对应 `MessageListenerOrderly` 与 `ConsumeMessageOrderlyService`。它保证的是**同一个 `MessageQueue` 内的顺序**，不保证不同队列之间的全局顺序。

完整的顺序需要生产者和消费者共同满足：生产者必须把同一业务键（例如订单 ID）的消息通过 `MessageQueueSelector` 投递到同一个队列；消费者再使用顺序监听器消费这个队列。只做其中一边都无法得到业务顺序。

```text
订单 1001: 创建 → 支付 → 发货
生产端按订单号选择 Q3 → Q3 内 FIFO 存储 → 消费端顺序处理 Q3
```

客户端会将消息以 `queueOffset` 为 key 放入 `ProcessQueue.msgTreeMap`。同一队列尽可能只提交一个消费任务；消费任务从最小 offset 开始获取消息。集群模式下，消费者还需要向 Broker 获取该队列的消费锁，避免重平衡交接时两个消费者同时顺序处理。

消费结果：

- `SUCCESS`：提交本批成功消息的最大 offset + 1；
- `SUSPEND_CURRENT_QUEUE_A_MOMENT`：暂停当前队列一段时间，失败消息重新放回本地 `ProcessQueue`，后续消息不能越过它；
- 抛异常通常等同于稍后重试当前队列。

常用参数：

| 参数 | 默认值（4.9.x） | 作用 |
| --- | ---: | --- |
| `consumeThreadMin` / `consumeThreadMax` | 20 / 20 | 多个队列可并行消费；同一个队列仍会串行。 |
| `consumeMessageBatchMaxSize` | 1 | 单次顺序回调的消息上限。批次越大，失败时阻塞后续消息的范围越大。 |
| `suspendCurrentQueueTimeMillis` | 1000 ms | `SUSPEND_CURRENT_QUEUE_A_MOMENT` 后当前队列的默认暂停时间，可通过 `ConsumeOrderlyContext` 覆盖本次结果。 |
| `maxReconsumeTimes` | `Integer.MAX_VALUE` | 顺序消费最大重试次数；超过后才转死信队列并放行后续消息。应按业务设定有限值，避免毒消息无限阻塞。 |

源码入口：`ProcessQueue.putMessage`、`ProcessQueue.takeMessages`、`ConsumeMessageOrderlyService.ConsumeRequest.run`、`ConsumeMessageOrderlyService.processConsumeResult`。

## 定时（延时）消息

RocketMQ 4.x不支持任意精度的延时消息, 目前只有18个等级
1s 5s 10s 30s 1m... 10m 20m 30m 1h 2h
完整的等级见org.apache.rocketmq.store.config.MessageStoreConfig.messageDelayLevel

生产者发送延迟消息时，其实只是发送到一个内部名为**SCHEDULE_TOPIC_XXXX**的主题中(到达broker后，由broker来替换的)，
这个主题有18个队列(分区)，刚好对应延迟消息的18个等级，后台定时任务会扫描这些队列，判断它有没有到期，若到期则投递
到真正的主题中。
注意:
多master broker集群，如brokera、brokerb、brokerc每个都会有18个队列，每个 Broker 都维护自己的延迟队列
不能设计成共享的，因为消息本来就是按照主题分布在多个队列中，如果消息本身路由到brokerb中，延迟等级设置为1，如果是
共享的，brokera(1-6)，brokerb(7-12), brokerc(13-18)，此时brokerb就没有延迟等级为1的队列了，没法进行投递。

如何判断是否到期:
consumequeue文件每行为20个字节，commitLogOffset(8)、messageSize(4)、tagHashCode(8)
本来tagHashCode只需要4个字节(java的hashcode方法返回int)，就是因为延迟消息使用了tagHashCode这个部分来
存储消息到期时间戳，所以给这部分设计成了8个字节。

定时任务行为:
1. 每个队列都对应一个定时任务
2. 同一延迟等级都发往了同一个队列，这样队列中的消息就是天然有序的(取决于到达broker的时间)，定时任务只需要扫描队头的元素有没有到期。
若没有到期，则会计算时间直接睡眠消息到期时再执行，而不是周期性扫描(每秒一次)，因此CPU开销会很低。

## 事务消息

事务消息采用二阶段提交方式，结合本地事务执行器来保证消息发送与本地事务的最终一致性：
 1. 生产者向Broker发送半消息（Half Message），此时消息对消费者不可见
 2. Broker存储半消息后向生产者返回确认，生产者开始执行本地事务
 3. 根据本地事务执行结果，生产者向Broker提交二次确认（Commit或Rollback）：
    - COMMIT：Broker将半消息投递给消费者
    - ROLLBACK：消费者不会收到该消息
    - UNKNOW：本地事务状态未知，等待Broker消息回查
 4. 如果Broker长时间未收到二次确认（如生产者宕机），会主动回查本地事务状态，
    生产者需要实现checkLocalTransaction方法来返回本地事务状态

 5. 消息回查机制说明
    - 消息回查频率：默认值60s，由参数transactionCheckInterval=60000控制
    - 消息回查最大次数：默认值15，由参数transactionCheckMax=15控制
    - transactionCheckInterval以及transactionCheckMax参数需要在Broker的broker.conf文件配置
    - 生产者端，可以单独对消息设置首次回查的免疫时间：
      msg.putUserProperty(MessageConst.PROPERTY_CHECK_IMMUNITY_TIME_IN_SECONDS, "15");
      默认15秒，半消息发送后多久才开始回查，这个值决定 Broker从发送半消息起等待多长时间才开始第一次回查。

 6. RMQ_SYS_TRANS_HALF_TOPIC：半消息主题，生产者发送事务消息时，Broker会先将消息存入该主题，此时不会被消费者消费。
    RMQ_SYS_TRANS_OP_HALF_TOPIC：操作记录主题，无论事务最终是提交 (Commit) 还是回滚 (Rollback)
    Broker都会在此记录一个OP消息（包含原消息的物理偏移量），用于标识该半消息已被处理。
    后台的回查服务会扫描半消息主题，并利用操作记录主题过滤掉已处理的消息

典型代码形态：

```java
TransactionMQProducer producer = new TransactionMQProducer("order_tx_group");
producer.setTransactionListener(new TransactionListener() {
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // 执行业务本地事务；提交、回滚或暂时 UNKNOWN
        return LocalTransactionState.COMMIT_MESSAGE;
    }

    public LocalTransactionState checkLocalTransaction(Message msg) {
        // 根据持久化的业务事务记录做幂等判定
        return LocalTransactionState.COMMIT_MESSAGE;
    }
});
producer.sendMessageInTransaction(message, arg);
```

## 消息拉取与过滤：Push 模式的真实工作方式

`DefaultMQPushConsumer` 不会被 Broker 通过独立通道“主动推送”消息。客户端为每个分配到的 `MessageQueue` 创建 `PullRequest`，不断发起拉取；若没有消息，Broker 将请求挂起（长轮询），有新消息或到达超时时才返回。因此它兼具拉模式的流量控制能力与推模式的使用体验。

```text
Rebalance 创建 PullRequest
        ↓
PullMessageService 调度拉取
        ↓
DefaultMQPushConsumerImpl.pullMessage 发起 Pull
        ↓
Broker 有消息：立即返回；无消息：挂起请求等待新消息/超时
        ↓
客户端过滤、放入 ProcessQueue、提交消费线程池，再继续 Pull
```

### 流控与长轮询参数

| 参数 | 默认值（4.9.x） | 作用 |
| --- | ---: | --- |
| `pullBatchSize` | 32 | 一次 Pull 最多拉取的消息条数。 |
| `pullInterval` | 0 ms | 一次 Pull 完成后额外等待多久再发起下一次；0 表示立即继续。 |
| `consumerPullTimeoutMillisWhenSuspend` | 30000 ms | Push 客户端发起长轮询 Pull 的超时时间。 |
| `brokerSuspendMaxTimeMillis` | 15000 ms | Broker 无消息时，单次 Pull 最长挂起时间。 |
| `pullThresholdForQueue` / `pullThresholdSizeForQueue` | 1000 / 100 MiB | 客户端 `ProcessQueue` 本地缓存数/体积上限，用于反压。 |
| `consumeConcurrentlyMaxSpan` | 2000 | 并发消费时本地 offset 跨度过大则暂缓拉取。 |

当消费慢于拉取时，客户端不是无限堆内存，而是通过上述阈值延迟下一次 Pull；但阈值只是一道保护，根治积压仍应提升消费能力、优化耗时或增加合理的队列数。

### 消息过滤

消费者通过 `subscribe(topic, expression)` 订阅，并可指定表达式类型。过滤的目标是减少无关消息从 Broker 传到消费者，而不是替代业务鉴权。

#### Tag 过滤

```java
consumer.subscribe("order_topic", "PAY || REFUND");
// consumer.subscribe("order_topic", "*"); // 订阅全部 Tag
```

Tag 是消息的一级分类。Broker 在 ConsumeQueue 中维护 tag hash，能高效完成初筛；由于可能存在hash冲突，客户端还要通过tag再过滤一次。

## 消息重试

重试的前提是“业务处理失败”。无论并发还是顺序消费，监听器都不应因为可预期的业务拒绝、参数错误等不可恢复问题无限返回失败；应根据业务决定记录、补偿或进入人工处理。

### 并发消费的重试

**注：广播模式不会进行重试**

并发监听器返回 `RECONSUME_LATER`（或抛异常）后，客户端不会原地阻塞当前业务队列，而是将失败消息发送回 Broker 的重试 Topic：`%RETRY%<consumerGroup>`。Broker 按延时级别稍后再将它投递给同一消费者组。

**并发消费的消息重试本质上是通过延时消息实现的, RocketMQ会将消息发送到消费者的重试队列中, 并且设置延时等级**

```text
业务 Topic Q0 中消息消费失败
        ↓
发送至 %RETRY%group，并设置下一次延时级别
        ↓
延时到期后重新投递给 group
        ↓
超过最大次数 → %DLQ%group（死信队列）
```

默认最大重试次数为 16 次（`maxReconsumeTimes=-1` 时采用默认值）。每次失败的下一次重试延时会逐级增加，最终受 Broker `messageDelayLevel` 的最高级别约束。可用 `ConsumeConcurrentlyContext#setDelayLevelWhenNextConsume` 覆盖下一次重试的延时级别；应谨慎使用，避免固定短延时导致失败风暴。

原业务队列的 offset 在消息成功、或失败消息已成功交给重试/DLQ 流程后会继续推进，因此后续消息不需要等待这一条失败消息。这也意味着“原队列 offset 已推进”不等于业务已经成功完成。

关键入口：`ConsumeMessageConcurrentlyService.processConsumeResult`、`DefaultMQPushConsumerImpl.sendMessageBack`、Broker `SendMessageProcessor.consumerSendMsgBack`。

### 顺序消费的重试

**注：广播模式直接不支持顺序消费，因而没有所谓的重试**

顺序监听器返回 `SUSPEND_CURRENT_QUEUE_A_MOMENT`（或异常）后，失败消息会重新插入本地 `ProcessQueue` 的有序 Map；当前队列暂停后从这条失败消息开始重试。它不会像并发消费那样先让后续消息通过重试 Topic 绕行，因此该队列之后的消息都会被阻塞，直到当前消息成功，或重试达到上限并进入死信队列。

默认 `maxReconsumeTimes` 为 `Integer.MAX_VALUE`，本意是尽量不破坏顺序，但生产环境通常应显式设定有限次数，并建立死信告警与人工补偿。否则一条毒消息会长期阻塞该分区的所有业务。

| 对比项 | 并发消费 | 顺序消费 |
| --- | --- | --- |
| 失败后存放位置 | `%RETRY%group` 延时重试队列 | 当前客户端 `ProcessQueue` 中原地重试 |
| 后续消息 | 可以继续消费 | 同一队列被阻塞 |
| 默认最大重试次数 | 16 | `Integer.MAX_VALUE` |
| 最终去向 | 超限进入 `%DLQ%group` | 超限进入 `%DLQ%group` 后放行后续消息 |

死信消息不会自动恢复到业务流程。应监控 `%DLQ%<consumerGroup>` 的堆积，定位根因后，通过独立消费者或运维工具做审计、修复和补偿；不要直接清空。

## 消费进度（offset）

offset 表示消费者组在某个 `MessageQueue` 上**下一条应拉取的消息位置**。例如已成功完成 0～9，提交 offset 为 10。它不是全局单值，而是以队列为粒度维护。

### 本地更新与 Broker 持久化并不是同一件事

消费结果完成后，客户端先调用 `OffsetStore.updateOffset` 更新本地 `offsetTable`；这一步只修改消费者内存，并不会马上请求 Broker。`MQClientInstance.startScheduledTask` 注册的定时任务会调用 `persistAll`，默认每 **5 秒**将集群消费的本地 offset 批量同步到 Broker。

```text
消费完成 / 成功交给重试流程
        ↓
ProcessQueue 计算可安全推进的 offset
        ↓
OffsetStore.updateOffset：只更新本地 offsetTable
        ↓  默认每 5 秒
RemoteBrokerOffsetStore.persistAll：提交到 Broker
        ↓
Broker ConsumerOffsetManager 持久化 consumerOffset.json
```

相关配置与实现：

| 相关代码 | 说明 |
| --- | --- |
| `persistConsumerOffsetInterval` | 客户端定时持久化间隔，4.9.x 默认 5000 ms。 |
| `RemoteBrokerOffsetStore` | 集群消费使用，向 Broker 提交 offset。 |
| `LocalFileOffsetStore` | 广播消费使用，每个客户端本地保存 offset。 |
| `ConsumerOffsetManager` | Broker 维护集群消费 offset，持久化文件通常为 `consumerOffset.json`。 |

这个 5 秒窗口是重复消费的重要来源：消费者已完成业务处理但还没来得及把最新 offset 持久化，就宕机或发生队列迁移时，新消费者会按 Broker 中较旧的 offset 重新拉取消息。

### 顺序消费和并发消费之间的差异

* 并发消费：不管是否消费成功，都会更新位点，并且使用`ProcessQueue`中`TreeMap`里最小的偏移量进行更新。并发消费的消费进度不能简单地提交“刚处理完的最大 offset”。例如 offset 8 先完成、offset 7 仍在执行，此时提交到 9，进程突然退出可能让 offset 7 永远被跳过。客户端会从 `ProcessQueue` 中找仍未处理消息的**最小 offset**，以此作为可安全提交的下一次拉取位置。
* 顺序消费：消费成功后或者超过最大重试次数进入到死信队列后才会更新消费位点，并且使用最大的偏移位点加+1。很好理解，因为顺序消费，只会有一个线程消费消息，不会出现小的偏移量消费失败而大的偏移量消费成功的场景。(进入死信队列除外)

### `ProcessQueue.dropped` 为什么不再更新 offset

重平衡时，一个不再归当前消费者负责的队列会先被标记为 `ProcessQueue.dropped=true`。这样做是为了防止旧消费者在队列已移交后继续拉取、继续提交进度，覆盖新消费者的状态。

消费线程可能已经在处理一批消息。即使该业务处理随后返回成功，`ConsumeMessageConcurrentlyService` / `ConsumeMessageOrderlyService` 在更新 offset 前都会检查 `processQueue.isDropped()`；若已 dropped，便不再更新**本地** offset，更不会由后续定时任务提交到 Broker。

这带来一个有意为之的结果：旧消费者可能已把消息处理成功，但进度未提交，新拥有者仍会重新消费它。RocketMQ 选择“允许重复”而不是“冒险跳过”，所以业务幂等不是可选项。

队列移除时，客户端还会尝试立即持久化移交前已经记录的 offset；但与正在执行的消费任务存在竞态，不能把它当作避免重复的保证。顺序消费额外依赖 Broker 队列锁交接来避免并行顺序消费，仍可能在交接后出现重复。

## 消息清理机制

消息是否删除主要由 Broker 的存储保留策略决定，**不以某个消费者是否已消费成功为条件**。即使所有消费者都消费完成，消息仍会保留到清理条件满足；反之，消费者长期落后超过保留窗口，历史消息也可能被清掉，导致无法继续按旧 offset 消费。

### 清理对象与触发条件

Broker 的消息主体顺序写入 `CommitLog` 文件（默认每个文件 1 GiB）；`ConsumeQueue` 和 Index 文件保存索引。清理以文件为单位，主要由两类条件触发：

1. **时间保留**：到配置的清理时间后，删除超过保留时长的旧 CommitLog 文件；
2. **磁盘水位**：磁盘使用率超过阈值时，即使未到保留时长也会加速清理，严重时强制清理以保护 Broker 可用性。

**如果只有一个CommitLog文件，是不会被删除的，此时存储的消息没有超过1G，没有发生滚动。**

常用 Broker 参数：

| 参数 | 默认值（4.9.x） | 作用 |
| --- | ---: | --- |
| `fileReservedTime` | 72 小时 | CommitLog 文件保留时长。消费者落后不能超过这个风险窗口。 |
| `deleteWhen` | `04` | 每天执行过期文件清理的小时（24 小时制）。 |
| `diskMaxUsedSpaceRatio` | 75% | 磁盘使用率达到该阈值时触发空间清理。 |
| `diskSpaceCleanForciblyRatio` | 85% | 达到后可更积极地强制清理。 |
| `diskSpaceWarningLevelRatio` | 90% | 磁盘严重告警水位；应在此之前扩容或转移流量。 |
| `cleanFileForciblyEnable` | `true` | 文件仍被引用且超过等待时间时，是否允许强制删除。 |
| `destroyMapedFileIntervalForcibly` | 120000 ms | 强制销毁仍被引用映射文件前的等待时间。 |

CommitLog 文件删除后，对应的 ConsumeQueue / Index 中失效条目也会由清理任务清除或截断。清理不是逐条打删除标记，而是删除完整的旧文件段，所以实际保留时间会受文件滚动、清理周期和磁盘压力影响，并不保证恰好等于 72 小时。

### 清理与消费的运维含义

- 消费延迟必须小于消息保留窗口，并应留出足够安全余量；恢复积压前先确认最早可用 CommitLog offset/时间。
- 不能把 Broker 当成永久归档系统。需要长期审计或可重放数据时，应同步落库、对象存储或数据仓库。
- 调大 `fileReservedTime` 会显著增加磁盘容量需求；只调大时间而不扩容，可能更快触发磁盘水位强制清理，效果适得其反。
- 清理前应优先处理异常积压、扩容磁盘、修复慢消费者；不要依赖调高磁盘阈值掩盖容量问题。

源码入口：`DefaultMessageStore.addScheduleTask`、`CleanCommitLogService`、`CleanConsumeQueueService`、`CommitLog.deleteExpiredFile`、`ConsumeQueue.deleteExpiredFile`。

