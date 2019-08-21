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

### 概述
本文是分布式WebSocket推送中心的第二章节， 本系列文章是在Spring Websocket Stomp的基础上实现的推送系统，计划包含如下几篇文章：

第一篇：Spring Websocket Stomp介绍<br>
第二篇：基于Websocket Stomp的推送中心实现<br>
第三篇：推送中心单机支持百万级连接的晋级之路<br>
第四篇：推送中心的分布式架构方案设计落地<br>


### 本章主线
上篇文章介绍了Spring WebSocket STOMP相关内容，奠定了推送中心使用Websocket协议来做，采用Spring Websocket STOMP框架快速实现了简单的服务端到客户端的推送。

本篇从推送中心设计开始，分享推送中心如何支持多项目，如何安全建立链接，如何对客户端鉴权，如何实现广播，多播，以及点对点等内容。

### 功能需求
推送中心的目标是为了满足全公司的后端推送业务的，从功能上需要满足多个项目同时接入，同时项目与项目之间数据要完全隔离，不能`电商项目`的广播消息被`理财项目`给接收到了。为了说明推送中心的功能需要，请看下图。
![](/img/2019-08-15(Message-Center)/architecture2.jpg)
1. 推送中心要支持支持广播，组播，单播接口。例如电商项目可以支持给所有客户端发送消息(`广播`)，也可以指定给所有地点为上海的客户端发送消息(`多播`)，也可以指定给用户1发送消息(`单播`)。
2. 推送中心要有鉴权功能，客户端连接推送中心时推送中心要能校验客户端是否合法。
3. 推送中心要灵活管理连接，实时查看客户端连接情况，主动断开与某个客户端的连接等功能。
4. 推送中心要有黑白名单功能，能防御恶意连接，能防御DDOS攻击。
5. 推送中心要高可用，支持高并发连接。

本文分享上述1，2，3，4解决方案。如何高可用如何支持高并发连接在第三篇和第四篇分享。

### 实现广组单播
#### 谈谈自己实现
在谈论如何使用Websocket实现广播、组播、单播之前，我们先来明确一下WebSocket的本质。WebSocket其实是客户端和服务端`多对1`建立的长连接，对于服务端(`推送中心`)来说，它和N个客户端连接，所以它自然可以给每个连接打`tag`，比如他可以标机一条连接是电商项目的用户1，另一条连接是理财项目的`shanghai`组。所以我们可以在应用内部设计这样一个映射表来实现:

> `Map<项目ID, Map<主题, List<链接>>>` <br>

其中主题是很灵活的，在不同的模式下，可以前后端约定好值即可，例如：
- 广播：主题约定设置为`all`，客户端订阅all，服务端找到所有订阅`all`的客户端连接，逐条推送即可完成广播。
- 多播：主题前后端约定好，比如客户端可以根据地区，订阅`shanghai`的主题，那么服务端就可以找到所有订阅`shanghai`的连接，逐条推送完成多播。
- 单播：主题约定为前端传的`userId`，或者`设备唯一id`，服务端还是根据主题找到连接，推送即可。

#### 谈谈借助STOMP实现
看到这里其实已经明白，所谓的广播、多播、单播，在WebSocket下就是`主题`和`链接`的关系。客户端订阅唯一的主题就是单播，客户端都订阅相同的主题就是广播，某一些客户端订阅相同的主题就是多播。这和第一章我们聊的STOMP协议是相似的，我把上篇的相关代码拿过来。

实际上Spring WebSocket STOMP的实现就是维护了类似于`destination`和`连接`的关系，这里的`destination`就是客户端订阅的目录格式的路径，感兴趣的同学可以翻阅官方源码类`SessionSubscriptionRegistry`。
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

