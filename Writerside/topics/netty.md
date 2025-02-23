# Netty

分成以下几部分去总结：
1. 网络基础
2. 事件循环组
3. 内存管理系统
4. 编码器

## 1. 网络基础

![网络基础示意图.png](网络基础示意图.png)

### 网络IO变化，模型

同步，异步，阻塞，非阻塞

按照同步的流程

在accept和recv阶段会产生阻塞
<p>
<img src="网络同步.png" alt="Alt text" width="550"/>
</p>

- BIO

<p>
<img src="BIO.png" alt="BIO" width="550"/>
</p>

BIO的阻塞在哪里
1. accept的时候就一直在等待着客户端的连接，这个等待过程中主线程就一直在阻塞
2. 连接建立之后， clone thread的时候会系统调用，在读取到socket信息之前，线程也是一直在等待，一直处于阻塞的状态下

![阻塞IO.png](阻塞IO.png)

当客户端并发访问量增加后，服务端的线程个数和客户端并发访问数呈 1:1 的正比关系，线程数量快速膨胀后，系统的性能将急剧下降

thread池化的问题：如果发生读取数据较慢时（比如数据量大、网络传输慢等），慢的连接会阻塞占用线程，而其他接入的消息，只能一直等待

- NIO
1. JDK new IO
2. Non-blocking

<p>
<img src="NIO.png" alt="BIO" width="350"/>
</p>

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
<p>
<img src="NIO示意.png" alt="Alt text" width="250"/>
</p>

因此引入了多路复用器：select， poll， epoll

多路复用器的两个好处：

1. 该模型能够在同一个线程内同时监听多个IO请求，系统不必创建大量的线程，从而大大减小了系统的开销
2. 等待的方式能减少无效的系统调用，减少了对CPU资源的消耗

<p>
<img src="selector.png" alt="selector" width="350"/>
</p>

linux以及netty

同步阻塞：程序自己读取，调用了方法一直等待有效返回结果（BIO）

同步非阻塞：程序自己读取，调用方法一瞬间，给出是否读到（NIO）

- select

select有fd大小的限制，而poll没有，FD_SETSIZE(1024)
<p>
<img src="select.png" alt="Alt text" width="350"/>
</p>
    
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
3. 读 read 一开始就要注册，但是write依赖以上关系，用的时候注册
4. 如果一开始就注册了write的事件，进入死循环，一直调起

## 2. Netty部分

### Reactor反应器模式

也叫做分发者模式或通知者模式，是一种将就绪事件派发给对应服务处理程序的事件设计模式

![netty架构.png](netty架构.png)

Netty is an **asynchronous event-driven** network application framework
for rapid development of **maintainable high performance protocol** servers & clients.

#### 前置知识

这里以文件输入输出流：FileInputStream、FileOutputStream为例。继承自InputStream和OutputStream，可以看到读写文件的时候，就需要创建两个流对象

<p>
<img src="stream.png" alt="Alt text" width="550"/>
</p>

#### 抽象类描述

InputStream流程如下：

1. read(byte b[]) 直接调用read(byte b[], int off, int len)函数
2. 校验byte数组是否为空
3. 校验读取范围是否正确
4. 校验读取长度
5. 调用read()函数读入一个字节
6.  验证字节是否到达了文件的末尾
7. 将该字节数据保存到b数组中
8. 循环将文件的数据，逐字节的从磁盘中读入放入b字节数组中

```java
// 只以文件流为例
public abstract class InputStream implements Closeable {
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length); // 直接调用read(byte b[], int off, int len)函数
    }
    
    public int read(byte b[], int off, int len) throws IOException {
        if (b == null) { // 校验byte数组是否为空
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) { // 校验读取范围是否正确
            throw new IndexOutOfBoundsException();
        } else if (len == 0) { // 校验读取长度
            return 0;
        }
        // 调用read()函数读入一个字节
        int c = read(); 
        if (c == -1) { // 验证字节是否到达了文件的末尾
            return -1;
        }
        b[off] = (byte)c; // 将该字节数据保存到b数组中
        int i = 1;
        try {
            // 将文件的数据，逐字节的从磁盘中读入放入b字节数组中
            for (; i < len ; i++) {
                c = read();
                if (c == -1) {
                    break;
                }
                b[off + i] = (byte)c;
            }
        } catch (IOException ee) {
        }
        return i;
    }
}
```

