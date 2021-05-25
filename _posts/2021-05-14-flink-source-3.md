---
layout:     post
title:      Flink源码解析[Source](三) - 旧版Data Sources详解&源码
subtitle:   Flink源码解析[Source](三) - 旧版Data Sources详解&源码
date:       2021-05-14
author:     baozi
header-img: img/2019-08-15(Message-Center)/top2.jpg
catalog: true 						
tags:								
    - flink


---

## 概述

本系列文章是旨在熟悉摸头flink的source-connect原理，希望可以做到自己可以实现一个新的source，代码解析将会以kafka的实现配合flink的api为主线解析。

>  flink版本为1.12.0

**第一篇**：[为什么要解析Source源码](https://shibd.github.io/2021/05/07/flink-source-1/)<br>
**第二篇**：[如何创建Flink kafka source](https://shibd.github.io/2021/05/11/flink-source-2/)<br>
**第三篇**：[旧版Data Source详解&源码](https://shibd.github.io/2021/05/14/flink-source-3/)<br>


## FlinkPulsarSource

以为PulsarSource为例，实现一个Source需要实现Flink提供的下面几个接口

![image-20210511211516947](/img/2021-flink-source/003.png)

下面将分别介绍其各个接口作用以及调用时机，并拿`FlinkPulsarSource`为例介绍分别在各个接口中做了哪些事情。


## SourceFunction
> org.apache.flink.streaming.api.functions.source.SourceFunction

SourceFunction一共有两个接口，`run`方法会在job启动时被Flink调用，并把`SourceContext`传给实现类。用户可以根据`SourceContext`发送拉取下来的消息(`collect`)。

```java
public interface SourceFunction<T> extends Function, Serializable {
    
    void run(SourceContext<T> ctx) throws Exception;

    void cancel();

}

```

`FlinkPulsarSource`主要在run方法中创建`Fetcher`并启动。
```java
    @Override
    public void run(SourceContext<T> ctx) throws Exception {
        if (ownedTopicStarts.isEmpty()) {
            ctx.markAsTemporarilyIdle();
        }

        log.info("Source {} creating fetcher with offsets {}",
                taskIndex,
                StringUtils.join(ownedTopicStarts.entrySet()));

        // from this point forward:
        //   - 'snapshotState' will draw offsets from the fetcher,
        //     instead of being built from `subscribedPartitionsToStartOffsets`
        //   - 'notifyCheckpointComplete' will start to do work (i.e. commit offsets to
        //     Pulsar through the fetcher, if configured to do so)

        StreamingRuntimeContext streamingRuntime = (StreamingRuntimeContext) getRuntimeContext();

        this.pulsarFetcher = createFetcher(
                ctx,
                ownedTopicStarts,
                periodicWatermarkAssigner,
                punctuatedWatermarkAssigner,
                streamingRuntime.getProcessingTimeService(),
                streamingRuntime.getExecutionConfig().getAutoWatermarkInterval(),
                getRuntimeContext().getUserCodeClassLoader(),
                streamingRuntime);

        if (!running) {
            return;
        }

        if (discoveryIntervalMillis < 0) {
            pulsarFetcher.runFetchLoop();
        } else {
            runWithTopicsDiscovery();
        }
    }

```

### ParallelSourceFunction

没有具体的接口，只要为了告诉flink这是一个可以并行运行的Function

### RichFunction

主要管理`Function`的声明周期，flink会在构造函数后调用open方法，在销毁时调用close方法

```java
@Public
public interface RichFunction extends Function {
    void open(Configuration var1) throws Exception;

    void close() throws Exception;

    RuntimeContext getRuntimeContext();

    IterationRuntimeContext getIterationRuntimeContext();

    void setRuntimeContext(RuntimeContext var1);
}

```

`FlinkPulsarSource`在`open`接口中干了如下事情

```java
public void open(Configuration parameters) throws Exception {
    
    
    // 1. 创建PulsarMetadataReader，PulsarMetadataReader并调用discoverTopicChanges()拿到所有的topic.
    // 创建PulsarMetadataReader管理了pulsar所有元数据的访问，比如拿到一个topic下有多少个partition等。
    this.metadataReader = createMetadataReader();
    Set<String> allTopics = metadataReader.discoverTopicChanges();
    
    // 2. 创建ownedTopicStarts成员，并赋值。赋值时依赖`restoredState(<String, MessageId>)`成员变量
    // ownedTopicStarts管理了所有topic下消费的开始offset(MessageId提供)    
    //  Map<String, MessageId> ownedTopicStarts;  
    ownedTopicStarts = new HashMap<>(); 
    if(restoredState ！= null) {
      //  从restoredState中恢复offser
      // restored在`CheckpointedFunction#initializeStates()`接口中赋值，可以参考下面具体实现分析
    } else {
      //  调用offsetForEachTopic()方法根据用户的配置(StartupMode)设置为从头还是从尾开始消费
    }
}

```


## CheckpointedFunction

TODO

```java
	void snapshotState(FunctionSnapshotContext context) throws Exception;

	void initializeState(FunctionInitializationContext context) throws Exception;

```

## CheckpointListener

TODO

```java
	/**
	 * This method is called as a notification once a distributed checkpoint has been completed.
	 * 
	 * Note that any exception during this method will not cause the checkpoint to
	 * fail any more.
	 * 
	 * @param checkpointId The ID of the checkpoint that has been completed.
	 * @throws Exception
	 */
	void notifyCheckpointComplete(long checkpointId) throws Exception;
```
