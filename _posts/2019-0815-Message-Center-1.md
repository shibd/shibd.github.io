---
layout:     post
title:      分布式Websocket推送中心(一)-Spring Websocket Stomp介绍
subtitle:   使用Spring Websocket Stomp协议设计推送中心
date:       2019-08-15
author:     baozi
header-img: img/2019-08-15(Message-Center)/top-ux.jpg
catalog: true 						
tags:								
    - Java
    - 架构
---

## 概述
公司内部项目众多，各项目都有和终端推送的需求，于是决定实现一个公司级的推送中心。笔者在设计实现完后记录下心路历程。本系列文章是在Spring Websocket Stomp的基础上实现的推送系统。

本系列文章计划包含如下几篇文章：

第一篇：Spring Websocket Stomp介绍<br>
第三篇：基于Websocket Stomp的推送中心的设计方案<br>
第四篇：单机支持百万级连接的晋级之路<br>
第五篇：推送中心的分布式架构方案设计落地<br>

## WebSocket协议
WebSocket协议不多解释，读者可以google学习。简单总结一下，WebSocket协议特点：
- 建立在TCP协议之上，全双工通信协议
- 握手阶段采用Http协议，与Http协议有较好的兼容性
- 没有同源限制，客户端可以与任意服务端建立连接

缺图片

## Stomp协议
xxx

## Spring WebSocket Stomp
xxx