FileInputStream流程如下：
1. read(byte b[]) 方法直接调用read(byte b[], int off, int len)方法
2. read(byte b[], int off, int len)方法直接调用readBytes(byte b[], int off, int len)方法
3. 可以看到readBytes方法为native，称之为JNI（Java Native Interface）方法

```java
public class FileInputStream extends InputStream{
    public int read(byte b[]) throws IOException {
        // 直接调用read(byte b[], int off, int len)方法
        return readBytes(b, 0, b.length); 
    }
    
    public int read(byte b[], int off, int len) throws IOException {
        // 直接调用readBytes(byte b[], int off, int len)方法
        return readBytes(b, off, len);
    }
    
    // 可以看到该方法为native，称之为JNI（Java Native Interface）方法
    private native int readBytes(byte b[], int off, int len) throws IOException;
}
```

关键流程如下：
1. malloc分配一片空间
2. 调用read函数读取数据
3. 将数据保存放入堆内存的bytes数组中

下图是readBytes的原理图：
<p>
<img src="readByte.png" alt="readByte" width="650"/>
</p>

```c
jint readBytes(JNIEnv *env, jobject this, jbyteArray bytes,
              jint off, jint len, jfieldID fid){
    ...
    if (len == 0) {
        return 0;
    } else if (len > BUF_SIZE) {
        buf = malloc(len); // 分配一片空间
    }
    ...
    if (fd == -1) {
       ...
    } else {
        nread = IO_Read(fd, buf, len); // 调用read函数读取数据
        if (nread > 0) {
            // 将数据保存放入堆内存的bytes数组中
            (*env)->SetByteArrayRegion(env, bytes, off, nread, (jbyte *)buf);
        }
        ...
    }
    ...
}
```

当然，也可以是BufferedInputStream，调用的是readBytes(byte[] b, int off, int len)

这时，可以引入NIO模型，不需要再为了传输数据创建两个数据流，只需要在两个通讯对象之间，创建channel，将对象抽象为buffer，这时我们只需要双方在该channel
通道传输buffer即可，数据放入buffer

<p>
<img src="nio模型.png" alt="nio模型" width="350"/>
</p>

- 直接内存和堆内存

1. 在JVM内存中分配的空间为：直接内存缓冲区（DirectByteBuffer）
2. 在JVM内存中的堆内存中开辟的空间为：堆内存缓冲区（HeapByteBuffer），操作系统想和JVM沟通，先从堆内存放到DirectByteBuffer，再拷贝到OS内存

![直接内存与堆内存原理.png](直接内存与堆内存原理.png)

零拷贝：指的是在JVM内存这个上下文中，直接从DirectBuffer里面读/写，避免拷贝到堆内内存

```java
// Buffer的基类，其中定义了四个索引下标的用法
public abstract class Buffer {
    // 不变性质: mark <= position <= limit <= capacity
    private int mark = -1; // 标记索引下标
    private int position = 0; // 当前处理索引下标
    private int limit; // 限制索引下标（通常用于在写入后，标记读取的最终位置）
    private int capacity; // 容量索引下标

    // 用于指向DirectByteBuffer的地址
    long address;
}

// 直接继承自Buffer基类，实现了字节缓冲区的基本操作和创建实际Buffer实例的工厂方法
public abstract class ByteBuffer extends Buffer implements Comparable<ByteBuffer>{
    final byte[] hb;                  // 用于HeapByteBuffer中的字节数组

    // 默认分配的为堆内存
    public static ByteBuffer allocate(int capacity) {
        if (capacity < 0)
            throw new IllegalArgumentException();
        return new HeapByteBuffer(capacity, capacity); // 创建堆内存对象
    }

    // 可以通过该方法创建直接内存缓冲区
    public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity); // 创建直接内存缓冲区
    }
}

// 该类继承自ByteBuffer，实现了堆内存中字节数组的实现，该类不是public共有类，只能通过ByteBuffer来创建实例
class HeapByteBuffer extends ByteBuffer{
    HeapByteBuffer(int cap, int lim) {
        super(-1, 0, lim, cap, new byte[cap], 0); // 直接初始化hb字节数组
    }
}

// 该类继承自ByteBuffer，提供了文件fd与用户空间地址的映射
public abstract class MappedByteBuffer extends ByteBuffer{
    private final FileDescriptor fd; // 文件fd
    MappedByteBuffer(int mark, int pos, int lim, int cap,FileDescriptor fd)
    {
        super(mark, pos, lim, cap);
        this.fd = fd;
    }
    
    MappedByteBuffer(int mark, int pos, int lim, int cap) { // 不使用FD映射，支撑DirectByteBuffer
        super(mark, pos, lim, cap);
        this.fd = null;
    }
}

// 该类继承自MappedByteBuffer，提供了直接内存缓冲区的实现
class DirectByteBuffer extends MappedByteBuffer implements DirectBuffer{
    protected static final Unsafe unsafe = Bits.unsafe(); // 操作的Unsafe对象
    DirectByteBuffer(int cap) {                   // package-private
        super(-1, 0, cap, cap); // 初始化父类
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size); // 分配直接内存
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); // 用于清理分配的直接内存
        att = null;
    }
}
```

