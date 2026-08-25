> 本文源码分析基于 RocketMQ 4.9.4。不同大版本的类名或调度细节可能略有变化，但核心模型一致。

## 什么是重平衡

RocketMQ 的一个 Topic 通常包含多个逻辑队列（`MessageQueue`，可理解为 `brokerName + topic + queueId`）。在**集群消费模式**（`MessageModel.CLUSTERING`）下，同一个消费者组会把这些队列分摊给组内消费者：一个队列在同一时刻只应由组内一个消费者拉取、消费。

重平衡（Rebalance）就是当“可消费的队列集合”或“消费者集合”发生变化时，消费者组重新计算“哪个消费者负责哪个队列”，并让本地的拉取、消费任务与新结果对齐的过程。

它解决的是**队列维度**的负载分配，而不是逐条消息的调度。因此有两个直接结论：

- 并行度上限主要由 Topic 的队列数决定。一个 Topic 有 8 个队列时，即使启动 20 个同组消费者，最多也只有 8 个消费者实际分到队列。
- 重平衡不能解决单个队列的热点问题。某个队列积压或耗时特别长时，它仍只能由一个消费者负责；需要从生产端的分区键、队列数和业务拆分上解决。

广播消费（`MessageModel.BROADCASTING`）不做组内分摊：每个在线消费者都会消费全部队列。所以本文重点讨论集群消费。

## 重平衡的参与者与目标

可以把一次集群消费的重平衡抽象为下面三个输入：

```text
Topic 的队列全集 MQAll
        +
消费者组的在线消费者全集 CIDAll
        +
分配策略 AllocateMessageQueueStrategy
        ↓
当前消费者应负责的队列集合 AllocateResultSet
```

例如，默认策略下，队列按 `brokerName、queueId` 排序，消费者按 `clientId` 排序后平均连续分配。8 个队列、3 个消费者时，可能得到：

| 消费者 | 分配到的队列 |
| --- | --- |
| C0 | Q0、Q1、Q2 |
| C1 | Q3、Q4、Q5 |
| C2 | Q6、Q7 |

没有中心节点专门下发“你消费哪些队列”的最终指令。每一个消费者客户端都会从 Broker 获取相同的消费者列表和路由信息，在本地以相同的策略计算结果；确定性排序是各客户端得出一致结论的前提。

## 哪些情况会让重平衡过程的结果发生变化

### 1. 消费者数量或存活状态变化

最常见的触发因素是同组消费者上线、正常关闭、进程崩溃或网络隔离。新消费者启动并向 Broker 发送心跳注册后，已有消费者需要让出一部分队列；消费者失联后，其原有队列会被其余在线消费者接管。

还要注意，分配单位是 `clientId`，而不是机器台数或你创建的 Java 对象数。若同组消费者因为 IP、`instanceName` 等配置而使用了相同的 `clientId`，Broker 会将它们视为同一个消费者，分配结果会异常。一个消费者组内的每个消费实例都应具有唯一、稳定的客户端标识。

### 2. Topic 队列集合变化

以下变化会改变 `MQAll`，从而需要重新分配：

- 新增或缩减 Topic 的读队列；
- 新增、下线 Broker，或 Broker 上该 Topic 的队列路由发生变化；
- NameServer 返回的 Topic 路由数据发生变化。

扩容队列只能提高**之后**的消费并行能力，已有消息仍留在原队列中；如果积压集中在旧队列，单纯加队列不会立刻均摊这部分历史积压。

### 3. 订阅关系或消费模式变化

消费者订阅/取消订阅 Topic

## 重平衡触发时机

### 真正执行入口

真正执行重平衡的入口类为`RebalanceService`，内部开启了一个单线程，**默认每20秒执行一次重平衡动作**，只要消费者数量并且逻辑队列没有发生变化，那么一次重平衡过程不会产生任何影响，是幂等的，可以重复执行。而且是在单线程中执行，不会发生并发执行。

**另外提供了`wakeup`方法来触发立即执行，原理就是如果线程处于20秒期间的沉睡期，立即唤醒线程来执行重平衡动作。**

