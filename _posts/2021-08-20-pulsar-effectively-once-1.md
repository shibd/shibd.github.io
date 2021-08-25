---
layout:     post
title:      Pulsar特性解析[Effectively once]
subtitle:   Pulsar特性解析[Effectively once]
date:       2021-08-20
author:     baozi
header-img: img/2019-08-15(Message-Center)/top2.jpg
catalog: true 						
tags:								
    - pulsar

---

## 前言

> https://www.splunk.com/en_us/blog/it/effectively-once-semantics-in-apache-pulsar.html

pulsar在文章中详细介绍了是如何支持Effectively once的，本文不再重复阐述，下面只把文章中描述的结论做总结。后面会进行源码解析。

为了实现Effectively once，pulsar从两个方面支持：

1. Effectively-once publishing：确保消息只发送一次
2. Effectively-once consumer: 确保消费只消费一次


## Effectively-once publishing

pulsar可以支持无论是在broker故障，生产者故障，网络故障等极端情况下保障消息只会在pulsar存储一份。主要依赖pulsar中的**message deduplication**功能保障。pulsar提供了从命名空间，topic等各个维度我开关来控制是否启动**message deduplication **

```shell
pulsar-admin namespaces set-deduplication $MY_NAMESPACE --enable
```

消息可靠的发给pulsar是通过：producer不断重试 + broker端的**message deduplication**来功能完成的。所以producer还需要设置不断重试配置。通过下面配置实现
```java
ProducerConfiguration conf = new ProducerConfiguration();
conf.setProducerName("my-producer");
conf.setSendTimeout(0, TimeUnit.SECONDS);
Producer producer = client.createProducer(TOPIC_NAME, conf);
```

**message deduplication**实现原理主要是依赖broker端维护了每个producer的highSequenceId，sequenceId是递增的，也可以由用户控制。每次消息到达broker时，都会根据是否小于当下highSequenceId来判断是否是重复消息。

关于**message deduplication**后面会有更详细的源码解析，这里不再过多阐述，这里要描述一个这种设计的局限性。（读者可以看完更详细的源码解析再来看该功能的局限性）

> Effectively-once publishing in practice only makes sense when the messages are coming from a replayable source as opposed to a non-replayable source (for example online HTTP requests). For non-replayable sources, there’s no way to re-send the previous pending messages after a crash.

pulsar为了更高性能的实现**message deduplication**，所以使用了sequenceId的设计，两个局限性：

1. 不能判断**无重放源的消息**(non-replayable source)去重：比如http请求，每次请求都是无状态随机的，并不能关联到sequenceId。
2. 只能判断最近一笔消息是否重复：pulsar的设计初衷就是为了应对producer与broker通行时各种故障下实现精确一次消息的生产，并不是为了解决业务消息幂等的。所以如果你的场景是有历史消息还可能重复投递，然后希望根据某个自定义id(idmpotentId)来让pulsar实现消息去重，那么pulsar是不支持的。

总结，pulsar利用sequenceId实现**message deduplication**性能是非常高的（只有一次hash和判断的损耗），快照以及持久化都是异步执行的。如果要支持上面两个功能，pulsar必然要维护一段时间内所有消息的messageId，并且还要设计如何高效的判断。

## Effectively-once consumer

pulsar只支持两种消费模式，subscribe和reader。

subscribe模式下，pulsar会保存consumer的消费位点，根据最新位点投递下一笔消息，用户消费完消息后，可以显明主动ack位点。
```java
Consumer consumer = client.subscribe(MY_TOPIC, MY_SUBSCRIPTION_NAME);

while (true) {
    Message msg = consumer.receive();
    // Process the message...
    consumer.acknowledge(msg);
}
```

对于subscribe模式来讲，有可能出现下面几种重复消费的情况：
1. broker故障：broker故障时，有可能用户消费了该数据并且处理，但是在ack时没有成功，那么broker恢复后会重新投递该笔消息。
2. consumer故障：同broker故障一样，消费了该笔消息并且处理，但是在ack之前consumer宕机，那么broker也会重新投递该笔消息。
3. 网络故障：网络超时等也会造成consumer提交ack时失败，broker重新投递消息。
4. 重复数据消费（特殊）：正如**Effectively-once publishing**结论中描述的局限性，有可能本身pulsar就存储了重复数据，那么即便没有上面三种故障的情况下，业务端也重复消费了数据。

针对上面三种故障，其中前3种故障可以使用pulsar的reader模式 + 依赖外部存储当下消费的offset即可解决。但面对本身的重复数据，想要做到幂等，则必须使用一个存储所有消息id的存储来完成。

