---
title: AbstractQueuedSynchronizer源码分析(六)
date: 2019-12-28 18:00:00
tags: java
category: AQS
---

前面两篇文章分析了利用AQS是如何实现锁的，接下来我们分析下如果利用AQS构建一些常见的并发工具类，如今天要分析的闭锁CountDownLatch。

不知道你是否有这样的业务场景，让所有线程同时开始执行，当所有线程执行完毕时，主线程可以得到某种通知。

CountDownLatch就是这样一种工具，下面的代码摘自javadoc，模拟了这种场景

```java
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
 } 
```
<!--more-->

上的代码中，在调用`startSignal.countDown(); `前，所有的的work线程都不会执行，都会在startSignal.await()挂起，当主线程执行完`startSignal.countDown();`后，所有的work线程都可以往下执行。

而此时，主线程会在` doneSignal.await(); `这个地方挂起，待所有work线程任务执行完毕后，主线程会继续执行。

接下来，我们就从CountDownLatch的构造函数开始分析

```java
public CountDownLatch(int count) {
    //可以看到，count不能为负数
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

//Sync类直接调用了AQS的setState，初始化了state为count
Sync(int count) {
    setState(count);
}
```

我们可以看到await方法，实质上封装的是AQS的acquireSharedInterruptibly方法
```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

接下来我们看下CountDownLatch是如何覆写父类方法的
```java
//覆写父类方法
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        //如果获取锁失败，加入sync队列
        doAcquireSharedInterruptibly(arg);
}
```

我们可以看到，只有当state等于0时，才会获取锁成功，否者加入sync队列。如果初始state设置大于0，则什么时候state会等于0呢？我们很容易想到countDown方法。

```java
public void countDown() {
    sync.releaseShared(1);
}
```
同样，countDown方法是对AQS中共享锁释放的封装。

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        //唤醒队列里的线程
        doReleaseShared();
        return true;
    }
    return false;
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    //下面就是一个cas自减的操作，而且也没有使用release，直接减一
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            //只有state等于0是，才认为取得共享锁
            return nextc == 0;
    }
}
```

结合上面的代码分析，我们不难发现，CountDownLatch的原理就是设置了一个state，然后当state为0时，sync里所有的线程都被唤醒，而且都可以取得锁。