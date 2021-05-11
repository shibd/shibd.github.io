---
layout:     post
title:      Flink源码解析[Source] - 00为什么要解析Source源码？
subtitle:   Flink源码解析[Source] - 00为什么要解析Source源码？
date:       2021-05-11
author:     baozi
header-img: img/2019-08-14(RocketMQ-Phoenix)/top-ux.jpg
catalog: true 						
tags:								
    - flink


---



## 





1. 为了可以明白flink是如何Source抽象的：
   1. source可有有文件，socket，mq，数据库等组成？
   2. flink是如何抽象的?

2. checkout依赖于Source可以回溯消费，代码上是怎么实现的？



## 目标



1. 理解flink-source的机制
2. 理解kafka-flink的实现
3. 理解pusar-flink的实现



## 目录



1. source是如何创建的？和flink的关系是什么？
2. 