如下图所示：Buffer等于四个变量+分配在JVM堆内存（DirectByteBuffer）或者堆内存中的数组空间（HeapByteBuffer）
<p>
<img src="NIO Buffer原理图.png" alt="NIO Buffer原理图" width="650"/>
</p>

### 事件循环组
![事件循环组模型.png](事件循环组模型.png)

#### 如何异步执行？Promise

1. Future接口
2. 通过Future接口获取任务执行结果即可

Future弊端：总得调用方来获取再执行步骤，如何解决？

Netty使用了观察者模式，当执行任务完成时，自动在执行线程回调callback;Promise, 加了监听器,和runnable组合即可

<p>
<img src="promise.png" alt="promise" width="450"/>
</p>

默认实现是DefaultPromise，使用方式
```Java
    private final Promise<?> terminationFuture = new DefaultPromise<Void>(GlobalEventExecutor.INSTANCE);
    private final FutureListener<Object> childTerminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            // Inefficient, but works.
            if (isTerminated()) {
            // 触发监听器
                terminationFuture.trySuccess(null);
            }
        }
    };
    .....
    
        private boolean setValue0(Object objResult) {
        if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
                RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {
            if (checkNotifyWaiters()) {
                notifyListeners();
            }
            return true;
        }
        return false;
    }
```

promise的两种用法，一种是直接包装返回
```Java
 final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            // 直接包装返回
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }
```

一种是和runnable组合
```Java
 if (eventLoop.inEventLoop()) {
            register0(promise);
        } else {
            try {
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        register0(promise);
                    }
                });
```

### 事件循环组
事件循环组的线程应该有哪些特性？
1. 负载均衡
2. 周期性调度的工作

事件循环组集成图

![事件循环组.png](事件循环组.png)

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

EventExecutorGroup组里面有多个线程，通过next函数返回**一个(one of the EventExecutor)**，具有调度功能

```Java
public interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor>

/**
     * Returns one of the {@link EventExecutor}s managed by this {@link EventExecutorGroup}.
     */
    EventExecutor next();
```

EventLoopGroup，是一个循环组，它的next()代表了，选择的EventLoop会是一个循环的概念
```Java
/**
 * Special {@link EventExecutorGroup} which allows registering {@link Channel}s that get
 * processed for later selection during the event loop.
 *
 */
public interface EventLoopGroup extends EventExecutorGroup {
    /**
     * Return the next {@link EventLoop} to use
     */
    @Override
    EventLoop next();
    
   ChannelFuture register(Channel channel);
```

EventLoop是一个标志接口，代表了EventLoopGroup的一个EventLoop
```Java
// EventLoop是一个线程，但也是一个特殊的循环组
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup
```

**总结如下：**
EventExecutorGroup继承自ScheduledExecutorService拥有了调度执行的功能，并且通过next()获取一个EventExecutor，EventExecutor是EventExecutorGroup里面专门执行事件的执行器，
是一个特殊的EventExecutorGroup。引入EventLoopGroup，注册channel，而且把线程连起来，形成循环组；EventLoop是一个线程，但也是一个特殊的循环组

MultithreadEventExecutorGroup: 上面继承图漏了AbstractEventExecutorGroup，结合上面图看
```Java
public abstract class MultithreadEventExecutorGroup extends AbstractEventExecutorGroup {
    private final EventExecutor[] children;
    private final Set<EventExecutor> readonlyChildren;
    private final AtomicInteger terminatedChildren = new AtomicInteger();
    private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
    
    // chooser选择children里面的EventExecutor
    private final EventExecutorChooserFactory.EventExecutorChooser chooser;
    
    @Override
    public EventExecutor next() {
        return chooser.next();
    }

// MultithreadEventExecutorGroup的构造方法
// nThreads：创建几个线程
// threadFactory：线程的构造方法
// chooserFactory：选择器

protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    checkPositive(nThreads, "nThreads");

    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args);
            success = true;
        }
```

