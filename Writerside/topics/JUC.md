# JUC -- 3 week

以面试题为主

https://maimai.cn/article/detail?fid=1547702195&efid=YHFWJkmaQFqBh6aKg8auxg

https://blog.csdn.net/weixin_44772566/article/details/136317453

Countdown官方的样例如下
```
 // Sample usage:
 // Here is a pair of classes in which a group of worker threads use two countdown latches:
 // The first is a start signal that prevents any worker from proceeding until the driver is ready for them to proceed;
 // The second is a completion signal that allows the driver to wait until all workers have completed.
 
  class Driver { // ...
    void main() throws InterruptedException {
      CountDownLatch startSignal = new CountDownLatch(1);
      CountDownLatch doneSignal = new CountDownLatch(N);
 
      for (int i = 0; i < N; ++i) // create and start threads
        new Thread(new Worker(startSignal, doneSignal)).start();
 
      doSomethingElse();            // don't let run yet
      startSignal.countDown();      // let all threads proceed
      doSomethingElse();
      doneSignal.await();           // wait for all to finish
    }
  }
 
  class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;
    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
      this.startSignal = startSignal;
      this.doneSignal = doneSignal;
    }
  public void run() {
      try {
        startSignal.await();
        doWork();
        doneSignal.countDown();
      } catch (InterruptedException ex) {} // return;
   }
 
    void doWork() { ... }
```

底层使用的是AQS的共享锁，


CountDownLatch

CountDownLatch是JDK提供的一个同步工具，它可以让一个或多个线程等待，一直等到其他线程中执行完成一组操作

await 不等于0的情况下，放到aqs挂起，等待唤醒

## Java线程与常用线程池体系

![常用线程体系结构.png](../images/常用线程体系结构.png)

### Executor
线程池顶级接口
```Java
public interface Executor {

    /**
     * 根据Exector的实现判断，可能在新线程、线程池、线程调用中执行
     */
    void execute(Runnable command);
}
```

### ExecutorService
线程池次级接口，对Executor做了一些扩展，增加了一些功能

ExecutorService = Executor + Service；等于说是围绕着Executor提供什么样的功能
```Java
public interface ExecutorService extends Executor{

// 关闭线程池，不再接受新任务，但已经提交的任务会执行完成
void shutdown();
/*
 * 立即关闭线程池，尝试停止正在运行的任务，未执行的任务将不再执行
 * 被迫停止及未执行的任务将以列表的形式返回
 */
List<Runnable> shutdownNow();

// 检查线程池是否已关闭
boolean isShutdown();

// 检查线程池是否已终止，只有在shutdown()或shutdownNow()之后调用才有可能为true
boolean isTerminated();

// 在指定时间内线程池达到终止状态了才会返回true
boolean awaitTermination (long timeout, TimeUnit unit) throws InterruptedException;

// 执行有返回值的任务，任务的返回值为task.call()的结果
<T> Future<T> submit ( Callable<T> task);

/*
 * 执行有返回值的任务，任务的返回值为这里传入的result
 * 当然只有当任务执行完成了调用get()时才会返回
 */
<T> Future<T> submit ( Runnable task, T result);

// 批量执行任务，只有当这些任务都完成了这个方法才会返回
<T> List<Future<T>> invokeAll ( Collection<? extends Callable<T>> tasks) throws InterruptedException;

/**
 * 在指定时间内批量执行任务，未执行完成的任务将被取消
 * 这里的timeout是所有任务的总时间，不是单个任务的时间
 */
<T> List<Future<T>> invokeAll ( Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;

// 返回任意一个已完成任务的执行结果，未执行完成的任务将被取消
<T> T invokeAny ( Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;

// 在指定时间内如果有任务已完成，则返回任意一个已完成任务的执行结果，未执行完成的任务将被取消
<T> T invokeAny ( Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
```

### ScheduledExecutorService
对ExecutorService做了一些扩展，增加一些定时任务相关的功能；
### AbstractExecutorService
抽象类，运用模板方法设计模式实现了一部分方法

