# 主流程

构建完 workThread 后，会首先创建 shuffleTopic：

```java
@Override
public void run() {
    try {
        this.planetaryEngine.start();

        this.planetaryEngine.runInLoop();
    } catch (Throwable e) {
        logger.error("worker thread=[{}], error:{}.", this.getName(), e);
        throw new RStreamsException(e);
    } finally {
        logger.info("worker thread=[{}], engin stopped.", this.getName());
        this.planetaryEngine.stop();
    }
}

void start() throws Throwable {
    createShuffleTopic();

    this.unionConsumer.start();
    this.producer.start();
    this.stateStore.init();
}
```

之后在 runInLoop 方法里循环拉取 mq 里的消息，并且每10毫秒处理消息，并通过 persist 进行持久化状态，以及提交消息。

其中，在 persist方法 和 recover 方法中的 loadState 方法， 都有 createStateTopic 方法可以同时构建 shuffleTopic 和 shuffleTopic-stateTopic 这两个 topic 以及它们的队列。

当有非 shuffleTopic 消息时，会递归处理，直到 SinkSupplier 将处理后的消息发送到对应的 shuffleTopic。

当有 shuffleTopic 消息时，会递归处理，通过 stateStore 获取对应 key 的状态，之后根据算子运算，并临时储存在 stateStore 中。

最后输出计算后的状态。

之后 persist 方法中，因为 calculating 中存在该 stateTopicQueueKey，所以会将 rocksDBStore 中的数据发送到 MQ 来持久化状态。并且 commit shuffleTopic 的消息。

# rebalance

RocketMQ 定期会进行 rebalance：

```java
    public void run() {
        this.log.info(this.getServiceName() + " service started");

        while(!this.isStopped()) {
            this.waitForRunning(waitInterval);
            this.mqClientFactory.doRebalance();
        }

        this.log.info(this.getServiceName() + " service end");
    }
    
    public void doRebalance() {
        Iterator var1 = this.consumerTable.entrySet().iterator();

        while(var1.hasNext()) {
            Entry<String, MQConsumerInner> entry = (Entry)var1.next();
            MQConsumerInner impl = (MQConsumerInner)entry.getValue();
            if (impl != null) {
                try {
                    impl.doRebalance();
                } catch (Throwable var5) {
                    log.error("doRebalance exception", var5);
                }
            }
        }

    }
```

如果是 shuffle_topic 需要 rebalance 的话，则会执行 recover 方法：

```java
//从shuffle topic中读出的数据才能进行有状态计算。
if (topic.endsWith(Constant.SHUFFLE_TOPIC_SUFFIX)) {
    Throwable throwable = this.recoverHandler.apply(addQueue, removeQueue);
    if (throwable != null) {
        throw new RuntimeException(throwable);
    }
    logger.info("recover messageQueue finish, addQueue: [{}], removeQueue:[{}].", addQueue, removeQueue);
}

this.wrapper.setRecoverHandler((addQueue, removeQueue) -> {
    try {
        PlanetaryEngine.this.stateStore.recover(addQueue, removeQueue);
        return null;
    } catch (Throwable e) {
        logger.error("recover error.", e);
        return e;
    }
});
```

对于每个 addQueue，还会通过 topologyBuilder 构建它的 processor：

```java
private void buildTask(Set<MessageQueue> addQueues) {
    for (MessageQueue messageQueue : addQueues) {
        String key = Utils.buildKey(messageQueue.getBrokerName(), messageQueue.getTopic(), messageQueue.getQueueId());
        if (!mq2Processor.containsKey(key)) {
            Processor<?> processor = topologyBuilder.build(messageQueue.getTopic());
            this.mq2Processor.put(key, processor);
        }
    }
}
```

recover 方法中的 loadState 为状态重放，从被迁移状态队列拉取数据到本地进行重放，需要从队列头开始消费，相同Key的数据只保留offset最大的数据，形成K-V状态对，放入本地临时存储RocksDB中：

```java
@Override
public void recover(Set<MessageQueue> addQueues, Set<MessageQueue> removeQueues) throws Throwable {
    this.loadState(addQueues);
    this.removeState(removeQueues);
}
```

# rocksdb的使用

rebalance 时，状态重放 loadState 会将 mq 中存储的 kv 状态对，放入 rocksdb 中。removeState 会将删除的队列对应的状态从 rocksdb 中删除。

persist 时，根据 staticTopicQueue 获取其正在计算的 keySet，再根据 keySet 从 rocksdb 中获取对应的 value，发送消息给 mq 用以持久化。

AccumulatorProcessor，AggregateProcessor 的 process 方法从 stateStore 中获取 key 对应的状态 value，处理后再将 value 存入 stateStore 中。

