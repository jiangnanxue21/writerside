# Kafka

## PageCache

app不能直接访问cpu网卡和硬盘，要通过kernel系统调用，修改PageCache之后标记为脏，可以通过调优参数，什么时候flush到磁盘。

![](pagecache.jpg)

| 参数                           | 含义                                                                                                      |
|------------------------------|---------------------------------------------------------------------------------------------------------|
| vm.dirty_background_ratio    | 内存可以填充脏数据的百分比。脏数据稍后会写入磁盘，pdflush/flush/kdmflush这些后台进程会清理脏数据。有32G内存，超过3.2G的话就会有后台进程来清理                   |
| vm.dirty_background_bytes    | 如果设置_bytes版本，则_ratio版本将变为0，反之亦然                                                                         |
| vm.dirty_ratio               | 脏数据填充的绝对最大系统内存量，当系统到达此点时，必须将所有脏数据提交到磁盘，同时所有新的I/O块都会被阻塞，直到脏数据被写入磁盘。这通常是长I/O卡顿的原因，但这也是保证内存中不会存在过量脏数据的保护机制 |
| vm.dirty_bytes               | 如果设置_bytes版本，则_ratio版本将变为0，反之亦然                                                                         |
| vm.dirty_writeback_centisecs | 指定多长时间 pdflush/flush/kdmflush 这些进程会唤醒一次，然后检查是否有缓存需要清理                                                   |
| vm.dirty_expire_centisecs    | 指定脏数据能存活的时间。在这里它的值是30秒。当 pdflush/flush/kdmflush 在运行的时候，他们会检查是否有数据超过这个时限，如果有则会把它异步地写到磁盘中                 |

## DMA

DMA技术就是我们在主板上放一块独立的芯片。在进行内存和 I/O 设备的数据传输的时候，我们不再通过 CPU 来控制数据传输，而直接通过
DMA 控制器（DMA Controller，简称 DMAC）。这块芯片，我们可以认为它其实就是一个协处理器（Co-Processor）。

DMAC 其实也是一个特殊的 I/O 设备，它和 CPU 以及其他 I/O
设备一样，通过连接到总线来进行实际的数据传输。总线上的设备呢，其实有两种类型。一种我们称之为主设备（Master），另外一种，我们称之为从设备（Slave）。

![](DMA.png)

数据传输的流程如下：

1. CPU 还是作为一个主设备，向 DMAC 设备发起请求。这个请求，其实就是在 DMAC 里面修改配置寄存器
2. 如果我们要从硬盘上往内存里面加载数据，这个时候，硬盘就会向 DMAC 发起一个数据传输请求。这个请求并不是通过总线，而是通过一个额外的连线。
3. DMAC 需要再通过一个额外的连线响应这个申请
4. DMAC 这个芯片，就向硬盘的接口发起要总线读的传输请求。数据就从硬盘里面，读到了 DMAC 的控制器里面
5. DMAC 再向我们的内存发起总线写的数据传输请求，把数据写入到内存里面
6. DMAC 会反复进行上面第 4、5 步的操作，直到 DMAC 的寄存器里面设置的数据长度传输完成

Kafka 里面会有两种常见的海量数据传输的情况。一种是从网络中接收上游的数据，然后需要落地到本地的磁盘上，确保数据不丢失。另一种情况呢，则是从本地磁盘上读取出来，通过网络发送出去
而后一种情况的 零拷贝就是通过DMA完成的，具体流程参见Picture 1.1

## 原始论文（Kafka: a Distributed Messaging System for Log Processing）

在abstract里面写到Our system
incorporates ideas from existing log aggregators and messaging
systems, and is suitable for both offline and online message
consumption，并且有着高性能、高可用的数据传输

两个要点：

### 为什么我们需要 Kafka 这样一个桥梁，连接起我们的应用系统、大数据批处理系统，以及大数据流式处理系统

![](Scribe.png)

facebook开源的日志收集器Scribe，存在一些问题

1. 如果需要收集1min的日志，需要每分钟执行一个 MapReduce 的任务，不是一个高效的解决问题的办法
2. 隐式依赖：可能会出现网络中断、硬件故障等等的问题，所以我们很有可能，在运行 MapReduce 任务去分析数据的时候，Scribe 还没有把文件放到
   HDFS 上。那么，我们的 MapReduce 分析程序，就需要对这种情况进行容错，比如，下一分钟的时候，它需要去读取最近 5 分钟的数据，看看
   Scribe 上传的新文件里，是否会有本该在上一分钟里读到的数据。而这些机制，意味着下游的 MapReduce
   任务，需要去了解上游的日志收集器的实现机制。并且两边根据像文件名规则之类的隐式协议产生了依赖，这就使得数据分析程序写起来会很麻烦，维护起来也不轻松

