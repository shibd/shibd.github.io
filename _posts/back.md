---

## 概述

本系列文章是旨在熟悉摸头flink的source-connect原理，希望可以做到自己可以实现一个新的source，代码解析将会以kafka的实现配合flink的api为主线解析。

>  flink版本为1.12.0

**第一篇**：[为什么要解析Source源码](https://shibd.github.io/2021/05/07/flink-source-1/)<br>
**第二篇**：[如何创建Flink kafka source](https://shibd.github.io/2021/05/11/flink-source-2/)<br>
**第三篇**：[旧版Data Source详解&源码](https://shibd.github.io/2021/05/14/flink-source-3/)<br>



该篇文章主要以官方文档为主线解析，并配合自己的理解，以及配合源码解析，[官方文档地址](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/stream/sources.html)。




## Data Source 概览









## KafkaSourceEnumerator源码解析



### KafkaPartitionSplit



Split的最小单位， topic  + patiton + 起始结束位置 

（有界，无界）



### SplitEnumeratorContext



负责assignSplits 主要jobManager去通知taskManager，另外还提供了线程模型去便于处理等