`MQClientInstance`类提供了两个方法`rebalanceImmediately`以及`rebalanceLater`来主动触发重平衡，内部就是通过调用`rebalanceService.wakeup()`，基本根据这两个方法就能找到客户端所有主动触发的场景了。

### 消费者

- `在启动时`，消费者会立即向所有Broker发送一次发送心跳(HEART_BEAT)请求，Broker则会将消费者添加由ConsumerManager维护的某个消费者组中。然后这个Consumer自己会立即触发一次Rebalance。
- `在运行时`，消费者接收到Broker通知会立即触发Rebalance。(见下方broker)
- `当停止时`，消费者向所有Broker发送取消注册客户端(UNREGISTER_CLIENT)命令，Broker将消费者从ConsumerManager中移除，并通知其他Consumer进行Rebalance。(见下方broker)
- 周期性(默认每20秒)触发Rebalance，一来为了防止broker通知丢失，二来队列数量变化时broker不会主动通知，客户端会定期从NameServer拉取路由信息存储到本地，周期性的重平衡动作总会感知到，重平衡本身只会使用本地的路由信息进行分配。

### Broker

broker检测到消费者组信息发生变化时(消费者会向broker发送心跳信息，所以broker能感知到)，会通知消费者，调用`notifyConsumerIdsChanged`请求，客户端的`ClientRemotingProcessor`的`notifyConsumerIdsChanged`方法会响应该请求，立即触发重平衡。

## 重平衡会带来哪些影响

### 短暂停顿与积压波动

队列交接期间，旧消费者需要停止消息拉取、已消费消息的进度提交，新的消费者需要创建拉取任务；顺序消费还需要完成 Broker 锁的交接。因此吞吐会出现短暂抖动，监控上常表现为消费延迟上升、TPS 降低后恢复。频繁扩缩容、频繁网络闪断会把这种正常成本放大成持续的消费抖动。

Rebalance 只会影响发生迁移的那些 MessageQueue，在旧 Consumer 释放、新 Consumer 接管之间可能出现短暂的消费空档。

### 至少一次语义下的重复消费

重平衡不是事务性“瞬间移交”。某条消息可能已被旧消费者执行成功，但 offset 尚未提交，或提交结果尚未对新消费者可见；新消费者从已提交 offset 继续拉取时，就可能再次收到它。这是 RocketMQ 至少一次投递语义中的正常情况。

业务消费必须幂等：例如用业务唯一键去重、数据库唯一约束、状态机的条件更新，或把“是否已处理”与业务写入放在同一可靠事务边界内。不要依赖“重平衡通常很快”来规避重复。

#### 并发消费

重平衡时，messageQueue迁移过程是可能存在并发的，新的消费者和旧的消费者执行顺序是随机的。

当新的消费者先接管队列时，旧的消费者再释放队列时

旧消费者可细分为3种情况

已经消费完成(进度还未持久化到broker，消费完只会更新本地进度，定时任务每5秒更新至broker)：新消费者会重新消费

消费中的消息：新旧消费者可能同时消费，旧消费者消费完不会提交进度，因为对应的ProcessQueue对象已经dropped

在线程池的阻塞队列中等待消费的消息：旧消费者线程拿到消息时直接结束，不会再消费，因为对应的ProcessQueue对象已经dropped

如果旧消费者先释放新消费者再接管
已经消费完成(进度还未持久化到broker，消费完只会更新本地进度，定时任务每5秒更新至broker)：新消费者不会消费，因为旧消费者释放时会先持久会进度到broker中

消费中的消息：新旧消费者可能同时消费，旧消费者消费完不会提交进度，因为对应的ProcessQueue对象已经dropped

在线程池的阻塞队列中等待消费的消息：旧消费者线程拿到消息时直接结束，不会再消费，因为对应的ProcessQueue对象已经dropped

#### 顺序消费

顺序消费模式，由于新消费者接管队列时需要先向broker申请锁，因此必须要等到旧消费者先释放掉才行。旧消费者释放队列时，会先把ProcessQueue dropped掉，然后尝试加消费锁(消息消费时也会加这个锁)，加锁成功后就会向broker申请解锁，如果此时消费者正在消费这个ProcessQueue中的消息，则会在下一次重平衡中重试这个过程。

