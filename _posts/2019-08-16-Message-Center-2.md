---
layout:     post
title:      分布式Websocket推送中心(二)-基于Stomp的推送中心设计
subtitle:   使用Spring Websocket Stomp协议设计推送中心
date:       2019-08-16
author:     baozi
header-img: img/2019-08-15(Message-Center)/top-ux.jpg
catalog: true 						
tags:								
    - Java
    - 架构
---

## 概述
本文是分布式WebSocket推送中心的第二章节， 本系列文章是在Spring Websocket Stomp的基础上实现的推送系统，计划包含如下几篇文章：

第一篇：Spring Websocket Stomp介绍<br>
第二篇：基于Websocket Stomp的推送中心实现<br>
第三篇：推送中心单机支持百万级连接的晋级之路<br>
第四篇：推送中心的分布式架构方案设计落地<br>


## 本章主线
上篇文章介绍了Spring WebSocket STOMP相关内容，奠定了推送中心使用Websocket协议来做，采用Spring Websocket STOMP框架快速实现了简单的服务端到客户端的推送。

本篇从推送中心设计开始，分享推送中心功能，如何安全建立链接，如何对客户端鉴权，如何实现广播，多播，以及点对点等内容。

## 推送中心功能设计
推送中心的目标是为了满足全公司的后端推送业务的，从功能上需要满足多个项目同时接入，所以推送中心需要支持多项目。同时项目与项目之间数据要完全隔离，不能`电商项目`的广播消息被`理财项目`给接收到了。为了说明推送中心的功能需要，请看下图。
![](/img/2019-08-15(Message-Center)/architecture2.jpg)
另外推送中心支持广播，组播，单播接口。例如电商项目可以支持给所有客户端发送消息(`广播`)，也可以指定给所有地点为**上海**的客户端发送消息(`多播`)，也可以指定给用户1发送消息(`单播`)。

## 如何实现广播、组播、单播
### 谈谈自己实现
在谈论如何使用Websocket实现广播、组播、单播之前，我们先来明确一下WebSocket的本质。WebSocket其实是客户端和服务端`多对1`建立的长连接，对于服务端(`推送中心`)来说，它和N个客户端连接，所以它自然可以给每个连接打`tag`，比如他可以标机一条连接是电商项目的用户1，另一条连接是理财项目的`shanghai`组。所以我们可以在应用内部设计这样一个映射表来实现:

> `Map<项目ID, Map<主题, List<链接>>>` <br>

其中主题是很灵活的，在不同的模式下，可以前后端约定好值即可，例如：
- 广播：主题约定设置为`all`，客户端订阅all，服务端找到所有订阅`all`的客户端连接，逐条推送即可完成广播。
- 多播：主题前后端约定好，比如客户端可以根据地区，订阅`shanghai`的主题，那么服务端就可以找到所有订阅`shanghai`的连接，逐条推送完成多播。
- 单播：主题约定为前端传的`userId`，或者`设备唯一id`，服务端还是根据主题找到连接，推送即可。

### 谈谈借助STOMP实现
看到这里其实已经明白，所谓的广播、多播、单播，在WebSocket下就是主题和链接的关系。客户端订阅唯一的主题就是单播，客户端都订阅相同的主题就是广播，某一些客户端订阅的主题就是多播。这和第一章我们聊的STOMP协议是相似的，我把上篇的相关代码拿过来。
``` java
// javascript客户端订阅主题
stompClient.subscribe('/topic/greetings', function (greeting) {
    // 服务端发送消息，客户端收到展示
    showGreeting(JSON.parse(greeting.body).content);
});
// 订阅单用户消息
stompClient.subscribe('/user/queue/' + project + '/', function (greeting) {
    showGreeting('UserMessage: ' + JSON.parse(greeting.body).content);
});

// java服务端望该主题发送消息
messagingTemplate.convertAndSend("/topic/greetings",
	new Greeting("Hello, " + HtmlUtils.htmlEscape(message.getName()) + "!"))
}
```
实际上Spring WebSocket STOMP的实现也是维护了类似于上面讲述的映射关系，感兴趣的同学可以翻阅源码`SessionSubscriptionRegistry`。我们可以通过STOMP加上约定来实现多项目下的广播、多播和单播，我们约定STOMP的订阅路径为:
>`/topic/group` <br>
`topic`: 为统一后端约定广播模式根路径。<br>
`group`: `group`可以自定义，可以前后端约定为`all`来实现广播，也可以定义子路径为`/greetings/group1`来实现多播。也可以定义为`唯一设备id`来实现单播。

另外Spring WebSocket STOMP还支持针对认证的用户单独发送消息，你可以认为这也是多播的一种方案(因为同一个用户有可能多个客户端建立连接)，约定STOMP订阅路径为：
>`/user/queue/`

综上，推送中心根据STOMP，约定了前端订阅规范，另外又提供了REST接口供后端调用推送数据。入参为：
``` java
public class WsTopicMessage {
	/**
	 * 项目ID
	 */
	private String projectId;

	/**
	 *
	 * 主题
	 */
	private String topic;

	/**
	 * 具体内容(和前端约定好的json数据)
     * 例如:
     * "{\"content\":\"解析我,做你想做的事情\"}"
	 */
	private String playLoad;
}
```
`项目前端`、`推送中心`、`项目后端`间完整调用流程为：
- `项目前端`向推送中心建立WebSocket连接，并调用subscribe订阅，传入和后端约定好的主题`shanghai`，完整订阅路径:`/topic/projectId/shanghai`。
- `推送中心`根据前端链接鉴权(后面会说)，同意建立连接，然后根据订阅的路径维护`订阅路径`与`连接`间的关系。
- `项目后端`调用推送中心的REST接口，传入**projectId**和**topic**以及**playLoad**。
- `推送中心`根据projectId和topic找到一堆或一个客户端链接发送消息。


## 鉴权方案
xxx

### 方案1
xxx

### 方案2
xxx

## Websocket安全
xxx