WindowAggregateProcessor process 方法先执行了 fireWindowEndTimeLassThanWatermark 方法，之后构造相应窗口，从 windowsStore 中根据 windowKey 获取旧状态，处理后再存入 windowStore中，最后再执行一次 fireWindowEndTimeLassThanWatermark 方法。其中 fireWindowEndTimeLassThanWatermark 方法將 windowStore 中过期窗口进行处理，并从 windowStore 中删除状态。

SessionWindowAggregateProcessor 的 process 方法通过 fireIfSessionOut 使用前缀查询从 windowStore 中找到 session state, 触发已经 session out的 watermark，再删除状态。最后处理 newSessionWindowTime 并将状态存入 windowsStore 中。

JoinStreamAggregateProcessor process 方法，通过 store 方法在 stateStore 中存储左流/右流数据，再通过 fire 方法从 stateStore 中获取并处理左流与右流数据。

JoinStreamWindowAggregateProcessor process 方法，通过 store 方法分别在 leftWindowStore 和 rightWindowStore 中存储左流与右流状态。在 fire 方法中将过期的左流与右流窗口进行匹配触发，最后再将左流/右流过期窗口的状态删除。

我有三个问题想要请教一下老师：

1. 《RocketMQ Streams 拓扑构建与数据处理过程》这篇文章中所提到的 “状态topic的队列数量等于source topic队列数量”，其中 “状态topic” 是否指的是 shuffleStateTopic，其中 “source topic” 是否指的是 shuffleTopic？

2. 《RocketMQ Streams 拓扑构建与数据处理过程》这篇文章中还提到了 “RocketMQ中队列数会随着Broker扩缩容而增加或者减少”，这个是 RoecketMQ 会自动对 topic 的队列数进行扩缩容的吗？那么是不是就是主动设置 topic 的队列数后就不会被自动改变了呢？

3. 《RocketMQ Streams 拓扑构建与数据处理过程》这篇文章中所提到的 “状态topic创建为Static topic（Logic Queue），即状态topic即是Compact topic也是Static topic。这样才能解耦队列数量与Broker数量，使队列数量在扩缩Broker情况下仍然不变，保证含有相同Key的数据能被发送到同一队列中。” 这个具体的做法是否为：将 shuffleTopic 和 shuffleStateTopic 两者的队列都设定为固定值？

另外我阅读了源码，项目中使用了本地 RocksDB 作为状态存储的地方有：

1. rebalance 时，状态重放 loadState 会将 mq 中存储的 kv 状态对，放入 rocksdb 中。removeState 会将删除的队列对应的状态从 rocksdb 中删除。

2. persist 时，根据 staticTopicQueue 获取其正在计算的 keySet，再根据 keySet 从 rocksdb 中获取对应的 value，发送消息给 mq 用以持久化。

3. watermark 方法，构建了 watermark_key 将 stateTopicMessageQueue 的 watermark 存入 rocksdb 中。  

4. AccumulatorProcessor，AggregateProcessor 的 process 方法从 stateStore 中获取 key 对应的状态 value，处理后再将 value 存入 stateStore 中。

5. WindowAggregateProcessor process 方法先执行了 fireWindowEndTimeLassThanWatermark 方法，之后构造相应窗口，从 windowsStore 中根据 windowKey 获取旧状态，处理后再存入 windowStore中，最后再执行一次 fireWindowEndTimeLassThanWatermark 方法。其中 fireWindowEndTimeLassThanWatermark 方法將 windowStore 中过期窗口进行处理，并从 windowStore 中删除状态。

6. SessionWindowAggregateProcessor 的 process 方法通过 fireIfSessionOut 使用前缀查询从 windowStore 中找到 session state, 触发已经 session out的 watermark，再删除状态。最后处理 newSessionWindowTime 并将状态存入 windowsStore 中。

7. JoinStreamAggregateProcessor process 方法，通过 store 方法在 stateStore 中存储左流/右流数据，再通过 fire 方法从 stateStore 中获取并处理左流与右流数据。

8. JoinStreamWindowAggregateProcessor process 方法，通过 store 方法分别在 leftWindowStore 和 rightWindowStore 中存储左流与右流状态。在 fire 方法中将过期的左流与右流窗口进行匹配触发，最后再将左流/右流过期窗口的状态删除。

目前我初步的想法是：

1. 因为 rocksdb 的设计，只有旧 WAL 中所有的 CF 都被 flush了，旧 WAL 才会被真正删除。因此 CF 的设计不应过多，且每个 CF 的访问模式应该趋同。
2. CF 的目的主要为单个读多的场景服务，在写多和遍历读场景下 CF 效果没那么明显。
3. CF 分为 watermark，windows，和非 windows。分割 watermark，windows 的状态 和非 windows 的状态。
4. CF 细粒度到每一个 processor， 分割每一个涉及到状态存储读取的 processor。