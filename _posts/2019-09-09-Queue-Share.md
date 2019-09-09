---
layout:     post
title:      无处不在的队列
subtitle:   队列数据结构在语言，各种中间件架构中的应用
date:       2019-09-09
author:     baozi
header-img: img/2019-08-14(RocketMQ-Phoenix)/top-ux.jpg
catalog: true 						
tags:								
    - Java
    - 数据结构
    - 随笔
---

# 队列定义
先进者先出

![image1.jpg](/img/2019-09-09(Queue-Share)/image1.jpg)

# 内存队列
顺序队列：使用数组实现的队列<br />
链式队列：使用链表实现的队列<br />
循环队列：避免顺序队列下队列满的数据搬移，动态计算tail和head指针<br />
阻塞队列：入队满阻塞，出队空阻塞<br />
并发队列：支持并发入队和并发出队<br />
双端队列：对头队尾都支持入队和出队操作。既是生产者又是消费者<br />有界队列：队列大小固定<br />
无界队列：队列大小无固定<br />
优先队列：实际不是队列<br />[]()

## 应用场景
### 生产者-消费者（阻塞并发队列）

#### 单消费者
1. 解耦
1. 异步
1. 流水线模型：地铁调度模型

会议室系统的邮件发送

```java
/**
 * @Auther: baozi
 * @Date: 2019/6/4 11:01
 * @Description:
 */
abstract class MailSend {

    /**
     * 邮件缓存队列
     */
    private static BlockingQueue<Object> queue = new ArrayBlockingQueue<>(100);

    /**
     * 线程运行标志
     */
    private static AtomicBoolean run = new AtomicBoolean(false);

    /**
     * 发送邮件线程
     */
    private static Thread sendThread;

    /**
     * 开始发送线程
     * @return
     */
    public static boolean start() {
        if (!run.compareAndSet(false, true)) {
            return false;
        }

        sendThread = new Thread(() -> {

            while (run.get()) {
                try {
                    Object take = queue.take();
                    // todo do send mail
                    // MailUtils.sendMail(take); 线程安全? 
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }

        });
        sendThread.start();
        return true;
    }

    /**
     * 通知发送线程
     * @return
     */
    public static boolean stop() {
        if (!run.compareAndSet(true, false)) {
            return false;
        }
        sendThread.interrupt();
        return true;
    }

    /**
     * 发送邮件
     * @param object
     */
    public static void put(Object object) throws InterruptedException {
        queue.put(object);
    }

    public static void main(String[] args) throws InterruptedException {

        // 开始邮件发送者线程
        MailSend.start();

        // doing
        MailSend.put(new Object());
    }
}
```

#### _多消费者（EventBus）_
![image2.jpg](/img/2019-09-09(Queue-Share)/image2.jpg)

### 线程池（阻塞并发队列）
![image4.jpg](/img/2019-09-09(Queue-Share)/image4.jpg)

**线程池创建**

