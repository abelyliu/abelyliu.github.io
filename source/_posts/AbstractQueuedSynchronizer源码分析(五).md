---
title: AbstractQueuedSynchronizer源码分析(五)
date: 2019-12-27 18:00:00
tags: java
category: AQS
---

# ReentrantReadWriteLock
前文分析完ReentrantLock后，我们再来分析下ReentrantReadWriteLock，ReentrantReadWriteLock会比ReentrantLock的逻辑复杂一些。

为了更容易理解ReentrantReadWriteLock的源码，这里我们先分析一下ReentrantReadWriteLock的锁降级的用法。下面的例子取自javadoc

```java
 class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();
     if (!cacheValid) {
       // Must release read lock before acquiring write lock
       rwl.readLock().unlock();
       rwl.writeLock().lock();
       try {
         // Recheck state because another thread might have
         // acquired write lock and changed state before we did.
         if (!cacheValid) {
           data = ...
           cacheValid = true;
         }
         // Downgrade by acquiring read lock before releasing write lock
         rwl.readLock().lock();
       } finally {
         rwl.writeLock().unlock(); // Unlock write, still hold read
       }
     }

     try {
       use(data);
     } finally {
       rwl.readLock().unlock();
     }
   }
 }
```

上面的例子可以发现，在获取写锁后，我们还可以获取读锁(无需释放写锁)。关于是否必要锁降级可以参考下面的知乎讨论链接。

<!--more-->

了解了锁降级的用法后，我们还需要了解下ReentrantReadWriteLock的状态表示。我们知道在AQS中，state用来表示锁持有状态，在ReentrantReadWriteLock中，用state同时表示读锁和写锁的状态。
```java
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```
一个int32位bit，高16位表示读锁状态，低16位表示写锁状态，所以会导致读写锁的可重入次数最高只有65535。


## WriteLock

我们先看下WriteLock的实现：

```java
public static class WriteLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -4992448646407690164L;
    private final Sync sync;

    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    public void lock() {
        sync.acquire(1);
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

   //...
}
```

可以看到WriteLock也是直接对AQS的封装。

我们把重点放到应该被覆写的方法
```java
//尝试获取写锁
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    //上面注释描述了三种进入此方法的情况
    //1. 前面有锁(读锁或写锁)且不是当前线程，则直接失败
    //2. 如果计数器饱和(获取锁的线程或重入次数达到最大阈值)，则直接失败
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        //w==0意味着有线程持有读锁，这里无法获取写锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //MAX_COUNT=(1 << SHARED_SHIFT) - 1=65535
        //判断锁是否饱和
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        //能走到这一步的，一定是锁重入逻辑，这里直接获取锁
        setState(c + acquires);
        return true;
    }
    //公平策略和非公平策略都是调用这个方法，区别就在于writerShouldBlock
    //公平策略会判断sync队列里是否有元素，如果有writerShouldBlock返回true，这样就会直接获取失败
    //非公平策略直接返回false，这样当前线程会尝试获取锁，如果获成功，则直接持有锁
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

我们看下写锁的释放逻辑
```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        //当前线程不持有锁，抛出异常
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    //释放锁，修改状态
    setState(nextc);
    return free;
}
```

## ReadLock
上面分析了排它锁，接下来分析下共享锁的逻辑
```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    //1. 其他线程持有写锁，直接失败
    //2. 其他情况，当前线程是有机会获取锁的
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        //如果存在排它锁且不是当前线程，则获取锁失败
        return -1;
    int r = sharedCount(c);
    //同写锁，公平非公平的逻辑封装在readerShouldBlock方法中
    //公平模式下，会判单sync队列是否为空，如果不为空则返回true，则!readerShouldBlock()执行失败
    //非公平模式下，如果发现sync队列的第一个元素是排它锁，则这里不会获取锁，如果获取，
    //在极端情况下(一直有读锁获取)可能导致排它锁饥饿，一直无法获得锁，其他情况会尝试获取共享锁
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        //共享锁在高16位，所以不是自增1
        compareAndSetState(c, c + SHARED_UNIT)) {
        //如果公平锁持有数为0，则将首个获取读锁的线程设置为当前线程
        //下面的if可以暂时先不用管，最后都会返回1，不同的是设置了不同的变量
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            //锁重入后自增
            firstReaderHoldCount++;
        } else {
            //HoldCounter表示最后一个获取锁的线程，这里设置一下其值
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        //获取锁成功
        return 1;
    }
    //完整锁逻辑获取
    return fullTryAcquireShared(current);
}
```

fullTryAcquireShared方法的整体逻辑是和上面方法是类似的

```java
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            //如果是当前线程获取的写锁，则当前线程还是可以获取读锁的
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        //是否超过最大限制，这里无法用负数判断，因为使用了位运算
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //尝试获取锁
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            //下面的if可以暂时先不用管，最后都会返回1，不同的是设置了不同的变量
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

### fullTryAcquireShared为什么要自旋？
看到fullTryAcquireShared的源码，不知道你有没有想过为什么要循环里呢，不是返回获取锁成功或者失败。
这里我们就需要分析下什么情况下会发生自旋。

如果存在排它锁，且不是当前线程，直接返回失败
公平模式下sync队列不为空，非公平模式下，sync队列队首为排它锁，且不是锁重入逻辑，直接返回失败
尝试获取共享锁，获取成功，或者自旋重新判断

可以看到自旋是为了获取锁，而且大概率能获取到锁，这样可以减少加入队列等不必要操作带来的开销

### firstReader，firstReaderHoldCount，cachedHoldCounter，readHolds这些变量作用是什么？
不知道你有没有发现，在获得共享锁时设置了一些变量，感觉不设置也没有什么问题。

我们可以查看下firstReader的依赖关系

![firstReader调用链](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/aqsclh%E7%BB%93%E6%9E%84.svg)

我们发下除了获取锁和释放锁的代码外，还有getReadHoldCount方法依赖这个变量

```java
final int getReadHoldCount() {
    if (getReadLockCount() == 0)
        return 0;

    Thread current = Thread.currentThread();
    if (firstReader == current)
        return firstReaderHoldCount;

    HoldCounter rh = cachedHoldCounter;
    if (rh != null && rh.tid == getThreadId(current))
        return rh.count;

    int count = readHolds.get().count;
    if (count == 0) readHolds.remove();
    return count;

```

可以看到这个方法会返回当前线程持有的读锁重入次数。也就是我们要提供一个重入次数统计的功能，最简单的办法就是用readHolds，把值存储在线程变量里。
而这里jdk为了提高方法的执行效率，会对第一个持有读锁的线程和最后一个持有读锁的线程进行缓存，不从readHolds里获取，估计jdk认为很大概率是单个线程或则最后一个线程会获取锁，这样可以提升速度。


下面看下释放共享锁的过程
```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```
上面释放锁的代码可以注意返回值，如果有两个线程持有共享锁，那么其中一个释放意义不是很大，共享锁还是能获取，排它锁还是获取不了，所以这里的返回值是判断是否完全释放。

# 参考链接：
1. https://www.zhihu.com/question/265909728/answers/updated
