---
layout:     post
title:      分布式Websocket推送中心(三)-单机服务100W连接(C1000K)目标达成
subtitle:   使用Spring Websocket Stomp协议设计推送中心
date:       2019-08-17
author:     baozi
header-img: img/2019-08-15(Message-Center)/top3.jpg
catalog: true 						
tags:								
    - Java
    - 架构
---

### 概述
本文是分布式WebSocket推送中心的第三章节， 本系列文章是在Spring Websocket Stomp的基础上实现的推送系统，计划包含如下几篇文章：

**第一篇**：[Spring Websocket Stomp介绍](https://shibd.github.io/2019/08/15/Message-Center-1/)<br>
**第二篇**：[基于Websocket Stomp的推送中心实现](https://shibd.github.io/2019/08/16/Message-Center-2/)<br>
**第三篇**：推送中心单机支持百万级连接的晋级之路<br>
**第四篇**：[推送中心的分布式架构方案设计落地](https://shibd.github.io/2019/08/18/Message-Center-4/)<br>

### 本章主线
上篇文章介绍了如何使用WebSocket和STOMP来实现推送中心的需求，确定了单体下推送中心功能。本章将分享推送中心在服务器单体如何支持百万级连接，从测试案例编写开始，一步一步进行服务器调优，踩坑，突破连接数，最后达到百万连接的过程。

### 必备知识
我们在谈单机支持百万连接前，先来明确一些结论。

对于服务器，Linux内核是通过（local_ip，local_port，remote_ip，remote_port）四元组来标识一条TCP连接的。

对于WebSocket服务端，local_ip和local_port是确定的，所以在内存和cpu足够的情况下，服务端不会出现瓶颈。在现代CPU和内存的配置下，服务器支持百万级连接实际是很轻松的。

对于WebSocket客户端，如果我们使用Linux服务器作为客户端来压测Websocekt服务的话，在一个客户端服务器下，由于remote_ip是固定的，所以每一条连接需要占用客户端服务器的一个remote_port。对于Linux服务器来说，net.ipv4.ip_local_port_range最大端口值为65535，所以一个客户端压测试最大可以压满65535条连接。

### 压测案例
计划是通过一个客户端进程发起百万连接（暂时不考虑单机65535端口限制，后面有解决方案）。这里客户端采用Java编写，官方提供了Websocket STOMP Client使用教程，这里对代码稍作调整，使用线程池并发发起连接，并且做了连接失败重试等处理。完整代码[参考github](https://github.com/shibd/msg-center/tree/master/benchmark)
``` java
/**
 * @author: baozi
 * @date: 2019/8/9 14:57
 * @description:
 */
public class StompClient {

  public static String TOKEN = "eyJ1c2VyTmFtZSI6Ind1ZGl4aWFvYmFvemkiLCJhbGciOiJSUzI1NiJ9.eyJleHAiOjE1Njc1OTE1NDYsInVzZXJOYW1lIjoid3VkaXhpYW9iYW96aSIsImlhdCI6MTU2NDk5OTU0Nn0.dEJzjgwwZCL6qh3NtluVSo0uZZZdUEzrNF2pLsUxprVOSE-pzaUVlOw2EmntXd4IpFs3qI0IwA4F51VOFIX65lc1RoX93AFeb44CYt9JpXKcGtGYWQr2D4nsNMaS7je8abtastBC8QIInCYtC7s8tvaAQRzYTvCZmSM8vtgu06g";

  public static String REQ_URL = "ws://127.0.0.1:8080/msg-center/websocket";

  public static WebSocketStompClient stompClient;

  public static MyStompSessionHandler sessionHandler;

  public static void main(String[] args) throws InterruptedException {

    long clientThreadNum = 1;
    if (args.length > 0 && !StringUtils.isEmpty(args[0])) {
      clientThreadNum = Long.valueOf(args[0]);
      System.out.println("客户端建立链接数:" + clientThreadNum);
    }

    if (args.length > 1 && !StringUtils.isEmpty(args[1])) {
      REQ_URL = "ws://" + args[1] + "/msg-center/websocket";
      System.out.println("远程连接地址:" + args[1]);
    }

    List<Transport> transports = new ArrayList<>(1);
    transports.add(new WebSocketTransport(new StandardWebSocketClient()));
    WebSocketClient transport = new SockJsClient(transports);
    stompClient = new WebSocketStompClient(transport);

    stompClient.setMessageConverter(new StringMessageConverter());
    sessionHandler = new MyStompSessionHandler();

    ExecutorService executorService = new ThreadPoolExecutor(50, 1000, 60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(500));

    // 每条线程处理链接数
    long singeDelNum = 100;
    while (clientThreadNum > 0) {
      long delNum = clientThreadNum > singeDelNum ? singeDelNum : clientThreadNum;
      try {
        executorService.submit(new DealThread(delNum));
      }
      catch (Exception e) {
        Thread.sleep(5000);
        continue;
      }
      clientThreadNum -= singeDelNum;
    }

    // 定时打印counter
    new Thread(() -> {
      while (true) {
        try {
          Thread.sleep(5000);
        }
        catch (InterruptedException e) {
          e.printStackTrace();
        }
        System.out.println("counter: " + sessionHandler.getCounter());

      }
    }).start();

    while (true) {
      Thread.sleep(Integer.MAX_VALUE);
    }

  }

  public static class DealThread implements Runnable {

    private long dealNum;

    public DealThread(long dealNum) {
      this.dealNum = dealNum;
    }

    @Override
    public void run() {
      int waitTime = 1;
      while (dealNum > 0) {

        try {
          Thread.sleep(waitTime);

          StompHeaders stompHeaders = new StompHeaders();
          stompHeaders.put("token", Arrays.asList(TOKEN));
          stompHeaders.put("projectId", Arrays.asList("fm"));

          // 设置心跳
          // stompClient.setTaskScheduler();
          // stompHeaders.setHeartbeat(new long[] { 10000, 10000 });

          ListenableFuture<StompSession> connect = stompClient.connect(new URI(REQ_URL), null, stompHeaders,
              sessionHandler);

          StompSession stompSession = connect.get();
          if (stompSession == null) {
            waitTime += 3000;
            continue;
          }
        }
        catch (Exception e) {
          System.out.println("this si errer ,retry:" + waitTime);
          e.printStackTrace();
          waitTime += 3000;
          continue;
        }

        // 走到后面说明连接成功,置回正常等待时间,处理下一位
        if (waitTime > 3000) {
          waitTime = 1;
        }
        dealNum--;
      }
      System.out.println(Thread.currentThread().getName() + " is die");
    }
  }
}
```
启动客户端时动态传入连接数进行压测
```
java -jar benchmark-1.0-SNAPSHOT.jar 1000 127.0.0.1
```

### 运行环境
>1台服务端：16G内存，8核CPU<br>
1台客户端：16G内存，8核CPU

开始默认配置，未对操作系统和Java虚拟机做任何参数调优。

### 测试历程(1024~500K)
虽然很多文章都介绍了C1000K在现代硬件机器的情况下都是很容易达到的。笔者在亲自试验后还是有很多坑存在，下面记录从1024到100W各个阶段踩坑的过程(一步一个坎)，出现问题发现问题的解决过程。

根据前两篇文章我们清楚，推送中心的服务端是用的Spring WebSocket STOMP框架，其实网络这一层都是用的Spring封装的框架，并没有深入研究，所以对于Spring WebSocket的实现是否能支持C1000K的连接，我们来测试一下吧！

>[github完整代码](https://github.com/shibd/msg-center/tree/master/benchmark)

#### 突破1024（调整最大文件描述符数量）
1台客户端压测虽然有65535的限制，期初先测试1W看看是否正常。在服务端正常启动后，客户端启动传入1W压测1W连接。发现客户端和服务端都在大概1024个连接后报错：`open file many`。

好了，我们遇到了服务器调优的第一个瓶颈：`操作系统最大文件符限制`，执行`ulimit -n`查看发现默认的进程的最大打开文件描述符为1024，和报错信息时的连接数一致。

**🚑解决办法**

⭐️ 系统最大打开文件描述符数：`/proc/sys/fs/file-max` <br>
- 临时设置：`echo 1000000 > /proc/sys/fs/file-max` <br>
- 永久设置：修改`/etc/sysctl.conf`文件，增加`fs.file-max = 1000000` <br>

⭐️ 进程最大打开文件描述符数: `ulimit -n`<br>
- 临时设置：`ulimit -n 1000000`。<br>
- 永久设置：修改`/etc/security/limits.conf`文件，增加下面的行<br>

```
*         hard    nofile      1000000
*         soft    nofile      1000000
root      hard    nofile      1000000
root      soft    nofile      1000000
```

还有一点要注意的就是hard limit不能大于`/proc/sys/fs/nr_open`，因此有时你也需要修改nr_open的值。
执行`echo 2000000 > /proc/sys/fs/nr_open`

#### 突破1W（更换Spring的Web容器）
服务端和客户端操作系统增加了文件描述符后，成功突破了1024个连接数，但是在连接数达到1W时，服务端出现假死状态，任何http请求查询无响应。观察服务端内存，负载情况都没问题。

⭐️ **操作系统还是应用程序问题**

期初一直怀疑是服务器有问题，怀疑是否是操作系统参数没调好导致的。所以就尝试在连接数达到1W后，在服务器上用`curl http:127.0.0.1:8080/msg-center`访问，发现请求照样阻塞，神奇的是，当客户端连接减少1个到9999时，其他请求就能打入。这种情况基本确定是某配置导致的，不是操作系统和应用程序出现了瓶颈。

另外使用`tcpdump -i lo`抓取本地回环网卡的包，发现在1W连接时虽然执行`curl http:127.0.0.1:8080/msg-center`阻塞，但是TCP三次握手是正常的，所以排除操作系统问题，基本确定是应用程序的问题。
```
07:00:18.997400 IP localhost.35750 > localhost.webcache: Flags [S], seq 1878590917, win 43690, options [mss 65495,sackOK,TS val 1152127886 ecr 0,nop,wscale 6], length 0
07:00:18.997412 IP localhost.webcache > localhost.35750: Flags [S.], seq 4293714672, ack 1878590918, win 43690, options [mss 65495,sackOK,TS val 1152127886 ecr 1152127886,nop,wscale 6], length 0
07:00:18.997421 IP localhost.35750 > localhost.webcache: Flags [.], ack 1, win 683, options [nop,nop,TS val 1152127886 ecr 1152127886], length 0
07:00:18.997485 IP localhost.35750 > localhost.webcache: Flags [P.], seq 1:89, ack 1, win 683, options [nop,nop,TS val 1152127886 ecr 1152127886], length 88: HTTP: GET /msg-center HTTP/1.1
07:00:18.997489 IP localhost.webcache > localhost.35750: Flags [.], ack 89, win 683, options [nop,nop,TS val 1152127886 ecr 1152127886], length 0
```

⭐️ **排查服务端Java程序**

1. 执行`jstat -gc pid`观察内存使用情况，发现内存使用正常，GC正常，没有负载，没有问题。
2. 执行`jstack pid`查看线程状态，发现在连接数达到1W时有大量的tomcat线程处于`WAITING`状态，其中一段有关`tomcat`的Acceptor的`countUpOrAwaitConnection`方法，看名字有点像接收器，难道是这个停止了吗？
![](/img/2019-08-15(Message-Center)/jstack1.jpg)

查看源码，done，找到问题，tomcat默认最大连接数是10000
``` java
    private int maxConnections = 10000;
    protected LimitLatch initializeConnectionLatch() {
        if (maxConnections==-1) return null;
        if (connectionLimitLatch==null) {
            connectionLimitLatch = new LimitLatch(getMaxConnections());
        }
        return connectionLimitLatch;
    }
```

**🚑解决办法**
1. 在application.yml中修改tomcat最大连接数`server.tomcat.max-connections: 1000000`。
2. 或者更换Spring默认的web容器为`undertow`，修改pom文件。
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
        </dependency>
```

#### 突破3W（增加操作系统端口范围）
修改了Spring的web容器为`undertow`，连接数突破了1W的限制，在望更高的链接数压测时，链接大概达到3W的时候，客户端报错，看信息是客户端端口申请不到了。
```
Caused by: java.net.BindException: Cannot assign requested address
        at sun.nio.ch.Net.connect0(Native Method)
        at sun.nio.ch.Net.connect(Net.java:454)
        at sun.nio.ch.Net.connect(Net.java:446)
        at sun.nio.ch.UnixAsynchronousSocketChannelImpl.implConnect(UnixAsynchronousSocketChannelImpl.java:326)
        at sun.nio.ch.AsynchronousSocketChannelImpl.connect(AsynchronousSocketChannelImpl.java:199)
        at org.apache.tomcat.websocket.WsWebSocketContainer.connectToServerRecursive(WsWebSocketContainer.java:305)
        ... 4 more
```
**🚑解决办法**

我们知道服务器端口是从0~65535，但是操作系统会预留很多端口出来，我们这里减少操作系统的预留端口即可。
```
vim /etc/sysctl.conf
net.ipv4.ip_local_port_range = 1024 65535
sysctl -p
```

#### 突破4W（调整JVM进程内存大小）
连接数达到4W时，客户端/服务端会报错OOM，通过`jstat -gc pid`观察Java进程，老年代GC特别频繁，堆内存不够用了。

**🚑解决办法**

加大JVM内存即可

`java -Xmx10G -Xms8G -XshowSettings:vm -jar benchmark-1.0-SNAPSHOT.jar 70000 139.217.99.53:8080`


#### 突破6W（客户端增加虚拟IP）
虽然外面在**突破3W**内容中增大了客户端的端口范围，但是单客户端端口上线为65535，在压满后照样会报错`Cannot assign requested address`，如果朝着服务端支持100W连接压测，那就需要10多台客户机才行。

**🚑解决办法**

可以增加更多的网卡，也可以使用虚拟IP来实现。比如可以使用命令增加20个IP地址，那么客户端就可以发起100多W条连接，可以使用下面命令给某网卡增加虚拟ip：
``` java
ifconfig eth0:0 192.168.10.10 netmask 255.255.255.0 up
ifconfig eth0:1 192.168.10.11 netmask 255.255.255.0 up
ifconfig eth0:2 192.168.10.12 netmask 255.255.255.0 up
```


#### 突破7W（调整服务端nf_conntrack_max）

当连接数达到6W多的时候, 客户端压测程序开始报错连接失败: `The HTTP request to initiate the WebSocket connection failed`。服务端日志无报错，使用`curl http:127.0.0.1:8080/msg-center`发现同样无响应，但是不同的是使用`tcpdump`没有抓到tcp握手的包。怀疑是从操作系统层面把连接丢弃了。

使用`dmesg`命令查看系统信息，发现有大量的`nf_conntrack: table full, dropping packet`日志，问题很明显，`nf_conntrack`模块报错丢弃了包。

**🚑解决办法**

`vim /etc/sysctl.conf`增加一行 `net.nf_conntrack_max = 2000000`，执行`sysctl -p`生效。

有些操作系统没有启动`nf_conntrack`模块，则不会遇到该问题。

#### 突破50W（调整tcp socket参数）

突破了7W的连接，本以为后面一番风顺，可以一直压了，大概到50W的时候，又出现了服务端假死的情况，同样适用`dmesg`命令查看，发现出现大量`TCP: too many of orphaned sockets`的错误，错误显示`tcp socket`过多。

**🚑解决办法**

这时候需要调整`tcp socket`参数了，关于参数的说明可以google学习。
```
echo "net.ipv4.tcp_mem = 786432 2097152 3145728">> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 4096 16777216">> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 4096 16777216">> /etc/sysctl.conf
```
执行`sysctl -p`生效，大功告成！

#### 调优总览
从1024到100W其实调整的参数也就没几个，简单总结一下。
- 更换Spring的Web容器为`undertow`。
- 服务端和客户端。
```
     echo "* - nofile 1048576" >> /etc/security/limits.conf
     echo "fs.file-max = 1048576" >> /etc/sysctl.conf
     echo "net.ipv4.ip_local_port_range = 1024 65535" >> /etc/sysctl.conf
	 echo "net.nf_conntrack_max = 2000000" >> /etc/sysctl.conf
 
     echo "net.ipv4.tcp_mem = 786432 2097152 3145728" >> /etc/sysctl.conf
     echo "net.ipv4.tcp_rmem = 4096 4096 16777216" >> /etc/sysctl.conf
     echo "net.ipv4.tcp_wmem = 4096 4096 16777216" >> /etc/sysctl.conf
```
- 增加服务端和客户端JVM的内存大小。


### 总结
到此，服务端通过调优，单体推送中心支持了百万级的连接，实际公司目前还没有这么大的规模。目前并没有测试百万连接下，推送数据，心跳保持是否正常。后续会测试再完善。

推送中心到此，设计和性能压测都已经完毕，但到现在为止还是单体的。那么推送中心如何做集群，集群模式下怎样共享数据，怎样高可用，怎样实现横向扩展，下章节会分享。

> [推送中心地址](https://github.com/shibd/msg-center)


<br>

参考文章:
-  [使用四种框架分别实现百万websocket常连接的服务器](https://colobu.com/2015/05/22/implement-C1000K-servers-by-spray-netty-undertow-and-node-js/#%E6%9C%80%E5%A4%A7%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6)
- [100万并发连接服务器笔记之1M并发连接目标达成](http://www.blogjava.net/yongboy/archive/2013/04/11/397677.html)