旧消费者可细分为3种情况

已经消费完成(进度还未持久化到broker，消费完只会更新本地进度，定时任务每5秒更新至broker)：新消费者不会消费，因为旧消费者释放时会先持久会进度到broker中

消费中的消息：新旧消费者不会发生同时消费现象，因为旧消费者占着broker的锁，新消费者无法接管这个队列，但是旧消费者消费完不会提交进度，因为对应的ProcessQueue对象已经dropped。所以新消费者在接管后会发生重复消费。

在ProcessQueue对象中TreeMap堆积的待消费的消息: 顺序消费是从ProcessQueue中的TreeMap中拿消息消费，此时会直接结束，因为对应的ProcessQueue对象已经dropped

## 实践建议

- **先规划队列数，再扩消费者。** 正常情况下，活跃消费者数不应长期超过可消费队列数；预留一定队列可为未来扩容留空间。
- **平滑扩缩容。** 分批启动或下线消费者，给每轮重平衡和积压恢复留出时间。优雅停机优于直接杀进程，可缩短重复消费和接管窗口。
- **保持组内配置一致。** 消费模式、订阅 Topic/过滤条件、分配策略、顺序/并发消费方式都应一致。部署前可把这些配置纳入启动参数校验。
- **把幂等作为必需能力。** 业务不能假设消息只会执行一次；重试、网络异常和重平衡都会带来重复投递的可能。
- **监控成员、分配和进度。** 除了整体 TPS，还应观察消费者在线数量、每个队列归属、各队列积压和消费 offset。`mqadmin consumerProgress`、`consumerConnection` 以及 Dashboard 都可辅助定位“消费者空闲但某些队列堆积”的问题。
- **谨慎调整 Topic 队列数。** 扩容无法重新分布已写入旧队列的消息；如果目标是消化旧热点积压，通常要结合生产端流量迁移、限流或业务侧处理优化。

## 一次重平衡具体做了什么

以 PushConsumer 的集群消费为例，核心过程如下：

```text
RebalanceService
  → MQClientInstance.doRebalance()
  → RebalanceImpl.doRebalance()
  → rebalanceByTopic()
       ├─ 查询 Topic 的 MessageQueue 集合(本地缓存)
       ├─ 向 Broker 查询本组 consumerId 列表
       ├─ 调用分配策略，得到本客户端的新队列集合
       └─ updateProcessQueueTableInRebalance()
            ├─ 移除不再归属自己的队列
            └─ 为新归属的队列创建 ProcessQueue 和 PullRequest
```

`ProcessQueue` 是消费者客户端对一个 `MessageQueue` 的本地处理缓存，保存已拉取但尚未完成处理的消息等状态；`PullRequest` 则代表该队列后续的拉取任务。它们是重平衡真正切换的本地资源，不是 Broker 中的逻辑队列本身。

### 失去队列时

对不再分配给自己的队列，客户端会先标记对应 `ProcessQueue` 为 `dropped`；随后持久化当前消费进度、清理本地 offset 缓存，并从 `processQueueTable` 中移除。设置为`dropped`后，拉取消息的线程会

顺序消费还会额外向 Broker 解锁该消息队列。新的拥有者必须获得该队列锁后才会开始顺序消费，从而避免两个消费者同时处理同一个顺序队列。

 `ProcessQueue` 为 `dropped`后会有以下几点影响

* 该队列对应拉取消息的的线程会检查这个状态，从而终止，不再拉取消息。
* 该队列正在消费中的消息消费完成后，不会再提交消费进度。
* 消费者消费线程也会也会检查这个状态，不会进行消费，因此消费者客户端已积压在本地内存但还未消费的消息在真正处理时会直接跳过。

### 获得新队列时

对新分配到的队列，客户端会清理可能残留的脏 offset，调用 `computePullFromWhereWithException` 计算起始拉取位点，创建 `ProcessQueue` 和 `PullRequest`，再把拉取请求投递出去。

