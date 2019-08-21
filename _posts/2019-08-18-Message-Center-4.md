---
layout:     post
title:      分布式Websocket推送中心(四)-推送中心的分布式架构方案设计落地
subtitle:   使用Spring Websocket Stomp协议设计推送中心
date:       2019-08-18
author:     baozi
header-img: img/2019-08-15(Message-Center)/top-ux.jpg
catalog: true 						
tags:								
    - Java
    - 架构
---

### 概述
本文是分布式WebSocket推送中心的第三章节， 本系列文章是在Spring Websocket Stomp的基础上实现的推送系统，计划包含如下几篇文章：

**第一篇**：[Spring Websocket Stomp介绍](https://shibd.github.io/2019/08/15/Message-Center-1/)<br>
**第二篇**：[基于Websocket Stomp的推送中心实现](https://shibd.github.io/2019/08/16/Message-Center-2/)<br>
**第三篇**：[推送中心单机支持百万级连接的晋级之路](https://shibd.github.io/2019/08/17/Message-Center-3/)<br>
**第四篇**：推送中心的分布式架构方案设计落地<br>

### 本章主线
TODO