没有实现具体的execute方法，由子类实现，但是定义了submit方法
```Java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

// 接口组合就是功能级别的整合
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
![future.png](../images/future.png)

Future是Runnable的代理对象，负责对执行体的观察..
<p>
<img src="../images/future mothod.png" alt="Alt text" width="350"/>
</p>

可知RunnableFuture又可以执行，又可以代理功能；具体的实现类是FutureTask

invokeAll方法

模板方法，子类只需要复写execute就行
```Java
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException {
    if (tasks == null)
        throw new NullPointerException();
    ArrayList<Future<T>> futures = new ArrayList<>(tasks.size());
    try {
        for (Callable<T> t : tasks) {
            RunnableFuture<T> f = newTaskFor(t); // FutureTask
            futures.add(f);
            execute(f);
        }
        for (int i = 0, size = futures.size(); i < size; i++) {
            Future<T> f = futures.get(i);
            if (!f.isDone()) {
                try { f.get(); }
                catch (CancellationException | ExecutionException ignore) {}
            }
        }
        return futures;
    } catch (Throwable t) {
        cancelAll(futures);
        throw t;
    }
}
```

invokeAny方法实际调用的是doInvokeAny；
1. 先会去执行第一个task，ecs.submit(it.next()) --> ExecutorCompletionService会调用, 包装的FutureTask，即QueueingFuture(f, completionQueue)
2. 执行完成之后会调用done函数，放入到completionQueue里面
3. 会尝试拿completionQueue队列的结果，如果为空，则执行其他task
4. 没有其他的task，则阻塞ecs.take()
5. 有结果则返回

```Java
public ExecutorCompletionService(Executor executor) {
    if (executor == null)
        throw new NullPointerException();
    this.executor = executor;
    this.aes = (executor instanceof AbstractExecutorService) ?
        (AbstractExecutorService) executor : null;
    this.completionQueue = new LinkedBlockingQueue<Future<V>>();
}

// doInvokeAny中的ecs.submit(it.next())调用
public Future<V> submit(Callable<V> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<V> f = newTaskFor(task);
    executor.execute(new QueueingFuture<V>(f, completionQueue));
    return f;
}

private static class QueueingFuture<V> extends FutureTask<Void> {
    QueueingFuture(RunnableFuture<V> task,
                   BlockingQueue<Future<V>> completionQueue) {
        super(task, null);
        this.task = task;
        this.completionQueue = completionQueue;
    }
    private final Future<V> task;
    private final BlockingQueue<Future<V>> completionQueue;
    
