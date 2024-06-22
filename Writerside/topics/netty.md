# Netty

分成以下几部分去总结：
1. 网络基础
2. 事件循环组
3. 内存管理系统
4. 编码器

## 网络基础

![网络基础示意图.png](网络基础示意图.png)

### 网络IO变化，模型

同步，异步，阻塞，非阻塞

按照同步的流程

在accept和recv阶段会产生阻塞
![网络同步.png](网络同步.png)


- BIO

![BIO.png](BIO.png)

BIO的阻塞在哪里
1. accept的时候就一直在等待着客户端的连接，这个等待过程中主线程就一直在阻塞
2. 连接建立之后， clone thread的时候会系统调用，在读取到socket信息之前，线程也是一直在等待，一直处于阻塞的状态下

![阻塞IO.png](阻塞IO.png)

当客户端并发访问量增加后，服务端的线程个数和客户端并发访问数呈 1:1 的正比关系，线程数量快速膨胀后，系统的性能将急剧下降

thread池化的问题：如果发生读取数据较慢时（比如数据量大、网络传输慢等），慢的连接会阻塞占用线程，而其他接入的消息，只能一直等待

- NIO
1. JDK new IO
2. Non-blocking

![NIO.png](NIO.png)
```Java
LinkedList<SocketChannel> clients = new LinkedList<>();
ServerSocketChannel ss = ServerSocketChannel.open(); // 服务端开启监听：接受客户端
ss.bind(new InetSocketAddress(9090));
ss.configureBlocking(false); // 重点: OS NONBLOCKING!!!, 只让接受客户端, 不阻塞

while (true) {
  // 接受客户端的连接
  SocketChannel client = ss.accept(); 
  if (client == null) {
    // 如果没有连接，直接返回null
   } else {
    // 读取数据流程
}
```

NIO慢在哪里：

while循环中，每次都需要全量遍历，用户态内核切换才能实现
![NIO示意.png](NIO示意.png)

因此引入了多路复用器：select， poll， epoll

多路复用器的两个好处：

1. 该模型能够在同一个线程内同时监听多个IO请求，系统不必创建大量的线程，从而大大减小了系统的开销
2. 等待的方式能减少无效的系统调用，减少了对CPU资源的消耗

![selector.png](selector.png)

linux以及netty

同步阻塞：程序自己读取，调用了方法一直等待有效返回结果（BIO）

同步非阻塞：程序自己读取，调用方法一瞬间，给出是否读到（NIO）

- select

select有fd大小的限制，而poll没有，FD_SETSIZE(1024)
    ![select.png](select.png)
    
> 无论NIO,SELECT,POLL都是要遍历所有的IO，并且询问状态；
> 只不过，NIO遍历的成本在用户态内核态切换，
> 而SELECT,POLL只触发了一次系统调用，把需要的fds传递给内核，内核重新根据用户这次调用传过来的fds，遍历修改状态
    
    
select的问题
1. 每次都要重新传递fds
2. 每次内核被调用之后，针对这次调用，触发一个遍历fds全量的复杂度

- epoll
![epoll.png](epoll.png)

三个函数：
1. epoll_create: 开辟一个红黑树空间
2. epoll_ctl: 将fd注册到红黑树
3. epoll_wait：查找事件链表相关fd的事件

时间复杂度：

select, poll: O(n)

epoll : O(1)