线程是如何构造的，很简单，直接new一个Thread
```Java
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        this.threadFactory = ObjectUtil.checkNotNull(threadFactory, "threadFactory");
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
```

children数组包含的是EventExecutor的类型，由子类去实现；NioEventLoopGroup的实现里面，children存放的是NioEventLoop
```Java
`@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    SelectorProvider selectorProvider = (SelectorProvider) args[0];
    SelectStrategyFactory selectStrategyFactory = (SelectStrategyFactory) args[1];
    RejectedExecutionHandler rejectedExecutionHandler = (RejectedExecutionHandler) args[2];
    EventLoopTaskQueueFactory taskQueueFactory = null;
    EventLoopTaskQueueFactory tailTaskQueueFactory = null;

    int argsLength = args.length;
    if (argsLength > 3) {
        taskQueueFactory = (EventLoopTaskQueueFactory) args[3];
    }
    if (argsLength > 4) {
        tailTaskQueueFactory = (EventLoopTaskQueueFactory) args[4];
    }
    return new NioEventLoop(this, executor, selectorProvider,
            selectStrategyFactory.newSelectStrategy(),
            rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
}`
```

Chooser的实现: DefaultEventExecutorChooserFactory.INSTANCE
```Java
`public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}`
```
对于2的指数使用PowerOfTwoEventExecutorChooser，否则直接取模操作
```Java
`public EventExecutor next() {
    return executors[idx.getAndIncrement() & executors.length - 1];
}`
```

Netty主方法的解释：
1. channel的参数跟的是母亲的角色，她下面有很多的childHandler跟着SocketChannel的
2. option参数是对NioServerSocketChannel生效的，而childOption是对SocketChannel生效的
3. boss接受请求，是处理ServerSocket连接；而worker执行请求，从连接里面取出来的Socket对象

```Java
public static void main(String[] args) throws Exception {
    // Configure SSL.
    final SslContext sslCtx = ServerUtil.buildSslContext();

    // Configure the server.
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    final EchoServerHandler serverHandler = new EchoServerHandler();
    try {
         // 启动类
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup)
        // channel的参数跟的是母亲的角色，她下面有很多的childHandler跟着的SocketChannel类型
         .channel(NioServerSocketChannel.class)
         .option(ChannelOption.SO_BACKLOG, 100)
         .handler(new LoggingHandler(LogLevel.INFO))
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             public void initChannel(SocketChannel ch) throws Exception {
                 ChannelPipeline p = ch.pipeline();
                 if (sslCtx != null) {
                     p.addLast(sslCtx.newHandler(ch.alloc()));
                 }
                 //p.addLast(new LoggingHandler(LogLevel.INFO));
                 p.addLast(serverHandler);
             }
         });

        // Start the server.
        ChannelFuture f = b.bind(PORT).sync();

        // Wait until the server socket is closed.
        f.channel().closeFuture().sync();
    } finally {
        // Shut down all event loops to terminate all threads.
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}
```

上面注册的pipeline会在哪里执行？

bind方法下：
```Java
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
            
// ServerBootstrap.java
@Override
void init(Channel channel) {
 

    ChannelPipeline p = channel.pipeline();

.....
// todo 这个不是主线程执行的?
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs,
                            extensions));
                }
            });
        }
    });
```
上面只是添加了ChannelInitializer，那真正是在哪里展开的呢？

AbstractChannel.java
```Java
 private void register0(ChannelPromise promise) {
        try {
            // check if the channel is still open as it could be closed in the mean time when the register
            // call was outside of the eventLoop
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }
            boolean firstRegistration = neverRegistered;
            doRegister();
            neverRegistered = false;
            registered = true;


            // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
            // user may already fire events through the pipeline in the ChannelFutureListener.
            pipeline.invokeHandlerAddedIfNeeded();

            safeSetSuccess(promise);
            pipeline.fireChannelRegistered();
```

上面这段代码是观察者？是咋操作的啊


### 内存管理系统

内存池设计

内存分配的问题：
1. 内碎片
![内碎片.png](内碎片.png)
2. 外碎片
![外碎片.png](外碎片.png)




**一致性的问题是由多副本引出的**




