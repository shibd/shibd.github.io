---
layout:     post
title:      分布式Websocket推送中心(二)-基于Stomp的推送中心设计
subtitle:   使用Spring Websocket Stomp协议设计推送中心
date:       2019-08-15
author:     baozi
header-img: img/2019-08-15(Message-Center)/top-ux.jpg
catalog: true 						
tags:								
    - Java
    - 架构
---



支持多项目

支持广播

支持单薄

消息风暴怎么解决的？

Q9:面前系统中的消息消费者可不可以分组？类似于Kafka。

客户端可以订阅不同产品的消息，接受不同的分组。接入的时候进行bind或者unbind操作
## 设计目标