#### Java示例
```Java
public class SocketMultiplexingSingleThreadv1 {
    private ServerSocketChannel server = null;
    // 可以通过-D指定
    // -Djava.nio.channels.spi.SelectorProvider=sun.nio.ch.EPollSelectorProvider
    private Selector selector = null; //linux多路复用器（可以是select poll epoll）
    int port = 9090;

    public void initServer() {
        try {
            server = ServerSocketChannel.open();
            server.configureBlocking(false);
            server.bind(new InetSocketAddress(port));

            // 如果在epoll模型下，epoll_create -> fd3
            selector = Selector.open(); // 优先选择：epoll

            // select，poll:jvm里开辟一个数组fd4放进去
            // epoll：epoll_ctl(fd3,ADD,fd4,EPOLLIN
            server.register(selector, SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void start() {
        initServer();
        try {
            while (true) {  //死循环
                Set<SelectionKey> keys = selector.keys();
                System.out.println(keys.size()+"size");
                //1,调用多路复用器(select,poll or epoll(epoll_wait))
                /*
                select():
                1，select，poll: select(fd4), poll(fd4)
                2，epoll：epoll_wait()
                参数可以带时间：没有时间，0:阻塞，有时间设置一个超时
                selector.wakeup() 结果返回0

                懒加载:
                其实再触碰到selector.select()调用的时候触发了epoll_ctl的调用
                 */
                while (selector.select() > 0) {
                    Set<SelectionKey> selectionKeys=selector.selectedKeys(); //返回的有状态的fd集合
                    Iterator<SelectionKey> iter=selectionKeys.iterator();
                    // 多路复用器只给状态，还得一个个的去处理R/W
                    // NIO对每一个fd调用系统调用，浪费资源，这里调用了一次select方法，知道那些可以R/W
                    while (iter.hasNext()) {
                        SelectionKey key = iter.next();
                        iter.remove(); //set 不移除会重复循环处理
                        if (key.isAcceptable()) {
                            //看代码的时候，这里是重点，如果要去接受一个新的连接
                            //语义上，accept接受连接且返回新连接的FD
                            //那新的FD怎么办？
                            //select，poll，因为他们内核没有空间，那么在jvm中保存和前边的fd4那个listen的一起
                            //epoll：希望通过epoll_ctl把新的客户端fd注册到内核空间
                            acceptHandler(key);
                        } else if (key.isReadable()) {
                            readHandler(key); //read还有write都处理了
                            //在当前线程，这个方法可能会阻塞,所以，为什么提出了IO THREADS
                            //redis是不是用了epoll，redis是不是有个io threads的概念，redis是不是单线程的
                        }  else if(key.isWritable()){ 
                            //写事件<-- send-queue(netstat -anp), 只要是空的，就一定会返回可以写的事件，就会回调写方法
                            //什么时候写？不是依赖send-queue是不是有空间
                            key.cancel();
                            key.interestOps(key.interestOps() & ~SelectionKey.OP_WRITE);
                            writeHandler(key);
                        }
                    }
                }
            }
        }
    }

    public void acceptHandler(SelectionKey key) {
        try {
            ServerSocketChannel ssc = (ServerSocketChannel)key.channel();
            SocketChannel client = ssc.accept(); //目的是调用accept接受客户端fd7
            client.configureBlocking(false);

            ByteBuffer buffer = ByteBuffer.allocate(8192);

            //调用了register
            /*
            select，poll：jvm里开辟一个数组fd7放进去
            epoll：epoll_ctl(fd3,ADD,fd7,EPOLLIN
             */
            client.register(selector, SelectionKey.OP_READ, buffer);
        }
    }

    public void readHandler(SelectionKey key) {...}
```

写事件处理：

send-queue(netstat -anp可以显示), 只要是空的，就一定会返回可以写的事件，就会回调写方法

什么时候写？不是依赖send-queue是不是有空间
1. 你准备好要写什么了，这是第一步
2. 第二步你才关心send-queue是否有空间
3. so，读 read 一开始就要注册，但是write依赖以上关系，什么时候用什么时候注册
4. 如果一开始就注册了write的事件，进入死循环，一直调起

## Reactor反应器模式

也叫做分发者模式或通知者模式，是一种将就绪事件派发给对应服务处理程序的事件设计模式

### 架构

![netty架构.png](netty架构.png)

Netty is an **asynchronous event-driven** network application framework
for rapid development of **maintainable high performance protocol** servers & clients.

#### 前置知识

操作系统想和JVM沟通，先从堆内存放到buffer

下图是readBytes的原理图：
![readBytes.png](readBytes.png)

在JVM内存中分配的空间为DirectByteBuffer，在堆内存中开辟的空间为HeapByteBuffer

### 事件循环组
![事件循环模型.png](事件循环模型.png)

如何异步执行？
1. Future接口
2. 通过Future接口获取任务执行结果即可

Future弊端：总得调用方来获取再执行步骤，如何解决？

使用观察者模式，当执行任务完成时，自动在执行线程回调callback ----》 Promise, 加了监听器

事件循环组的线程应该有哪些特性？
1. 负载均衡
2. 周期性调度的工作

```Java
// EventExecutor继承EventExecutorGroup，是一个特殊的组，next指向他自己
public interface EventExecutor extends EventExecutorGroup {
   /**
     * Returns a reference to itself.
     * 永远指向自己
     */
   @Override
    EventExecutor next();
```

EventExecutorGroup组里面有多个线程，形成循环
```Java
public interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor>
```

```Java
// EventLoop是一个线程，但也是一个特殊的循环组
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup 
```

EventExecutorGroup继承自ScheduledExecutorService拥有了调度执行的功能，并且通过next()获取一个EventExecutor，EventExecutor是EventExecutorGroup里面专门执行事件的执行器，
是一个特殊的EventExecutorGroup。引入EventLoopGroup，注册channel，而且把线程连起来，形成循环组；EventLoop是一个线程，但也是一个特殊的循环组


```Java
// chooser选择children里面的EventExecutor
public abstract class MultithreadEventExecutorGroup extends AbstractEventExecutorGroup {

    private final EventExecutor[] children;
    private final Set<EventExecutor> readonlyChildren;
    private final AtomicInteger terminatedChildren = new AtomicInteger();
    private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
    private final EventExecutorChooserFactory.EventExecutorChooser chooser;
    
    @Override
    public EventExecutor next() {
        return chooser.next();
    }

```

Chooser的实现



**一致性的问题是由多副本引出的**