reader模式，用户可以主动指定拉取从某个消息开始拉取，用户只需保存好当下消费到的位点即可，比如把lastMessageId的存储和业务状态修改在一个事务内提交。
```java
MessageId lastMessageId = recoverLastMessageIdFromDB();
Reader reader = client.createReader(MY_TOPIC, lastMessageId,
                                    new ReaderConfiguration());

while (true) {
    Message msg = reader.readNext();
    byte[] msgId = msg.getMessageId().toByteArray();

    // Process the message and store msgId atomically
}
```

综上，为了实现完全的消费者精确一次性消费，如果producer端不能保证发送的消息没有重复消息时，则需要consumer端使用一张大的幂等持久化状态存储来实现，当然这个幂等状态可以根据业务场景配置一定的淘汰机制。


## Message Deduplication源码解析

在上面**Effectively-once publishing**描述，我们知道pulsar利用维护producer的maxSequenceId来保障对于某个producer重试时消息的去重。这里简单对源码做解析。

所有的消息去重逻辑的实现都在MessageDeduplication类当中，每个PersistentTopic对象都持有一个MessageDeduplication对象。

### 如何判断是否是重复消息?

主要依赖两个集合判断:
```java
    @VisibleForTesting
    final ConcurrentOpenHashMap<String, Long> highestSequencedPushed = new ConcurrentOpenHashMap<>(16, 1);
    final ConcurrentOpenHashMap<String, Long> highestSequencedPersisted = new ConcurrentOpenHashMap<>(16, 1);
```
这两个集合存储了 produceName对应最大的seuenceId，一个是持久化的，一个是非持久化的，日常判断时都是通过非持久化的判断（高速），后台有个线程定期的打快照，最终判断是否是消息重复主要依赖持久化的。


PersistentTopic在接收到消息写入时，首先会调用MessageDeduplication#isDuplication来判断是否是重复消息。判断逻辑也很简单，下面为省略后代码。

```java
    public MessageDupStatus isDuplicate(PublishContext publishContext, ByteBuf headersAndPayload) {
        // Synchronize the get() and subsequent put() on the map. This would only be relevant if the producer
        // disconnects and re-connects very quickly. At that point the call can be coming from a different thread
        synchronized (highestSequencedPushed) {
            Long lastSequenceIdPushed = highestSequencedPushed.get(producerName);
            if (lastSequenceIdPushed != null && sequenceId <= lastSequenceIdPushed) {
                if (log.isDebugEnabled()) {
                    log.debug("[{}] Message identified as duplicated producer={} seq-id={} -- highest-seq-id={}",
                            topic.getName(), producerName, sequenceId, lastSequenceIdPushed);
                }

                // Also need to check sequence ids that has been persisted.
                // If current message's seq id is smaller or equals to the
                // lastSequenceIdPersisted than its definitely a dup
                // If current message's seq id is between lastSequenceIdPersisted and
                // lastSequenceIdPushed, then we cannot be sure whether the message is a dup or not
                // we should return an error to the producer for the latter case so that it can retry at a future time
                Long lastSequenceIdPersisted = highestSequencedPersisted.get(producerName);
                if (lastSequenceIdPersisted != null && sequenceId <= lastSequenceIdPersisted) {
                    return MessageDupStatus.Dup;
                } else {
                    return MessageDupStatus.Unknown;
                }
            }
            highestSequencedPushed.put(producerName, highestSequenceId);
        }
        return MessageDupStatus.NotDup;
    }      
```

可以看到一共返回有三种状态：
- MessageDupStatus.NotDup(非重复消息): 如果producer发送消息的的sequenceId**大于**维护的**内存**的highSequenceId，则一定是重复消息。PersistentTopic会继续执行后续存储步骤。
- MessageDupStatus.Dup(重复消息): 如果sequenceId < highSequenceId, 并且 sequenceId < highPersistentSequenceId，则一定是重复消息。PersistentTopic会返回确认是重复消息。
- MessageDupStatus.Unknown(未知状态): 如果sequenceId < highSequenceId 并且 sequenceId > highPersistentSequenceId，则是未知状态。PersistentTopic会抛出DupUnknownException来使producer端重试

出现Unknown状态因为highPersistentSequenceId集合和highSequenceId集合的维护时间点是不一样的：

- highSequenceId: 在每次判断结果是NotDup时，则进行highSequenceId集合的更新（消息持久化之前）。
- highPersistentSequenceId: 当实际把消息写入到bk之后，再更新highPersistentSequenceId集合（消息持久化之后）。