起始位点优先使用消费者组在 Broker 持久化的该队列 offset。只有没有历史进度时，才会按照 `consumeFromWhere` 配置选择从最新位点、最早位点或指定时间点开始。因此，一个队列从消费者 A 转移到 B 并不意味着从头消费；但 A 已处理、尚未来得及提交 offset 的消息可能被 B 再次拉到。

### 默认分配策略

默认策略是 `AllocateMessageQueueAveragely`。它先对队列和消费者排序，之后尽量平均地为每个消费者分配连续的队列区间。队列数不能整除消费者数时，排在前面的消费者会多一个队列。

RocketMQ 还提供了环形平均、机房就近、一致性 Hash 等策略。选择策略时，最重要的不是“看起来分得更均匀”，而是**同一消费者组所有实例必须配置同一个策略，并且策略在相同输入下必须产生相同输出**。否则不同客户端会认为同一队列属于不同消费者，破坏消费排他性。

### 源码入口

下表列出阅读源码时最值得关注的类和方法：

| 位置                                                  | 职责                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| `RebalanceService.run`                                | 周期等待或被唤醒后发起重平衡。                               |
| `MQClientInstance.doRebalance`                        | 遍历当前客户端注册的消费者，逐个执行重平衡。                 |
| `RebalanceImpl.doRebalance`                           | 按订阅 Topic 进入广播或集群模式的分配逻辑。                  |
| `RebalanceImpl.rebalanceByTopic`                      | 获取队列、消费者 ID 列表，调用分配策略。                     |
| `RebalanceImpl.updateProcessQueueTableInRebalance`    | 对比新旧结果，回收旧 `ProcessQueue`、创建新 `ProcessQueue` 与 `PullRequest`。 |
| `RebalancePushImpl.removeUnnecessaryMessageQueue`     | 持久化 offset；顺序消费场景还负责解锁。                      |
| `RebalancePushImpl.computePullFromWhereWithException` | 根据已提交 offset 和消费起点策略计算新队列的拉取位置。       |
| `AllocateMessageQueueAveragely.allocate`              | 默认平均分配算法。                                           |

### 源码细节说明

RebalanceImpl.java

```java
/**
 * 按照主题进行分配
 * @param isOrder 消费者如果是顺序消费则是true, 并发消费则是false
 */
private void rebalanceByTopic(final String topic, final boolean isOrder) {
    switch (messageModel) {
        case BROADCASTING: {
            Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
            if (mqSet != null) {
                boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
                if (changed) {
                    this.messageQueueChanged(topic, mqSet, mqSet);
                    log.info("messageQueueChanged {} {} {} {}",
                        consumerGroup,
                        topic,
                        mqSet,
                        mqSet);
                }
            } else {
                log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
            }
            break;
        }
        case CLUSTERING: {
            // 从消费者客户端内存中获取该主题的所有队列信息
            Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
            // 发起请求从broker中获取消费者组下的所有消费者实例
            List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
            if (null == mqSet) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
                }
            }

            if (null == cidAll) {
                log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
            }

            if (mqSet != null && cidAll != null) {
                List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
                mqAll.addAll(mqSet);

                // 进行排序，确保所有消费者在调用分配算法时参数是一致的
                Collections.sort(mqAll);
                Collections.sort(cidAll);
				// 分配策略，默认是平均连续分配
                AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

                List<MessageQueue> allocateResult = null;
                try {
                    // 进行队列分配，返回的结果就是当前消费者被分配到队列集合
                    allocateResult = strategy.allocate(
                        this.consumerGroup,
                        this.mQClientFactory.getClientId(),
                        mqAll,
                        cidAll);
                } catch (Throwable e) {
                    log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
                        e);
                    return;
                }

                Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
                if (allocateResult != null) {
                    allocateResultSet.addAll(allocateResult);
                }
				/*
				 * 新的队列集合和旧的队列集合比较
				 * 1. 将不再分配给当前消费者的队列释放
				 * 2. 新的队列新增，分配一个新的ProcessQueue以及一个PullRequest用来拉取消息
				 * 3. 没有变化的队列不会产生任何影响
				 */
                boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
                if (changed) {
                    log.info(
                        "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
                        strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
                        allocateResultSet.size(), allocateResultSet);
                    this.messageQueueChanged(topic, mqSet, allocateResultSet);
                }
            }
            break;
        }
        default:
            break;
    }
}
```

