---
layout:     post
title:      分布式Websocket推送中心(一)-Spring Websocket Stomp介绍
subtitle:   使用SpringWebsocket实现百万级连接的分布式推送中心
date:       2019-08-15
author:     baozi
header-img: img/2019-08-15(Message-Center)/top-ux.jpg
catalog: true 						
tags:								
    - Java
    - 架构
---

### 概述
公司开发项目众多，各个项目都有和终端推送的需求，于是决定实现一个公司级的推送中心，屏蔽技术细节，服务全公司。笔者在设计实现完后记录下心路历程。

本系列文章是在Spring Websocket Stomp的基础上实现的推送系统，计划包含如下几篇文章：

**第一篇**：Spring Websocket Stomp介绍<br>
**第二篇**：[基于Websocket Stomp的推送中心实现](https://shibd.github.io/2019/08/16/Message-Center-2/)<br>
**第三篇**：[推送中心单机支持百万级连接的晋级之路](https://shibd.github.io/2019/08/17/Message-Center-3/)<br>
**第四篇**：[推送中心的分布式架构方案设计落地](https://shibd.github.io/2019/08/18/Message-Center-4/)<br>

### WebSocket协议
Websocket是为了解决服务端和客户端双向通信问题，采用长链接，避免了HTTP协议无状态反复解析请求头的问题。WebSocket协议细节不多解释，读者可以google学习。简单总结一下，WebSocket协议特点：
- TCP/IP中的应用层协议
- 建立在TCP协议之上，全双工通信协议
- 握手阶段采用Http协议，需要从Http协议升级而来
- 没有同源限制，客户端可以与任意服务端建立连接

### STOMP协议(Simple Text Oriented Messaging Protocol)
STOMP是一个用于C/S之间进行异步消息传输的简单文本协议, 全称是Simple Text Oriented Messaging Protocol。

>[STOMP官方网站](http://stomp.github.io/index.html)

其实STOMP协议并不是为WS所设计的，它其实是消息队列的一种协议，和AMQP，JMS是平级的。 只不过由于它的简单性恰巧可以用于定义WS的消息体格式。
目前很多服务端消息队列都已经支持了STOMP，比如RabbitMQ，Apache ActiveMQ等。很多语言也都有STOMP协议的客户端解析库，像JAVA的Gozirra，C的libstomp，Python的pyactivemq，JavaScript的stomp.js等等。[原文](https://juejin.im/post/5b7071ade51d45665816f8c0)

RabbitMQ提供了WebSocket的插件，你可以通过使用STOMP + Websocket + RabbitMQ实现服务端推送。[参考该文](https://www.ibm.com/developerworks/cn/opensource/os-cn-rabbit-mq/index.html)

STOMP只是文本传输协议，并不参与通信细节。可以简单理解它是一个生产者/消费者模型的规范，服务端和客户端都可以推送和消费。而且他支持灵活的消息分发，比如应用可以自定义以为`/topic`打头的为发布/订阅模式，以`/user`打头的为点对点模式。

Webcoekt结合STOMP就相当于实现了一个消息队列，服务端与客户端都可以作为生产者和消费者发送和消费消息，可以根具订阅不同的队列实现单薄、广播和多播(其实WebSocket是点对点的，广播和多播都只是服务端轮训所有客户端发送实现)。

### SpringBoot实现Websocket和STOMP
Spring遵循STOMP协议内部做了实现，Spring内部对服务做了大量的抽象，可以参照[官网](https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/websocket.html)和[该文](https://juejin.im/post/5b7071ade51d45665816f8c0#heading-20)结合理解其实现。

简单理解，你可以利用Spring配置服务端作为消费者订阅哪些topic消息，以及收到消息后处理方法。同样可以作为生产者为指定的topic里发送消息，例如`simple.send("/topic/group"， "message")`

其实应用在和对手建立链接后Spring会维护类似`Map<"/topic/group", List<地址>>`这样的映射关系，如果是点对点消息则`Map<"/topic/userName/queue", 地址>`。对于用户来说就不用维护链接信息，消息该发送给谁等问题，发送消息和接收消息都是操作**topic**。

### Spring运行Websocket实现简单服务端推送消息
#### 场景描述
我们来实现推送中心的第一步，实现服务端向客户端推送消息，客户端展示。基于[官方demo](https://spring.io/guides/gs/messaging-stomp-websocket/)改造实现。我放到了推送中心的demo分支上。

**[参考代码](https://github.com/shibd/msg-center/tree/simple/demo)**

#### 为Spring配置STOMP消息
通过`@EnableWebSocketMessageBroker`启动Spring Websocket STOMP，注册端点，配置消息代理前缀
``` java
package hello;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

/**
 * @author baozi Websocket配置类
 */
@Configuration
@EnableWebSocketMessageBroker // 使用此注解启动websocket，使用broker来处理消息
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

	@Override
	// 实现WebSocketMessageBrokerConfigurer中的此方法，配置消息代理（broker）
	public void configureMessageBroker(MessageBrokerRegistry config) {
		// 启用SimpleBroker，使得订阅到此"topic"前缀的客户端可以收到greeting消息.
		config.enableSimpleBroker("/topic");
	}

	@Override
	// 用来注册Endpoint，“/gs-guide-websocket”即为客户端尝试建立连接的后缀地址。
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint("/gs-guide-websocket").withSockJS();
	}

}
```

#### 编写发送Controller
编写一个Controller接收http请求，解析并发送消息至`/topic/greetings`下的topic。
``` java
package hello;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.util.HtmlUtils;

/**
 * @author baozi
 */
@Controller
public class GreetingController {

	@Autowired
	private SimpMessagingTemplate messagingTemplate;

	@RequestMapping(value = "/hello", method = RequestMethod.POST)
	public void greeting(@RequestBody HelloMessage message) throws Exception {
		// simulated delay
		Thread.sleep(1000);

		// 发送消息到所有订阅topic/greetings的客户端
		messagingTemplate.convertAndSend("/topic/greetings",
				new Greeting("Hello, " + HtmlUtils.htmlEscape(message.getName()) + "!"));

	}

}
```
#### 前端实现页面
前端SockJS实现了WebSocket，stomp.js实现了STOMP协议。
``` javascript
function connect() {
    // 建立与服务器的gs-guide-websocket断电的websocket链接
    var socket = new SockJS('/gs-guide-websocket');
    stompClient = Stomp.over(socket);
    stompClient.connect({}, 
      // 连接成功后的回调方法
      function (frame) {
        setConnected(true);
        console.log('Connected: ' + frame);
        // 订阅服务端开启的/topic下的greetings地址
        stompClient.subscribe('/topic/greetings', function (greeting) {
            // 服务端发送消息，客户端收到展示
            showGreeting(JSON.parse(greeting.body).content);
        });
    });
}
```

#### 演示

- 打开`http://127.0.0.1:8080`点击Connect进行websocket连接。
- 文本框输入消息点击发送，前端收到后端推送的刚才输入的消息。

![](/img/2019-08-15(Message-Center)/msg-ui.jpg)


### 总结
目标是实现一个可以支持百万链接的分布式推送中心，推送中心如何高可用，如何鉴权，websocket安全，如何使用该文都未涉及，后续篇章会陆续介绍。截止目前，我们使用Spring Websocket实现了前后端建立WebSocket链接，实现了后端给前端推送消息。简单架构，当然该"推送中心"还不是推送中心。

![](/img/2019-08-15(Message-Center)/architecture1.jpg)