JDK其中阻塞队列：[https://www.cnblogs.com/konck/p/9473677.html](https://www.cnblogs.com/konck/p/9473677.html)

```java
public ThreadPoolExecutor(int corePoolSize, // 核心线程数
                              int maximumPoolSize, // 线程池允许最大的线程数
                              long keepAliveTime, // 空闲线程超时时间
                              TimeUnit unit, // 枚举时间单位
                              BlockingQueue<Runnable> workQueue) //阻塞队列
```

**队列在线程池中的应用**：队列存放任务，利用队列实现线程复用，利用阻塞线程的超时来实现非核心线程的退出，以及核心线程和非核心线程的转换。下面为线程池中工作线程源码。

```java
private final class Worker implements Runnable {
 
	final Thread thread;
    
	Runnable firstTask;
    
	Worker(Runnable firstTask) {
		this.firstTask = firstTask;
		this.thread = getThreadFactory().newThread(this);
	}
    
	public void run() {
		runWorker(this);
	}
	
	final void runWorker(Worker w) {
		Runnable task = w.firstTask;
		w.firstTask = null;
		while (task != null || (task = getTask()) != null){
		task.run();
	}
        
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
}
```

### 批量处理（阻塞并发队列）
**先决**：数据一致性不高，可靠性不高业务

例如数据库异步批量插入，日志批量写入ElasticSearch，Kafka批量发送模型，监控框架批量上报等

```java
public class MessageCache {

    private static final Logger LOG = LoggerFactory.getLogger(Sender.class);

    protected BlockingQueue<ProducerBatch> queue = new ArrayBlockingQueue<>(10000);

    public void addMessage(Message message) {
        try {
            queue.put(meesage);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            LOG.error(e.getMessage(), e);
        }
    }
    /**
     * 获取缓存策略,能拿就拿,拿不到null就返回,最多拿的数量不大于maxSendRecords
     * 这里存在一种场景:如果消费者线程大于生产者线程速率,那么就永远不会拿到批量的数据.
     * 在实际场景中rocketmq的同步发送是比较慢的,所以不会出现上述情况
     * @return
     */
    public List<Message> getMessages() {
        List<Message> messages = new ArrayList<>(maxSendRecords);
        ProducerBatch producerBatch = queue.peek();
        // 上来就没消息,就休息1ms
        if (producerBatch == null) {
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                LOG.error(e.getMessage(), e);
            }
            return messages;
        }
        long size = 0;
        // 发送条件判断: 拿不到数据立即发 | 判断消息个数 | 判断总消息大小
        while (producerBatch != null && messages.size() < maxSendRecords ) {
            ProducerBatch batch = queue.poll();
            messages.add(batch.getMessage());
            size = producerBatch.getSize() + size;

            if(size > LIMIT_SIZE) {
                break;
            }

            producerBatch = queue.peek();
        }
        return messages;
    }
}
```

### 管程模型（阻塞并发队列 ）

synchronized，wait-notify实现, AQS，ReentryLock&Condition等

![image5.jpg](/img/2019-09-09(Queue-Share)/image5.jpg)

### 工作密取（双端队列）

    在生产者-消费者设计中，所有消费者共享一个工作队列，而在工作密取中，每个消费者都有各自的双端队列。如果一个消费者完成了自己双端队列中的全部工作，那么他就可以从其他消费者的双端队列末尾秘密的获取工作。具有更好的可伸缩性，这是因为工作者线程不会在单个共享的任务队列上发生竞争。

    在大多数时候，他们都只是访问自己的双端队列，从而极大的减少了竞争。当工作者线程需要访问另一个队列时，它会从队列的尾部而不是头部获取工作，因此进一步降低了队列上的竞争。

## Disruptor队列

[https://blog.csdn.net/zhouzhenyong/article/details/81303011](https://blog.csdn.net/zhouzhenyong/article/details/81303011)

    Disruptor是一款高性能的有界阻塞并发内存队列，期初是英国外汇交易中心LMAX架构开发用来做交易的，单线程能支持600W QPS。目前很多知名框架都使用了Disruptor，比例Storm，Log4J2，HBase等。高性能的主要几个原因为：

1. 内存分配合理，使用RingBuffer数据结构，数组元素在初始化时一次性全部创建，提升缓存命中率；对象循环利用，避免频繁GC。（空间局部性原则）

![image6.jpg](/img/2019-09-09(Queue-Share)/image6.jpg)

2. 避免伪共享，提升缓存命中率；
2. 采用无锁算法（写时拷贝），避免频繁加锁、解锁的性能损耗；
2. 支持批量消费，消费者可以无锁方式消费多个消息

# 消息队列
业界产品：ActiveMQ，RabbitMQ，RocketMQ，Kafka，MetaMQ

## 应用场景
解耦，最终一致性，广播，削峰填谷，异步通知。

## 功能点
高可用，可靠投递，重复消息，顺序消息，push还是pull

## Kafka
kafka是一个高可用，数据高可靠，高吞吐，低延迟的中心化消息队列<br />

### 系统架构图
![image9.jpg](/img/2019-09-09(Queue-Share)/image9.jpg)

### Partition多副本
![image10.jpg](/img/2019-09-09(Queue-Share)/image10.jpg)

### 文件存储
    Kafka中消息是以topic进行分类的，生产者通过topic向Kafka broker发送消息，消费者通过topic读取数据。然而topic在物理层面又能以partition为分组，一个topic可以分成若干个partition，那么topic以及partition又是怎么存储的呢？partition还可以细分为segment，一个partition物理上由多个segment组成，那么这些segment又是什么呢？<br />[]()

### 发送模型
![image11.jpg](/img/2019-09-09(Queue-Share)/image11.jpg)

1. 和kafka交互获取到broker的元数据信息（往哪个broker发），获取不到就抛出异常
1. 判断消息包大小是否符合，默认为1MB，如果超过该大小则抛异常，修改配置需要和broker一起配合修改，具体看文章[https://blog.csdn.net/hanjibing1990/article/details/51673540](https://blog.csdn.net/hanjibing1990/article/details/51673540)
1. 往客户端内部的双端队列send消息，队列默认大小为32M，当大小不够时则会阻塞，阻塞时间可配，当阻塞时间过后则会抛出异常给客户端。

    同时有一个Sender线程会实时批量拉取双端队列内消息往kafka集群发送。如果发送成功，则进入回调函数，返回RecordMetadate对象。如果发送失败，则进行重试，重试次数可配，当重试次数用完，则进入客户端定义的回调函数由客户端处理。

### 参数配置

#### ProducerConfig.ACKS_CONFIG
指定必须有多少个**分区副本**收到消息，生产者才会认为**消息写入是成功**的。程序中设置为-1

1. ack=0：生产者不会等待任何来自服务器的响应。<br />如果当中出现问题，导致服务器没有收到消息，那么生产者无从得知，会造成消息丢失<br />由于生产者不需要等待服务器的响应所以可以以网络能够支持的最大速度发送消息，从而达到很高的吞吐量
1. acks=1（默认值）：只要集群的Leader节点收到消息，生产者就会收到一个来自服务器的成功响应<br />如果消息无法到达Leader节点（例如Leader节点崩溃，新的Leader节点还没有被选举出来）生产者就会收到一个错误响应，为了避免数据丢失，生产者会重发消息<br />_如果一个没有收到消息的节点成为新Leader，消息还是会丢失_<br />此时的吞吐量主要取决于使用的是同步发送还是异步发送，吞吐量还受到发送中消息数量的限制，例如生产者在收到服务器响应之前可以发送多少个消息
1. acks=-1：只有当所有参与复制的节点全部都收到消息时，生产者才会收到一个来自服务器的成功响应<br />这种模式是最安全的，可以保证不止一个服务器收到消息，就算有服务器发生崩溃，整个集群依然可以运行<br />延时比acks=1更高，因为要等待不止一个服务器节点接收消息

#### **ProducerConfig.RETRIES_CONFIG**
生产者从服务器收到临时性错误（如分区找不到Leader）时，retries参数决定了生产者可以重发消息的次数，程序中设置为 Integer.MAX
1. 默认情况下，生产者会在每次重试之间等待100ms，控制参数为`retry.backoff.ms`
2. 可以先测试一下恢复一个崩溃节点需要多少时间，假设为T，让生产者总的重试时间比T长，否着生产者会_过早地放弃重试_有些错误不是临时性错误，没办法通过重试来解决（例如消息太大），这个可以通过配置来改变producer，consumer以及broker可接受的的消息大小。（默认可接受的单个消息大小为1MB）一般情况下，因为生产者会自动进行重试。当出现不可重试的错误或者重试次数超过上限的情况时，现在的处理逻辑是打印错误信息，继续重试。client.id 任意字符串，服务器会用它来识别消息的来源，还可以用在日志和配额指标里。暂时没有配置 关于配置的相关介绍：[http://www.aboutyun.com/thread-24147-1-1.html](http://www.aboutyun.com/thread-24147-1-1.html)

#### ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION
Producer的另一个问题是消息的乱序问题。假设客户端代码依次执行下面的语句将两条消息发到相同的分区
```java
producer.send(record1);
producer.send(record2);
```
如果此时由于某些原因(比如瞬时的网络抖动)导致record1没有成功发送，同时Kafka又配置了重试机制和max.in.flight.requests.per.connection大于1(默认值是5，本来就是大于1的)，那么重试record1成功后，record1在分区中就在record2之后，从而造成消息的乱序。

#### 推荐配置
```java
// 只有当所有参与复制的节点**全部都收到消息时，生产者才会收到一个来自服务器的成功响应
props.put(ProducerConfig.ACKS_CONFIG, "-1");
// 失败重试的次数(每次间隔100ms重试次数用完需要10多年)
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
//限制客户端在单个连接上能够发送的未响应请求的个数。设置此值是1表示kafka broker在响应请求之前client不能再向同一个broker发送请求
props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 1);
// 缓冲区已满或元数据不可用时的阻塞时间
props.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 1000);
```

### 相关链接
- Java线程池： [https://silencedut.github.io/2016/06/25/从使用到原理学习Java线程池/](https://silencedut.github.io/2016/06/25/%E4%BB%8E%E4%BD%BF%E7%94%A8%E5%88%B0%E5%8E%9F%E7%90%86%E5%AD%A6%E4%B9%A0Java%E7%BA%BF%E7%A8%8B%E6%B1%A0/)
- 并发队列：[https://www.cnblogs.com/konck/p/9473677.html](https://www.cnblogs.com/konck/p/9473677.html)
- 消息队列详解：[https://tech.meituan.com/2016/07/01/mq-design.html](https://tech.meituan.com/2016/07/01/mq-design.html)
- kafka详解：[https://hiddenpps.blog.csdn.net/article/details/71091774](https://hiddenpps.blog.csdn.net/article/details/71091774)