// java服务端向该主题发送消息
messagingTemplate.convertAndSend("/topic/greetings",
 new Greeting("Hello, " + HtmlUtils.htmlEscape(message.getName()) + "!"))
}
```

综上，我们可以使用STOMP协议加上和项目约定好订阅路径来实现多项目下的广播、多播和单播。我们约定STOMP的订阅路径为:
>`/topic/projcetId/xxx` <br><br>
`topic`: 统一订阅的根路径名称。<br>
`projcetId`: 项目Id, 必传 <br>
`xxx`: `xxx`可以自定义，可以前后端约定为`all`来实现广播，也可以定义子路径为`/greetings/group1`来实现多播。也可以定义为`唯一设备id`来实现单播。

Spring WebSocket STOMP还支持针对认证的用户单独发送消息，你可以认为这也是多播的一种方案(因为同一个用户有可能多个客户端建立连接)，约定STOMP订阅路径为：
>`/user/queue/`

推送中心和项目客户端以上述约定格式建立ws连接后，项目后端可以通过调用推送中心暴露的restful接口推送消息。接口入参如下：
``` java
{
  "projectId": "电商项目id",
  "playLoad": "{\"content\":\"解析我,做你想做的事情\"}",
  "topic": "all/group1/",
  "updateTime": 1557676800000
}
```
`项目客户端`、`推送中心`、`项目后端`间完整调用流程为：
1. `项目客户端`向推送中心建立WebSocket连接，并调用subscribe订阅，传入和后端约定好的主题`shanghai`，完整订阅路径:`/topic/projectId/shanghai`。
2. `推送中心`根据前端链接鉴权(后面会说)，同意建立连接，然后根据订阅的路径维护`订阅路径`与`连接`间的关系。
3. `项目后端`调用推送中心的REST接口，传入**projectId**和**topic**以及**playLoad**。
4. `推送中心`根据projectId和topic找到一堆或一个客户端链接发送消息。


### 实现鉴权
#### 方案设计
客户端在和推送中间建立ws连接时需要鉴权，如果一个不合法的客户端成功和Websocket建立链接，那么就可以收到他想窃听的消息。客户端和推送中心建立ws连接时，推送中心需要去找业务系统确认客户端的合法性，具体实现方案有很多种。

期初想的方案很直白，比如下述鉴权流程：
1. 建立ws之前，先客户端拿着业务系统给的token，再发送到业务系统； <br>
2. 业务系统后端再到推送服务去拿key（因为对于推送服务，业务系统的后端是可信赖的）；<br>
3. 然后业务系统后端把key返回给客户端；<br>
4. 客户端用key再构造ws的url，尝试和推送服务建立长连接，推送服务从url里提取key，校验合法性，accept连接

这种方案虽然没问题，但是交互复杂，而且没法识别用户。我理想的交互方式是在对客户端鉴权时不要让推送中心和项目后盾最交互。其实推送中心对客户端鉴权，我认为是JWT的一种典型场景，JWT就像是业务系统颁布的合法的签名，推送中心作为第三方部门系统，是认可这个签名的。具体步骤如下，关于JWT相关推荐[文章]()

1. 业务系统使用推送中心初始化公钥，推送中心维护项目和公钥的集合。
2. 客户端正常登陆业务系统，业务系统使用私钥生成jwt颁布给客户端。 
3. 客户端鉴权时传入projectId和jwt，推送中心根据projectId获取公钥，然后通过公钥校验，通过连接建立成功，不通过，连接建立失败。

![](/img/2019-08-15(Message-Center)/auth.jpg)


#### STOMP协议建立时鉴权
>The WebSocket protocol, RFC 6455 “doesn’t prescribe any particular way that servers can authenticate clients during the WebSocket handshake.” In practice, however, browser clients can use only standard authentication headers (that is, basic HTTP authentication) or cookies and cannot (for example) provide custom headers. Likewise, the SockJS JavaScript client does not provide a way to send HTTP headers with SockJS transport requests. See sockjs-client issue 196. Instead, it does allow sending query parameters that you can use to send a token, but that has its own drawbacks (for example, the token may be inadvertently logged with the URL in server logs).

Spring Websocket STOMP[官网](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#websocket-stomp-authentication-token-based)在Token Authenication中说到，由于WebSocket协议并没有规定在WebSocket握手期间对客户端进行身份认证，而且SockJS JavaScript客户端不提供建立连接时自定义请求头，但是允许传入请求参数，所以我们可以把token放到请求参数当中。

Spring Websocket STOMP官网没有选择把token在ws握手时传入，推荐在创建STOMP协议时带入token到服务端认证，通过创建`ChannelInterceptor`实现，可以参见[推送中心完整项目地址](https://github.com/shibd/msg-center)

客户端在STOMP建立时传入JWT
``` javascript
stompClient = Stomp.over(socket);
stompClient.connect({
    token: token,
    projectId: projectId
}, connectCallback, errorCallback);

//连接失败时的回调函数
function errorCallback(res) {
  if(res属于鉴权失败) {
    // 取消连接，调用回调函数
  }
  if(res属于服务端不存在或者网络错误) {
    // 隔段时间重试
  }
}
}
```

服务端配置STOMP认证管道校验JWT
``` java
@Configuration
@EnableWebSocketMessageBroker
public class MyConfig implements WebSocketMessageBrokerConfigurer {