这种设计的初衷是因为pulsar的执行是异步化的，当前一笔消息判断完之后，如果该笔消息还没写入bk成功，下一笔消息再来，为了高并发的处理，这时不应该等待前一笔消息写入完再做该笔消息的判断，所以有了内存的集合和持久化的集合。

在大多数能写入bk都成功的情况下，highSequenceId和highPersistentSequenceId是能保持一致的，所以不会发生Unknown状态。在发生写入bk异常时，highPersistentSequenceId则不会更新，这时就会发生Unknown状态。PersistentTopic接收到Unknown以及Dup后则会调用MessageDeduplication#resetHighestSequenceIdPushed()方法来用highPersistentSequenceId覆盖highSequenceId集合来保持两个集合的一致性。

### MessageDeduplication状态是如何持久化的?

pulsar的每个broker是无状态的，如果某个broker挂机，那么该broker中负责的topic则会调度到另外可用的broker上运行。所以MessageDeduplication的状态应当是具备持久化的。MessageDeduplication中主要需要持久化的状态是：highestSequencedPersisted集合。

broker在启动时会根据用户的配置启动一个定时线程调用MessageDeduplication#takeSnapshot方法来进行状态快照的持久化。状态是写入bk当中的，使用了ManagedCursor的properties元数据存储。

```java
    private void takeSnapshot(PositionImpl position) {
        if (log.isDebugEnabled()) {
            log.debug("[{}] Taking snapshot of sequence ids map", topic.getName());
        }
        Map<String, Long> snapshot = new TreeMap<>();
        highestSequencedPersisted.forEach((producerName, sequenceId) -> {
            if (snapshot.size() < maxNumberOfProducers) {
                snapshot.put(producerName, sequenceId);
            }
        });

        managedCursor.asyncMarkDelete(position, snapshot, new MarkDeleteCallback() {
            @Override
            public void markDeleteComplete(Object ctx) {
                if (log.isDebugEnabled()) {
                    log.debug("[{}] Stored new deduplication snapshot at {}", topic.getName(), position);
                }
                lastSnapshotTimestamp = System.currentTimeMillis();
            }

            @Override
            public void markDeleteFailed(ManagedLedgerException exception, Object ctx) {
                log.warn("[{}] Failed to store new deduplication snapshot at {}", topic.getName(), position);
            }
        }, null);
    }
```

那么问题来了，既然状态是异步持久化的，pulsar是如何保证未持久化的状态在飘逸后可以正确恢复呢？

在broker启动时，首先会从cursor中读取存储的最新状态，然后会从该状态对应的position开始，重新消费到ledger的最新position，然后来保障恢复到该topic下每个producer最新的sequenceId。具体代码可以参考replayCursor方法
```java
    private CompletableFuture<Void> recoverSequenceIdsMap() {
        // Load the sequence ids from the snapshot in the cursor properties
        managedCursor.getProperties().forEach((k, v) -> {
            highestSequencedPushed.put(k, v);
            highestSequencedPersisted.put(k, v);
        });

        // Replay all the entries and apply all the sequence ids updates
        log.info("[{}] Replaying {} entries for deduplication", topic.getName(), managedCursor.getNumberOfEntries());
        CompletableFuture<Void> future = new CompletableFuture<>();
        replayCursor(future);
        return future;
    }
```

## 总结

Pulsar对于实现Effectively once语义是需要用户配合外部存储来完成的，Pulsar只是提供了api以及最佳解决方案。在生产者端，Pulsar通过维护producer对应highSequenceId的关系来实现生产者去重，可以解决具有会溯源的producer的生产者幂等。如果需要Effectively once语义的保证，需要根据具体的业务场景做合适的解决方案。

业务场景1：具有可会溯源的生产者

比如，producer端的数据是从文件中读的，可以使用sequenceId来保证生产者幂等。那么可以使用  producer message deduplication + consumer reader模式，这样consumer端只需要依赖外部存储当下消费的lastMessageId即可。

业务场景2：不具有可会溯源的生产者

比如，producer端的数据是从http请求发送的，那么则不能使用pulsar producer message deduplication，所以consumer端需要依赖外部存储存储所有的messageId（业务属性的），从而实现Effectively once语义。

## 推荐阅读

- [Apache Pulsar 如何保证消息不丢不重？](https://mp.weixin.qq.com/s/WhZq1o12OxuMdtSV2lEf-A)
- [Effectively-Once Semantics in Apache Pulsar](https://www.splunk.com/en_us/blog/it/effectively-once-semantics-in-apache-pulsar.html)
- [TOAB-Bookkeeper](https://www.splunk.com/en_us/blog/it/scaling-out-total-order-atomic-broadcast-with-apache-bookkeeper.html)
    