    // hook function，task执行完成之后调用
    protected void done() { completionQueue.add(task); }
}
```

```Java
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        int ntasks = tasks.size();
        if (ntasks == 0)
            throw new IllegalArgumentException();
        ArrayList<Future<T>> futures = new ArrayList<>(ntasks);
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);

        try {
            // Record exceptions so that if we fail to obtain any
            // result, we can throw the last exception we got.
            ExecutionException ee = null;
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            // Start one task for sure; the rest incrementally
            // 先会去执行第一个task
            futures.add(ecs.submit(it.next()));
            --ntasks;
            int active = 1;

            for (;;) {
                Future<T> f = ecs.poll();
                if (f == null) {
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    else if (active == 0)
                        break;
                    else if (timed) {
                        f = ecs.poll(nanos, NANOSECONDS);
                        if (f == null)
                            throw new TimeoutException();
                        nanos = deadline - System.nanoTime();
                    }
                    else
                        f = ecs.take();
                }
                if (f != null) {
                    --active;
                    try {
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            }

            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
            cancelAll(futures);
        }
    }
```

- ThreadPoolExecutor：普通线程池类，包含最基本的一些线程池操作相关的方法实现

设计原理
<p>
<img src="../images/threadpool设计原理.png" alt="Alt text" width="550"/>
</p>

```Java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
// additionally supports operations that wait for the queue to become non-empty when retrieving an element,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
```
![线程池.png](../images/线程池.png)

使用
```Java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 5, 
        0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
```

默认的线程工厂ThreadFactory是什么？

创建的线程不是Daemon
```Java
private static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        // 继承的是ThreadGroup的isDaemon          
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

那默认的拒绝策略是什么？
```Java
private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
```

经典的四个策略：
1. CallerRunsPolicy

   这种策略会将被拒绝的任务直接在调用execute方法的线程中运行。如果线程池已经关闭，则任务会被丢弃。
   这种方式可以减缓新任务的提交速度--**背压**，因为它依赖于任务提交的线程来执行任务

2. AbortPolicy

   这是**默认**的拒绝策略。当任务无法被线程池执行时，会抛出一个RejectedExecutionException异常。
   这种策略适用于必须对任务被拒绝的情况做出响应的场景。

3. DiscardPolicy
   当任务无法被线程池执行时，任务将被直接丢弃，不抛出异常，也不执行任务。
   这种策略适用于那些任务完成不是必须的罕见情况

4. DiscardOldestPolicy

   当任务无法被线程池执行时，线程池会丢弃队列中最旧的任务，然后尝试再次提交当前任务。
   这种策略很少被使用，因为它可能会导致其他线程等待的任务被取消，或者必须记录失败的情况

```Java
// 整形变量的高三位
// 29, 因为线程需要5个状态，3位，2^3
ivate static final int COUNT_BITS = Integer.SIZE - 3;

// 0x1FFFFFFF（29个1）
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 111 0000....000
private static final int RUNNING    = -1 << COUNT_BITS;
// 000 0000....000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
// 010 0000....000
private static final int TIDYING    =  2 << COUNT_BITS;
// 011 0000....000
private static final int TERMINATED =  3 << COUNT_BITS;
```

ThreadPool的五种状态
- RUNNING：接收新任务和进程队列任务
- SHUTDOWN：不接受新任务，但是接收进程队列任务
- STOP：不接受新任务也不接受进程队列任务，并且打断正在进行中的任务
- TIDYING：所有任务终止，待处理任务数量为0，线程转换为TIDYING，将会执行terminated钩子函数
- TERMINATED：terminated()执行完成

```Mermaid
stateDiagram-v2
    [*] --> Running: Initialize
    Running --> Shutdown: shutdown()
    Running --> Stop: shutdownNow()
    Shutdown --> Stop: shutdownNow()
    Shutdown --> Tidying: All tasks are done
    Stop --> Tidying: All tasks cancelled/interrupted
    Tidying --> Terminated: Terminated() hook executed
```

shutdown()
```Java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        // 中断Idle线程
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

shutdownNow()
```Java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        // 中断所有Worker线程
        interruptWorkers();
        // 清空Queue，后续可能要做些补偿操作，所以返回tasks
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

demo源码分析：
```Java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 5, 
        0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
threadPoolExecutor.execute(() -> {});
threadPoolExecutor.shutdown();
threadPoolExecutor.awaitTermination(1, TimeUnit.SECONDS);
```
1. 向ThreadPool提交任务：execute()
2. 创建新线程：addWorker(Runnable firstTask, boolean core)
3. 线程的主循环：Worker.runWorker(Worker w)
4. 从队列中获取排队的任务：getTask()
5. 线程结束：processWorkerExit(Worker w, boolean completedAbruptly)
6. 关闭线程池的方法：shutdown()、shutdownNow()、tryTerminate()

```Java
 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            // 多个线程的情况，ctl已经改了，把值重新获取
            c = ctl.get();
        }
        // 放到工作队列
        if (isRunning(c) && workQueue.offer(command)) {
            // 假如其他线程把线程池关闭，需要移除队列
            int recheck = ctl.get();
            if (!isRunning(recheck) && remove(command))
                reject(command);
            // 临界：allowCoreThreadTimeOut, 允许将Core释放的情况
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 工作队列已经满了
        else if (!addWorker(command, false))
            reject(command);
    }
```

```Java
private boolean addWorker(Runnable firstTask, boolean core) {
    // 判断状态，且对工作线程+1
    retry:
    for (int c = ctl.get();;) {
        // Check if queue empty only if necessary.
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP)
                || firstTask != null
                || workQueue.isEmpty()))
            return false;

        for (;;) {
            // core/非core的情况下，工作线程判断
            if (workerCountOf(c)
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                return false;
            // 工作线程+1；先改状态，有问题再回退
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 多线程导致失败，重新读
            c = ctl.get();  // Re-read ctl
            if (runStateAtLeast(c, SHUTDOWN))
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 多线程保持添加的完整性
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int c = ctl.get();

                if (isRunning(c) ||
                    (runStateLessThan(c, STOP) && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 添加成功启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
    // 启动失败则减少数量，和上文对应起来了
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}

private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        // 减少Worker数量
        decrementWorkerCount();
        // 钩子函数
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

开始执行线程
```Java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 现在开始允许线程中断了
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 如果第一次执行完成task==null的话，从getTask()里面拿
        while (task != null || (task = getTask()) != null) {
            // 执行期间是不允许中断的，所以需要lock
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                try {
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

所以可以衍生出来几个问题：如何去捕获任务执行的异常：
1. 复写afterExecute(task, ex)
2. task.run()；task里面捕获异常


### ScheduledThreadPoolExecutor
定时任务线程池类，用于实现定时任务相关功能；
### ForkJoinPool
新型线程池类，java7中新增的线程池类，基于工作窃取理论实现，运用于大任务拆小任务，任务无限多的场景；
### Executors
线程池工具类，定义了一些快速实现线程池的方法

```Java
ScheduledThreadPoolExecutor复用了ThreadPoolExecutor的构造函数
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```

DelayedWorkQueue是无界还是有界队列，因为无界的话，maximumPoolSize等参数是无用的

**是无界的**
```Java
private void grow() {
    int oldCapacity = queue.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
    if (newCapacity < 0) // overflow
        newCapacity = Integer.MAX_VALUE; // 是无界的
    queue = Arrays.copyOf(queue, newCapacity);
}
```

```Java
public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                       long delay,
                                       TimeUnit unit) {
    if (callable == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<V> t = decorateTask(callable,
        new ScheduledFutureTask<V>(callable,
                                   triggerTime(delay, unit),
                                   sequencer.getAndIncrement()));
    delayedExecute(t);
    return t;
}
```

RunnableScheduledFuture里面代表了单一职责原则，用到了接口隔离和接口组合
```Java
public interface RunnableScheduledFuture<V> extends RunnableFuture<V>, ScheduledFuture<V> {
    // 是否是周期性的任务
    boolean isPeriodic();
}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}

// 组合了Future和Delayed，返回超时时间
public interface ScheduledFuture<V> extends Delayed, Future<V> {
}
```

比较重要的是leader的使用
```Java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 以响应中断的方式进行加锁等待
    lock.lockInterruptibly();
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                // 已经超时
                if (delay <= 0L)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                // 为什么需要leader? 多个线程，一次可能唤醒多个线程；惊群
                if (leader != null)
                // 第二个线程过来直接等待就行
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
       // 没有任务的直接解锁；否则，唤醒
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

```Java
public void run() {
    if (!canRunInCurrentRunState(this))
        cancel(false);
    else if (!isPeriodic())
        super.run();
    // 恢复线程状态，等待下一次执行
    else if (super.runAndReset()) {
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
}

private void setNextRunTime() {
        long p = period;
        if (p > 0)
            time += p;
        else
            time = triggerTime(-p);
    }
```

### CompletableFuture

stage代表了一个异步执行的动作，而动作和动作之间可以用stage关联起来，有触发的先后顺序

<p>
<img src="../images/stage1.png" alt="stage1" width="500"/>
</p>

可以异步或者同步
- 同步执行就是当前执行stage2的线程和stage1是同一个线程
- 异步执行就是当前执行stage2的线程和stage1不是同一个线程

CompletionStage接口：
- 名字 () 同步执行动作；如thenRun
- 名字Async () 异步执行动作，也即放入线程池中执行；如thenRunAsync

<p>
<img src="../images/stage2.png" alt="stage2" width="400"/>
</p>

A CompletableFuture may have dependent completion actions, collected in a linked stack


![completableFuture方法示意.png](../images/completableFuture方法示意.png)

上面的注释，有两段代码看原因
```Java
 CompletableFuture<Void>  completableFuture = CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println();
        });
completableFuture.thenRun(() -> System.out.println(1));
completableFuture.thenRun(() -> System.out.println(2));
completableFuture.thenRun(() -> System.out.println(3));
completableFuture.thenRun(() -> System.out.println(4));
```
output: 4, 3, 2, 1

```Java
CompletableFuture<Void>  completableFuture = CompletableFuture.runAsync(() -> {
   System.out.println();
});
completableFuture.thenRun(() -> System.out.println(1));
completableFuture.thenRun(() -> System.out.println(2));
completableFuture.thenRun(() -> System.out.println(3));
completableFuture.thenRun(() -> System.out.println(4));
```
output: 1, 2, 3, 4

![completablefuture逻辑.png](../images/completablefuture逻辑.png)

**为什么要采取上面这种结构？completion的出现到底是为了什么？**

线程池接纳的必须是runnable future。而completeFuture呢？联合forkJoin使用。forkJoin池只接收forkJoinTask


为什么不直接把runnable放到asyncPool，没有结果，需要包装

```Java
 public static CompletableFuture<Void> runAsync(Runnable runnable) {
     return asyncRunStage(asyncPool, runnable);
 }
```

```Java
 static CompletableFuture<Void> asyncRunStage(Executor e, Runnable f) {
     if (f == null) throw new NullPointerException();
     CompletableFuture<Void> d = new CompletableFuture<Void>();
     e.execute(new AsyncRun(d, f));
     return d;
 }
```
AsyncRun是一个包装对象。执行f，回调d

```Java
volatile Object result; 
volatile Completion stack;

    static final class AsyncRun extends ForkJoinTask<Void>
            implements Runnable, AsynchronousCompletionTask {
        CompletableFuture<Void> dep; Runnable fn;
        AsyncRun(CompletableFuture<Void> dep, Runnable fn) {
            this.dep = dep; this.fn = fn;
        }

        public final Void getRawResult() { return null; }
        public final void setRawResult(Void v) {}
        public final boolean exec() { run(); return true; }

        public void run() {
            CompletableFuture<Void> d; Runnable f;
            if ((d = dep) != null && (f = fn) != null) {
            // help gc
                dep = null; fn = null;
                // 代表d没有被执行过，只能执行一次f
                if (d.result == null) {
                    try {
                        f.run();
                        // 占位
                        d.completeNull();
                    } catch (Throwable ex) {
                        d.completeThrowable(ex);
                    }
                }
                d.postComplete();
            }
        }
    }
```
为什么需要继承ForkJoinTask和Runnable？ 因为可能会把放到不是ForkJoin的线程池

```Java
 `final void postComplete() {
     CompletableFuture<?> f = this; Completion h;
     while ((h = f.stack) != null ||
            (f != this && (h = (f = this).stack) != null)) {
         CompletableFuture<?> d; Completion t;
         if (f.casStack(h, t = h.next)) {
             if (t != null) {
                 if (f != this) {
                     pushStack(h);
                     continue;
                 }
                 h.next = null;    // detach
             }
             f = (d = h.tryFire(NESTED)) == null ? this : d;
         }
     }
 }`
```


thenRun方法
```Java
public CompletableFuture<Void> thenRun(Runnable action) {
     return uniRunStage(null, action);
 }
 
// e是线程池，f就是线程执行体
private CompletableFuture<Void> uniRunStage(Executor e, Runnable f) {
    if (f == null) throw new NullPointerException();
    
    // 新的stage
    CompletableFuture<Void> d = new CompletableFuture<Void>();
    // 如果e不等于空，也即线程池不为空，那么表明需要异步执行，这时包装UniRun中
    // d是新的CompletableFuture, this是d依赖的上个stage
    if (e != null || !d.uniRun(this, f, null)) {
        UniRun<T> c = new UniRun<T>(e, d, this, f);
        // 如果上一个stage没有完成，则将其压入当前CompletableFuture Completion栈中
        this.push(c);
        // 由于压入可能失败，这是由于当前CompletableFuture已经执行完成了，那么需要补救一下
        c.tryFire(SYNC);
    }
    return d;
}

 final boolean uniRun(CompletableFuture<?> a, Runnable f, UniRun<?> c) {
     Object r; Throwable x;
     // (r = a.result) == null : 看下上个stage有没有完成
     if (a == null || (r = a.result) == null || f == null)
         return false;
     // 判断当前的stage的result是不是等于null
     if (result == null) {
         if (r instanceof AltResult && (x = ((AltResult)r).ex) != null)
             completeThrowable(x, r);
         else
             try {
                 if (c != null && !c.claim())
                     return false;
                 // 直接执行f，完成当前stage
                 f.run();
                 completeNull();
             } catch (Throwable ex) {
                 completeThrowable(ex);
             }
     }
     return true;
 }
```

```Java
final void push(UniCompletion<?, ?> c) {
    if (c != null) {
        // 由于循环CAS压入Completion栈中的条件必须为"当前stage结果为null,也即未完成状态"
        while (result == null && !tryPushStack(c))
            lazySetNext(c, null); // clear on failure
    }
}
```

```Java
final boolean uniRun(CompletableFuture<?> a, Runnable f, UniRun<?> c) {
    Object r; Throwable x;
    // 如果a未完成，那么返回false，由外部压入依赖Completion栈中
    if (a == null || (r = a.result) == null || f == null)
        return false;
    // 到这里，那么a已经完成。当前stage未完成，也即保证只调用一次
    if (result == null) {
        // 如果依赖的a stage出现执行异常，那么completeThrowable
        if (r instanceof AltResult && (x = ((AltResult)r).ex) != null)
            completeThrowable(x, r);
        else {
            // 正常完成
            try {
                if (c != null && !c.claim())
                    return false;
                // 直接执行f，然后完成当前stage
                f.run();
                completeNull();
            } catch (Throwable ex) {
                completeThrowable(ex);
            }
        }
    }
    return true;
}
```

上述代码是异步执行不保证顺序的原因所在

接下去分析下 
```Java
static final class UniRun<T> extends UniCompletion<T,Void> {
```

Completion模板类

由于completion需要放入普通线程池和forkJoin线程池都必须兼容。所以我们必须让它继承自ForkJoinTask，并且也实现runnable
```Java
abstract static class Completion extends ForkJoinTask<Void>
   implements Runnable, AsynchronousCompletionTask {
    volatile Completion next;  // 指向栈中下一个Completion

    // 执行动作并返回所需要传播执行完成的stage，SYNC同步执行，ASYNC异步执行，NESTED嵌套执行
    abstract CompletableFuture<?> tryFire(int mode);
    
    // 
    abstract boolean isLive();

    // 兼容普通线程池执行
    public final void run() { tryFire(ASYNC); }

    // 兼容ForkJoinPool执行
    public final boolean exec() { tryFire(ASYNC); return true; }

    public final Void getRawResult() { return null; }
    public final void setRawResult(Void v) {}
}
```

UniCompletion模板类

Uni指的是单个
```Java
abstract static class UniCompletion<T, V> extends Completion {
    Executor executor;       // 执行使用的线程池
    CompletableFuture<V> dep; // 依赖完成的stage
    CompletableFuture<T> src; // 动作源

    UniCompletion(Executor executor, CompletableFuture<V> dep,
                   CompletableFuture<T> src) {
        this.executor = executor; this.dep = dep; this.src = src;
    }

    // 判断当前completion是否可以被执行
    final boolean claim() {
        Executor e = executor;
        // 通过ForkJoinTask的Tag标记位从0->1，这时将当前completion放入线程池中执行并返回true
        if (compareAndSetForkJoinTaskTag((short) 0, (short) 1)) {
            if (e == null)
                return true;
            executor = null; // 解除线程池的引用 帮助GC
            e.execute(this);
        }
        return false;
    }
}
```

```Java
 static final class UniRun<T> extends UniCompletion<T,Void> {
     Runnable fn;
     UniRun(Executor executor, CompletableFuture<Void> dep,
            CompletableFuture<T> src, Runnable fn) {
         super(executor, dep, src); this.fn = fn;
     }
     
     // 依赖的stage完成之后回调
     final CompletableFuture<Void> tryFire(int mode) {
         CompletableFuture<Void> d; CompletableFuture<T> a;
         if ((d = dep) == null ||
             !d.uniRun(a = src, fn, mode > 0 ? null : this))
             return null;
         dep = null; src = null; fn = null;
         return d.postFire(a, mode);
     }
 }

```

**阶段总结：**

第一句是异步执行的，并返回一个 CompletableFuture。第二句当调用thenRun的时候，会生成一个UniRun的Completion对象，并将其压入到CompletableFuture的执行栈中。
此时，当前的动作还没有执行完成，Completion对象会被压入到CompletableFuture的执行队列中，随后其他的Completion对象也会依次压入。
当这个方法执行完成之后，也就是asyncRun方法执行完成之后，会触发postComplete回调。postComplete会遍历已经压入的Completion对象，并回调它们的tryFire方法。
这样，这些Completion对象就可以被执行了。这相当于各个阶段（stage）之间的依赖是用什么来绑定在一起的呢？是使用stage加上Completion对象来实现的
```Java
CompletableFuture<Void>  completableFuture = CompletableFuture.runAsync(() -> {
   System.out.println();
});
completableFuture.thenRun(() -> System.out.println(1));
```


### AQS

![juc.png](../images/juc.png)

### 并发容器

![](../images/blockingQueue.png)