RebalancePushImpl.java

```java
private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet, final boolean isOrder) {
    boolean changed = false;

    // 将不再分配给当前消费者的队列释放掉
    Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<MessageQueue, ProcessQueue> next = it.next();
        MessageQueue mq = next.getKey();
        ProcessQueue pq = next.getValue();

        if (mq.getTopic().equals(topic)) {
            if (!mqSet.contains(mq)) {
                // 标记dropped, 会影响到拉取消息的线程以及消费消息的线程
                pq.setDropped(true);
                /*
                 * 1. 持久化本地内存的消费进度到broker中
                 * 2. 清理掉本地内存消费进度
                 * 3. 如果是集群消费并且是顺序消费，会尝试向broker释放这个队列的锁，如果这个队列的消息正在消费中，是无法释放的，只能等下一次重平衡过程了
                 *    实现原理是消费者在消费消息时也会加一把锁，向broker申请释放队列锁时也会先申请这个锁，申请不到该方法返回false。
                 *    因为顺序消费一定要保证同一个队列不能由同一消费者组下的两个消费者同时消费，即便这个期间很短也不行。
                 * 4. 非顺序消费该方法永远返回true
                 */
                if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                    // 移除掉
                    it.remove();
                    changed = true;
                    log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
                }
            } else if (pq.isPullExpired()) {
                switch (this.consumeType()) {
                    case CONSUME_ACTIVELY:
                        break;
                    case CONSUME_PASSIVELY:
                        pq.setDropped(true);
                        if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                            it.remove();
                            changed = true;
                            log.error("[BUG]doRebalance, {}, remove unnecessary mq, {}, because pull is pause, so try to fixed it",
                                consumerGroup, mq);
                        }
                        break;
                    default:
                        break;
                }
            }
        }
    }

    // 处理新分配到的队列
    List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
    for (MessageQueue mq : mqSet) {
        if (!this.processQueueTable.containsKey(mq)) {
            // 顺序消费，对应上面释放逻辑，接管新队列时一定要等到旧消费者释放了向broker申请的锁才行
            if (isOrder && !this.lock(mq)) {
                log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
                continue;
            }

            this.removeDirtyOffset(mq);
            ProcessQueue pq = new ProcessQueue();

            long nextOffset = -1L;
            try {
                // 计算拉取消息时的偏移量, broker有记录就从broker获取，没有就按照消费者设置的来
                nextOffset = this.computePullFromWhereWithException(mq);
            } catch (Exception e) {
                log.info("doRebalance, {}, compute offset failed, {}", consumerGroup, mq);
                continue;
            }

            if (nextOffset >= 0) {
                ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
                if (pre != null) {
                    log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
                } else {
                    log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
                    PullRequest pullRequest = new PullRequest();
                    pullRequest.setConsumerGroup(consumerGroup);
                    pullRequest.setNextOffset(nextOffset);
                    pullRequest.setMessageQueue(mq);
                    pullRequest.setProcessQueue(pq);
                    pullRequestList.add(pullRequest);
                    changed = true;
                }
            } else {
                log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
            }
        }
    }
    // 将拉取消息请求放到拉取线程的阻塞队列中，拉取消息的线程会不断从队列中取请求，然后从broker拉取消息
    this.dispatchPullRequest(pullRequestList);

    return changed;
}

```

## 小结

RocketMQ 重平衡的本质是：所有消费者根据同一份队列集合、消费者集合和分配策略，在本地得出一致的队列归属，并据此交接 `ProcessQueue`、拉取任务和消费位点。它带来了横向扩展能力，也天然伴随短暂停顿与重复消费风险。

理解它时可以始终抓住三点：**队列是分配单元、offset 是接管起点、幂等是业务兜底**。这三点明确后，消费者扩缩容、队列扩容和顺序消费中的多数现象都更容易解释。
