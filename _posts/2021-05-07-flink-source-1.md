---
layout:     post
title:      Flink源码解析[Source](一) - 为什么要解析Source源码
subtitle:   Flink源码解析[Source](一) - 为什么要解析Source源码
date:       2021-05-07
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
**第三篇**：[新版Data Srouces详解&源码](https://shibd.github.io/2021/05/14/flink-source-3/)<br>



## 问题

1. 为了可以明白flink是如何Source抽象的

2. checkout依赖于Source可以回溯消费，代码上是怎么实现的？
3. source中watermark是如何实现的，代码结构是如何？

## 目标

1. 理解flink-source的机制
2. 理解kafka-flink的实现
3. 理解pusar-flink的实现

可以熟透这块源代码，熟悉kafka以及pulsar的实现，并可以贡献源代码。

## connect实现

1. [kafka source connect]()
2. [pusar source connect]()

