---
title: AbstractQueuedSynchronizer源码分析(七)
date: 2019-12-29 18:00:00
tags: java
category: AQS
---

上文分析了闭锁CountDownLatch的实现，今天来分享另一个比较常用的协作工具类Semaphore信号量。

不知道你是否有看过这样一个面试题，两个线程轮流执行，如线程A输出1，线程B输出2，怎么才能输出12121212这样的序列。

其实通过Semaphore就可以很简单的做到上面的功能。

```java
public static void main(String[] args) {
    Semaphore semaphore1 = new Semaphore(0);
    Semaphore semaphore2 = new Semaphore(0);
    Task1 task1 = new Task1(semaphore1, semaphore2);
    Task2 task2 = new Task2(semaphore1, semaphore2);
    new Thread(task1).start();
    new Thread(task2).start();
    semaphore1.release();
}

public static class Task1 implements Runnable {
    Semaphore semaphore1;
    Semaphore semaphore2;


    public Task1(Semaphore semaphore1, Semaphore semaphore2) {
        this.semaphore1 = semaphore1;
        this.semaphore2 = semaphore2;
    }

    @Override
    public void run() {
        try {
            while (true) {
                semaphore1.acquire();
                System.out.print("1");
                semaphore2.release();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public static class Task2 implements Runnable {
    Semaphore semaphore1;
    Semaphore semaphore2;


    public Task2(Semaphore semaphore1, Semaphore semaphore2) {
        this.semaphore1 = semaphore1;
        this.semaphore2 = semaphore2;
    }

    @Override
    public void run() {
        try {
            while (true) {
                semaphore2.acquire();
                System.out.print("2");
                semaphore1.release();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

<!--more-->

如果你对Semaphore信号量不太熟悉，可能不太明白上面是如何实现的，我们今天来分析下Semaphore是如何利用AQS实现的。

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

NonfairSync(int permits) {
    super(permits);
}

Sync(int permits) {
    setState(permits);
}
```

我们可以看到`new Semaphore(0)`就是初始化了state的值，这点FairSync和NonfairSync都是一样的。

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

我们看到acquire方法也是直接封装了AQS的acquireSharedInterruptibly的方法，我们分别来看下公平和非公平是如何获取锁的

```java
//FairSync实现
protected int tryAcquireShared(int acquires) {
    for (;;) {
        //如果sync不为空，则获取锁失败
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        //如果remaining<0获取失败
        //compareAndSetState(available, remaining)修改成功，则获取锁成功
        //如果cas修改失败，则自旋重试
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
//NonfairSync实现
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        //和上面的区别在于不会判断sync队列是否为空
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

接下来看下release方法

```java
public void release() {
    sync.releaseShared(1);
}
//FairSync和NonfairSync都使用此方法
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        //增加许可，如果许可达到最大值则溢出
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        //cas增加许可
        if (compareAndSetState(current, next))
            return true;
    }
}
```

我们可以看到，release方法基本是一定能增加成功，逻辑就是将state+1。

现在回想下Semaphore的逻辑，在调用acquire时，如果state大于0则成功，在调用release时，只是简单的将state+1。
不知道你有没有想过Semaphore可以用来干嘛，没错，Semaphore可以用来实现限流功能，像spring cloud gateway，Hystrix等等，都支持Semaphore实现限流功能。

如果感觉理解的还不错，可以试试https://leetcode-cn.com/problems/print-in-order/这道简单的打印题。