### 其次是 Kafka 系统的设计，和传统的“消息队列”系统有什么不同，以及为什么 Kafka 需要这样来设计

1. 而传统的消息队列，则关注的是小数据量下，是否每一条消息都被业务系统处理完成了。因为这些消息队列里的消息，可能就是一笔实际的业务交易，我们需要等待
   consumer 处理完成，确认结果才行。但是整个系统的吞吐量却没有多大。
2. 而像 Scribe 这样的日志收集系统，考虑的是能否高吞吐量地去传输日志，至于下游如何处理日志，它是不关心的。
3. 而 Kafka
   的整体设计，则主要考虑的是我们不仅要实时传输数据，而要开始实时处理数据了。我们需要下游有大量不同的业务应用，去消费实时产生的日志文件。并且，这些数据处理也不是非常重要的金融交易，而是对于大量数据的实时分析，然后反馈到线上。

#### 无消息丢失配置怎么实现？

一句话概括，Kafka 只对“已提交”的消息（committed message）做有限度的持久化保证

“消息丢失”案例

1. 生产者端
    - 设置acks=all。代表了对“已提交”消息的定义
    - Producer要使用带有回调通知producer.send(msg,callback)的发送API，不要使用producer.send(msg)
      。一旦出现消息提交失败的情况，可以有针对性地进行处理
    - 设置retries为一个较大的值。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了retries > 0的Producer
      能够自动重试消息发送，避免消息丢失
2. 消费者端
    - 维持先消费消息，再更新位移
    - enable.auto.commit=false，手动提交位移
3. broker端
    - 设置replication.factor>= 3，目前防止消息丢失的主要机制就是冗余
    - unclean.leader.election.enable=false。控制哪些Broker有资格竞选分区的Leader。不允许一个Broker落后原先的Leader太多当Leader，

Netty特性:

1. 异步事件驱动( asynchronous event-driven )
2. 可维护性(maintainable)
3. 高性能协议服务器和客户端构建(high performance protocol servers & clients)

hbase面试题
https://baijiahao.baidu.com/s?id=1719816304849626834&wfr=spider&for=pc

## 4. Producer

1. 生产者消息分区机制
2. meta更新策略
3. 如何和broker建立连接
4. 内存管理和分配
5. kafka的NIO模型

### 4.1 数据分区分配策略

分区策略是决定生产者将消息发送到哪个分区的算法

1. Round-Robin
2.

## Controller

在ZooKeeper的帮助下管理和协调整个Kafka

1. 选举控制器的规则
   第一个成功创建/controller节点的Broker会被指定为控制器
2. 控制器是做什么？
    - 主题管理（创建、删除、增加分区）
    - 分区重分配
    - Preferred 领导者选举
    - 集群成员管理（新增Broker、Broker主动关闭、Broker宕机）
    - 数据服务：控制器上保存了最全的集群元数据信息，其他所有Broker会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据
3. 控制器故障转移（Failover）
   当运行中的控制器突然宕机或意外终止时，Kafka能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器

![](ControllerFailover.png)

Broker 0是控制器。当Broker 0宕机后，ZooKeeper通过Watch机制感知到并删除了/controller临时节点

### 实现

把多线程的方案改成了单线程加事件队列的方案

Controller是在KafkaServer.scala#startup中初始化并且启动的

```Java
  /* start kafka controller */
  _kafkaController = new KafkaController(config, zkClient, time, metrics, brokerInfo, brokerEpoch, tokenManager, brokerFeatures, metadataCache, threadNamePrefix)
  kafkaController.startup()
```

```Java
// 第1步：注册ZooKeeper状态变更监听器，它是用于监听Zookeeper会话过期的
 zkClient.registerStateChangeHandler(new StateChangeHandler {
   override val name: String = StateChangeHandlers.ControllerHandler
   override def afterInitializingSession(): Unit = {
     eventManager.put(RegisterBrokerAndReelect)
   }
   override def beforeInitializingSession(): Unit = {
     val queuedEvent = eventManager.clearAndPut(Expire)

     // Block initialization of the new session until the expiration event is being handled,
     // which ensures that all pending events have been processed before creating the new session
     queuedEvent.awaitProcessing()
   }
 })
 
 // 第2步：写入Startup事件到事件队列
 eventManager.put(Startup)
 
 // 第3步：启动ControllerEventThread线程，开始处理事件队列中的ControllerEvent
 eventManager.start()
```

