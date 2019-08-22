---
layout:     post
title:      分布式Websocket推送中心(四)-推送中心的分布式架构方案设计落地
subtitle:   使用Spring Websocket Stomp协议设计推送中心
date:       2019-08-18
author:     baozi
header-img: img/2019-08-15(Message-Center)/top4.jpg
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
前三章介绍了推送中心的设计，单机支持百万连接。到目前为止推送中心还是单体的，如果公司业务规模超过百万，该如何横向扩容，推送中心如何避免单点故障，如何实现高可用。

本章会提出多种分布式架构，每种架构的适用场景不同。

### 目前架构
到目前为止，推送中心就只有一个推送服务，`系统后端`调用推送服务的Restful接口，推送服务根据`主题`找到相对应的WebSocket连接，推送消息至客户端。

![](/img/2019-08-15(Message-Center)/architecture4-1.jpg)

要做分布式架构，我们先尝试很暴力的加一个服务进来，看看会是什么情况。

![](/img/2019-08-15(Message-Center)/architecture4-2.jpg)

增加一个推送服务，`系统后端`和`客户端`要使用就要配置新的推送服务的地址。这种架构下，`系统后端`要发送广播消息就要分别调用俩个推送服务。而且如果推送服务更换地址，增加或减少实例个数，对`系统后端`和`客户端`都不是透明的。

### 分布式架构（使用Kafka + WebSocket代理）
目前阶段采用的架构方式
#### 对系统后端透明
上述增加一个推送服务后，`系统后端`要调用俩个推送服务才能达到给各个客户端广播的效果，本质上是因为推送服务并不是无状态的，各个推送服务和不同的客户端建立的长连接，这个`连接`是做不了无状态的。

为了达到推送中心对于`系统后端`透明，我们需要在推送服务之上加一层服务代理。这个服务代理要有自动发现推送服务，而且可以把一个推送请求分别调用不同的推送服务完成推送。

消息中间件再合适不过了，如下。面向`系统后端`在所有推送服务至上加一层MQ，公司内部使用Kafka。每个推送服务使用不同的group订阅相同的MQ的topic，根据topic中的内容，每个推送服务找到自己对应的`连接`推送即可。

我们认为kafka不会成为系统瓶颈，消息量很大的情况下，我们可以不同的项目使用不同的topic来均摊broker的压力。

![](/img/2019-08-15(Message-Center)/architecture4-3.jpg)

所以现在，对于`系统后端`来说，我们可以说推送中心是可以横向扩容的了。

#### 对客户端透明
我们在推送中心之前加了一层kafka作为消息代理，做到了推送中心对`系统后端`透明。但对于客户端还是不透明的，客户端发起连接时还要考虑到底连接哪个推送服务，如果推送服务更换IP，更换实例都会影响到客户端。

最容易想到的解决办法是在推送服务面向客户端加一层网络代理，比如使用Nginx代理websocket连接。nginx可以轮训推送服务，代理发起WebSocket连接，这样推送服务对客户端也是透明的了。
![](/img/2019-08-15(Message-Center)/architecture4-4.jpg)

Nginx代理WebSocket连接，实则是先自己作为服务端和客户端建立长连接，然后再作为客户端和推送中心建立长连接。我们在上篇文章说到，一个客户端最多可以发起65535个长连接，所以使用nginx代理WebSocket连接就会遇到65535瓶颈。本来一个推送服务可以支持百万的长连接，但是一个nginx就把连接数卡到了6W，想想上篇的文章调优，难道要无功而返？

目前这种架构方式虽然长连接瓶颈卡在了6W，但目前公司使用是可以接受的，业务量级还没有达到如此。另外相关文章提到可以使用LVS的DR模式来做负载均衡可以突破端口的限制，笔者没有使用，感兴趣的同学可以[参考](https://blog.csdn.net/weixin_40470303/article/details/80541639)。


#### 如何高可用
上面分别使用Kafka和Nginx做到对服务端和客户端透明，虽然因为nginx的瓶颈，做不了横向扩容，但是推送中心应该是要保证高可用的。比如一台推送服务挂了，客户端和服务端应该可以正常使用才行。

因为推送服务是维护了`<主题, 连接>`的状态的，这个`连接`是客户端和服务端握手好的资源，这个是不能无状态的。为了实现推送中心的高可用，要保障客户端能自动重连。

>一个推送服务死掉，断开所有客户端的连接，客户端检测到重新发起连接请求，nginx代理到另一台好的推送服务，连接完毕。

javascript重连代码(来自**大都督**)
``` javascript
function errorCallback(error) {

    setConnected(false);

    if (_breakReason === INVALID_TOKEN) {
        // error callback will be called twice, that's why we record _breakReason with first call
        // we should just leave the callback if _breakReason from last call exist and match invalid_token
        return
    }
    // quit here since token is checked as invalid by server
    // no need to re-connect with same arguments
    if (
        error &&
        error.headers &&
        error.headers.message &&
        error.headers.message.includes('Failed to send message')
    ) {
        _setBreakReason(INVALID_TOKEN)
        return
    }
    reconnect()
}
function reconnect() {
    console.log("30s retrying connect")
    setTimeout(() => {
        connect();
    }, 30000)
}
```

### 分布式架构（Kafka + 注册中心）

上述架构由于使用nginx代理WebSocket会遇到65535端口限制，虽然目前公司体量不大，采用了这种方式。但后续免不了改造，下面简单说一下后续的改造思路。

使用nginx代理WebSocket是为了做到`推送中心`对`客户端`透明时提出的解决方案。首先透明背后的含义是：`如果推送中心实例重启，更换个数，更换地址，应该做到客户端不改变代码，不重新编译，能自动识别，自动切换`。

我们也可以不使用代理，客户端直接连接推送服务，只不过在连接推送服务之前先动态获取可用的服务的地址，可以引入`注册中心`来解决该问题，该`注册中心`是面向客户端的。`注册中心`负责维护推送中心中各个推送服务的状态。引入注册中心后客户端调用流程如下:
![](/img/2019-08-15(Message-Center)/liucheng1.jpg)

架构图
![](/img/2019-08-15(Message-Center)/architecture4-5.jpg)

该架构模式下，推送中心也做到了对客户端透明，而且可以动态扩充实例个数。该注册中心还可以负责统一收集各个推送服务的连接情况，统一管理，解决使用nginx代理架构下连接分散管理的问题。

### 总结
推送中心到此分享完毕，其实推送中心的代码很少，借助了框架来实现，四篇文章分享了笔者在设计推送中心以及实践过程中的一些总结，很多地方写的很粗糙，欢迎指正，谢谢。