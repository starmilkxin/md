## 消息持久性
从生产者的角度出发，可以选择同步发送消息

```Java
//处理事件
public void fireMessage(Message message) {
    //将事件发布到指定的主题
    try {
        kafkaTemplate.send("TOPIC_COMMENT", JSONObject.toJSONString(message)).get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
```

<br/>
又或者是异步发送消息

```Java
//处理事件
public void fireMessage(Message message) {
    kafkaTemplate.setProducerListener(new ProducerListener() {
        @Override
        public void onSuccess(ProducerRecord producerRecord, RecordMetadata recordMetadata) {
            // log.info("消息投递成功")
        }

        @Override
        public void onError(ProducerRecord producerRecord, RecordMetadata recordMetadata, Exception exception) {
            // log.error("消息投递失败");
        }
    });
    //将事件发布到指定的主题
    kafkaTemplate.send("TOPIC_COMMENT", JSONObject.toJSONString(message));
}
```

这里我们采用异步消息投递+回调的方法，因为异步投递可以减少响应时间，增强用户体验并能够抗住高并发。通过回调函数判断消息是否投递成功，如果失败则可以进行反复投递并通过日志记录。<br/>
为了防止消息在投递的过程中就丢失，所以我们在投递前要事先记录日志，这样就可以日志进行补偿。<br/>
<br/>
从Kafka的角度出发，Kafka默认的也是仅有的只有异步刷盘策略，如果在刷盘前Kafka就宕机的话那么数据还是会丢失的，我们可以通过设置Kafka的日志参数，使得它在消息数量很少时、短时间间隔时就进行一次刷盘，从而使其变成“同步刷盘”策略。<br/>
<br/>
从消费者的角度出发，我们要取消自动提交，为的就是防止消费者还未将消息消费完后，便宕机，此时却已经自动提交offset，所以重启Kafka后消费者会再次进行重复消费。

```Java
@KafkaListener(topics = {"TOPIC_COMMENT"})
public void handleComment(ConsumerRecord record, Acknowledgment ack) {
    if (record == null || record.value() == null) {
        logger.error("消息的内容为空");
        ack.acknowledge();
        return;
    }
    Message message = JSONObject.parseObject(record.value().toString(), Message.class);
    if (message == null) {
        logger.error("消息格式错误");
        ack.acknowledge();
        return;
    }
    messageService.addMessage(message);
    ack.acknowledge();
}
```

<br/>
至此，通过生产者->服务端->消费者，使得消息尽量能够不丢失。

## 消息顺序性
[https://blog.csdn.net/u011439839/article/details/90349596](https://blog.csdn.net/u011439839/article/details/90349596)

kafka的特性
1. kafka中，写入一个partion照片中的数据是一定有顺序的
2. kafka中一个消费者消费一个partion的数据，消费者取出数据时，也是有顺序的

需要顺序的场景
1. 数据库中的binlog
2. 一些业务需要，比如希望把某个订单的数据写入一个partion

为何消息会错乱
1. 由于消费者消费消息之后，消费之后，有可能交给很多个线程去处理数据（如下图），这样就导致数据顺序错乱

![消息错乱](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220604145117.png)


解决办法
+ 为了保证一个消费者中多个线程去处理时，不会使得消息的顺序被打乱，则可以在消费者中，消息分发至不同的线程时，加一个队列，消费者去做hash分发，将需要放在一起的数据，分发至同一个队列中，最后多个线程从队列中取数据，如下图所示。

![消息错乱解决方法](https://xin-xinblog.oss-cn-shanghai.aliyuncs.com/img/20220604145247.png)

## 避免重复消费问题
针对于消费者，会出现三种情况
+ 先消费后提交，可能会造成消息重复消费的问题
+ 先提交后消费，可能会造成消息丢失问题
+ 采用原子操作，使得消费和提交不会分离

第二种情况，可以通过日志记录的方式来进行补偿。<br/>
第一种情况的解决方法有：
1.  提高消费者的处理速度。例如：对消息处理中比较耗时的步骤可通过异步的方式进行处理、利用多线程处理等。在缩短单条消息消费的同时，根据实际场景可将max.poll.interval.ms值设置大一点，避免不必要的Rebalance。可根据实际消息速率适当调小max.poll.records的值。
2. 通过redis或数据库中存放已消费过消息的唯一key，来做到消息去重。