这里主要看下ControllerEventManager eventManager是Controller事件管理器，负责管理事件处理线程

## 延时操作模块

因未满足条件而暂时无法被处理的Kafka请求

举个例子，配置了acks=all的生产者发送的请求可能一时无法完成，因为Kafka必须确保ISR中的所有副本都要成功响应这次写入。因此，通常情况下，这些请求没法被立即处理。只有满足了条件或发生了超时，Kafka才会把该请求标记为完成状态

### 怎么实现延时请求呢？

#### 1. 为什么不用Java的DelayQueue

DelayQueue有一个弊端：它插入和删除队列元素的时间复杂度是O(logN)。对于Kafka这种非常容易积攒几十万个延时请求的场景来说，该数据结构的性能是瓶颈

```Java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {

    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    
    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader;
    
     /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     */
    private final Condition available = lock.newCondition();
```

这边说用了Leader-Follower
pattern，刚开始没看懂，[stackoverflow](https://stackoverflow.com/questions/48493830/what-exactly-is-the-leader-used-for-in-delayqueue)
说的比较清楚。
Leader的用处主要是minimize unnecessary timed
waiting，其实是多个线程不必要的唤醒和睡眠。如果让所有线程都available.awaitNanos(delay)
进入take方法，它们将同时被调用，但只有一个可以真正从队列中获取元素，其他线程将再次陷入休眠，这是不必要的且浪费资源。

采用Leader-Follower模式，Leader available.awaitNanos(delay)、Follower available.await()
。因此Leader将首先醒来并检索过期的元素，然后在必要时向另一个等待线程发出信号。这样效率更高

主要的实现是take和offer方法

```Java
public boolean offer(E e) {
     final ReentrantLock lock = this.lock;
     lock.lock();
     try {
         q.offer(e); // q保存的是Delayed类型的数据，根据Comparable比较的小根堆
         if (q.peek() == e) { // 如果插入的元素在堆顶，唤醒其他可能在等待的线程
             leader = null;
             available.signal();
         }
         return true;
     } finally {
         lock.unlock();
     }
 }
```

take方法的实现

```Java
 public E take() throws InterruptedException {
     final ReentrantLock lock = this.lock;
     lock.lockInterruptibly();
     try {
         for (;;) {
             E first = q.peek();
             if (first == null)
                 available.await();
             else {
                 long delay = first.getDelay(NANOSECONDS);
                 if (delay <= 0L)
                     return q.poll();
                 first = null; // don't retain ref while waiting
                 if (leader != null) // leader在则等leader线程先获取，等待唤醒
                     available.await();
                 else {
                     Thread thisThread = Thread.currentThread();
                     leader = thisThread;
                     try {
                         available.awaitNanos(delay); // leader拿到的堆顶的元素，等待delay时间
                     } finally {
                         if (leader == thisThread)
                             leader = null;
                     }
                 }
             }
         }
     } finally {
         if (leader == null && q.peek() != null)
             available.signal();
         lock.unlock();
     }
 }
```

# MapReduce

![](mapreduce.png)

Map:以一条记录为单位做映射

Reduce:以一组为单位做计算

## MapReduce在Hadoop中的工作机制

- Client提交规划：支撑了计算向数据移动和并行度
- MapTask
- ReduceTask

# 网络模型

### BIO

![](BIO.png)

1. Socket创建文件描述符fd3
2. Socket绑定fd3和网络端口号
3. 监听
4. 在accept处阻塞，一旦有链接则创建一个线程执行

BIO的问题：如果客户端链接过大那么需要新建若干个线程去执行，每台服务器可以运行的线程数是有限的。那么多线程的上下文切换的消耗也是巨大的

### NIO

两种意思：JDK new IO/OS non-blocking

[//]: # (![]&#40;NIO.png&#41;)

NIO的问题： 每次都有无意义的系统调用 O(n)

多路复用器的引入

1. 加入master节点的流程是什么？
2. 如果本节点被选为master，接下去做什么
3. 普通节点如何监控master健康状态
4. master如何监控普通节点健康状态
5. ES如何避免脑裂？它有哪些举措？