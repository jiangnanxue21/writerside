# JUC

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