	/**
	 * 配置STOMP认证管道
	 * @param registration
	 */
	@Override
	public void configureClientInboundChannel(ChannelRegistration registration) {
		registration.interceptors(new ChannelInterceptor() {
			@Override
			public Message<?> preSend(Message<?> message, MessageChannel channel) {
				StompHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
				if (StompCommand.CONNECT.equals(accessor.getCommand())) {
					Map<String, LinkedList> headers = (Map) message.getHeaders()
							.get(SimpMessageHeaderAccessor.NATIVE_HEADERS);
					// 鉴权,校验失败会抛出异常,通过ws把消息给到客户端
					Principal user = authenticate(headers);
					accessor.setUser(user);
				}
				return message;
			}
		});
	}
}
```

#### Websocket建立时鉴权
上述官方推荐在建立STOMP协议时鉴权，虽然socket js不支持修改ws的headers，但是也可以放在请求入参中完成在WebSocket握手时鉴权。相关代码如下

客户端在WS建立时传入JWT
``` javascript
var socket = new SockJS('/msg-center/websocket?token=' + token + "&projectId=" + projectId)
stompClient = Stomp.over(socket);
stompClient.connect({}, connectCallback, errorCallback);
```
服务端配置拦截器校验JWT
```java
/**
 * 注册STOMP协议节点并映射url
 * @param registry
 */
@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
 registry
     // 注册一个 /websocket 的 websocket 节点
     .addEndpoint("/websocket").addInterceptors()
     // 添加 websocket握手拦截器
     .addInterceptors(myHandshakeInterceptor())
     // 添加 websocket握手处理器
     .setHandshakeHandler(myDefaultHandshakeHandler())
     // 设置允许可跨域的域名(一定程度预防CSRF攻击)
     .setAllowedOrigins("*")
     // 指定使用SockJS协议
      .withSockJS();
}

/**
 * WebSocket 握手拦截器 可做一些用户认证拦截处理
 */
private HandshakeInterceptor myHandshakeInterceptor() {
  return new HandshakeInterceptor() {
    /**
     * websocket握手连接
     * @return 返回是否同意握手
     */
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
        WebSocketHandler wsHandler, Map<String, Object> attributes) {
      ServletServerHttpRequest req = (ServletServerHttpRequest) request;

      // 根据token认证用户，不通过返回拒绝握手
      String token = req.getServletRequest().getParameter("token");
      String projectId = req.getServletRequest().getParameter("projectId");
      Principal user = authenticate(projectId, token);
      if (user == null) {
        return false;
      }

      // 保存会话信息
      attributes.put("user", user);
      attributes.put("remoteUrl", request.getRemoteAddress());
      attributes.put("projectId", projectId);
      return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
        WebSocketHandler wsHandler, Exception exception) {
    }
  };
}

/**
 * WebSocket 握手处理器
 */
private DefaultHandshakeHandler myDefaultHandshakeHandler() {
  return new DefaultHandshakeHandler() {
    @Override
    protected Principal determineUser(ServerHttpRequest request, WebSocketHandler wsHandler,
        Map<String, Object> attributes) {
      // 设置认证通过的用户到当前会话中
      return (Principal) attributes.get("user");
    }
  };
}
```


### 实现管理连接
在Spring Web STOMP中可以使用`SimpUserRegistry`对象获取所有`session`的集合。推送中心在支持多项目的情况下，对`SimpUserRegistry`结果做了处理，支持分项目分订阅主题来查看连接，但是暂时不能对连接做修改。样例数据：
```
  "data": [
    {
      "projectId": "fm",
      "wsUserSessionVos": [
        {
          "userId": "wudixiaobaozi",
          "sessions": [
            {
              "sessionId": "jdtcl3r4",
              "remoteUrl": "/127.0.0.1:52476",
              "subscriptions": [
                "/user/queue/fm/",
                "/topic/fm/all/group1/dss"
              ]
            },
            {
              "sessionId": "fyhgruy5",
              "remoteUrl": "/127.0.0.1:52465",
              "subscriptions": [
                "/topic/fm/all/group1/",
                "/user/queue/fm/"
              ]
            }
          ]
        }
      ]
    }
  ]
```

### 实现安全
其实WebSocket涉及很多安全性的问题，笔者目前还没实验踩坑，后续会单独对安全性进行测试，感兴趣的可以先参考[该篇文章](https://www.anquanke.com/post/id/85999)。

### 总结
本篇介绍了如何使用Spring WebSocket STOMP支持多项目下广播，多播，单播推送设计，通过前后端约定主题，后端调用推送中心restful接口实现。另外介绍了推送中心的鉴权方案设计，以及两种实现方式。可参考
>[推送中心完整代码](https://github.com/shibd/msg-center)。

到现在为止，单体的推送中心设计已经结束，后续会分享单体推送中心服务器调优达到百万级长连接，以及推送中心的集群方案设计。