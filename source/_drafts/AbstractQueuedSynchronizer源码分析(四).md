---
title: AbstractQueuedSynchronizer源码分析(四)
date: 2019-12-26 18:00:00
tags: java
category: AQS
---

前面我们分析AbstractQueuedSynchronizer源码时，虽然以锁为业务场景进行分析，但实际上并没有涉及其一些定制代码，今天我们就来看看AQS的子类是如何实现各自的业务场景。

# ReentrantLock
我们知道ReentrantLock分为公平模式和非公平模式，而且默认为非公平模式

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

![ReentrantLock内部结构](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/Snipaste_2019-12-25_10-15-21.png)

我们可以看到，无论是`FairSync`还是`NonfairSync`，都是Sync的子类，Sync是ReentrantLock的抽象静态内部类，这个也是AbstractQueuedSynchronizer推荐的方式，推荐用静态内部类继承AQS。

<!--more-->

## FairSync

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    //获取锁
    final void lock() {
        //AQS的acquire，不知道是否还有印象
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        //获取当前状态，0代表没有线程持有锁，大于0代表有线程持有锁
        int c = getState();
        if (c == 0) {
            //这个时候虽然有锁，但是因为是公平模式，所以还需要判断sync队列里是否有节点
            //如果对列为空，或则第一个节点是当前线程则获取锁
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //判断持有锁的线程是否是当前线程，用于锁的重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                //当重入次数大于int最大值时，抛出错误
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        //获取锁失败
        return false;
    }
}
```

我们可以看到FairSync的lock方法直接调用了AQS的acquire方法，主要逻辑在于覆写的tryAcquire方法，用于定义在什么情况下能获取锁。

## NonfairSync

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    final void lock() {
        //如果能将state状态从0修改成1，则意味着获取了锁，这里可以直接将锁分配给请求线程
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            //调用AQS的acquire方法
            acquire(1);
    }
    
    //这里调用了Sync的nonfairTryAcquire方法
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

从目前看来，公平锁和非公平锁最大的区别在于，公平锁即使能获取锁，也要判断他前面是否有先请求的线程，而非公平锁直接获得锁。

不知道你这里会不会有个疑问，为什么把NonfairSync的tryAcquire的逻辑封装到父类Sync中呢？

## Sync

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    //上面的注释明确的表明了，为什么要将逻辑封装到父类，无论你使用的是公平锁还是非公平锁
    //在调用trylock方法时，一定是非公平的，即使你是公平锁
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //这里和公平锁的区别在于不用判断队列
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    
    //释放锁代码
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            //当前线程不持有锁无法释放
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        
        //释放锁
        setState(c);
        return free;
    }
    //......
}
```

如果你了解锁的接口，那么会发现还有好多的接口没有实现，上面只有lock的方法的实现，接口的其他方法也都是直接调用AQS的方法的
```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

举个例子,sync直接调用父类AQS的acquireInterruptibly方法
```java
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

所以理解了AQS的各个方法，也就理解ReentrantLock大半了。


