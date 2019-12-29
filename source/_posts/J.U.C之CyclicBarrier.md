---
title: J.U.C之CyclicBarrier
date: 2019-1-3 18:00:00
tags: java
category: J.U.C
---

在介绍AbstractQueuedSynchronizer时，我们曾经介绍过一个子类CountDownLatch，不知道是否对闭锁还有印象。

今天我们介绍的CyclicBarrier有点类似于闭锁，但是比其功能更强。

我们知道CountDownLatch在state变为0后，就不会在限制await的线程。CyclicBarrier可以理解为在state状态为0时，会把所有await线程释放，然后重新将state置为初始化值，这样再有线程进入时，则还需要等待足够的线程await，而且CountDownLatch可以设置一个回调函数，当state为0时，会触发回调函数。

不过在介绍AQS时，我们并没有介绍过CountDownLatch，而没有介绍CyclicBarrier，其实就是CyclicBarrier并没有依赖AQS，所以也并没有什么state变量，接下来我们就看下CyclicBarrier的实现原理。

<!--more-->

```java
//可以看到，构造器还支持一个回调函数
//parties和count都会初始化为参数值
public CyclicBarrier(int parties, Runnable barrierAction) {
	if (parties <= 0) throw new IllegalArgumentException();
	this.parties = parties;
	this.count = parties;
	this.barrierCommand = barrierAction;
}
```

```java
public int await() throws InterruptedException, BrokenBarrierException {
	try {
		//因为await还有个重载方法，需要设置等待时间，那个方法可能出现TimeoutException
		return dowait(false, 0L);
	} catch (TimeoutException toe) {
		throw new Error(toe); // cannot happen
	}
}
```

```java
private int dowait(boolean timed, long nanos)
	throws InterruptedException, BrokenBarrierException,
		   TimeoutException {
	final ReentrantLock lock = this.lock;
	//首先获取重入锁，否则线程直接挂起
	lock.lock();
	try {
		//这是一个屏障状态对象
		final Generation g = generation;
		//如果屏障损坏，则抛出异常
		if (g.broken)
			throw new BrokenBarrierException();
		//如果当前线程被中断，则设置屏障损坏，重置await值并唤醒其他等待的线程
		//然后抛出异常
		if (Thread.interrupted()) {
			breakBarrier();
			throw new InterruptedException();
		}

		//...
		
	} finally {
		lock.unlock();
	}
}
```
从上面我们有几点值得注意
1. 为什么需要锁住，需要保护什么资源？
2. 有哪些情况会抛出BrokenBarrierException？

我们带着疑问继续读下去

```java
//...
//进入线程减1
//当index为0是，意味着加上当前线程，一共已经有了parties次(parties是初始化的数量)
int index = --count;
if (index == 0) {  // tripped
	boolean ranAction = false;
	try {
		final Runnable command = barrierCommand;
		//如果初始化时，设置了回调函数，这里触发回调函数
		//这值得的注意的是，调用的run方法，而不是在线程中调用start方法
		//这意味着使count变为0的线程会执行回调方法
		if (command != null)
			command.run();
		//回调函数执行成功
		ranAction = true;
		//唤醒所有等待的线程
		//重置count为parties
		//重新设置屏障对象
		nextGeneration();
		return 0;
	} finally {
		//如果在回调方法中出现了异常,需要做三件事
		//1. 屏障错误状态设置为true
		//2. 重置count为parties
		//3. 唤醒所有线程
		if (!ranAction)
			breakBarrier();
	}
}
//...
```

上面的代码`nextGeneration();`为什么需要重新设置屏障对象？

```java
// loop until tripped, broken, interrupted, or timed out
for (;;) {
	try {
		//是否设置了超时时间，如果没设置，直接调用Condition的await方法
		if (!timed)
			trip.await();
		else if (nanos > 0L)
			//如果设置了超时时间，则调用超时await
			nanos = trip.awaitNanos(nanos);
	} catch (InterruptedException ie) {
		//如果没有设置屏障错误，则设置，并唤醒其他线程，然后抛出中断异常
		if (g == generation && ! g.broken) {
			breakBarrier();
			throw ie;
		} else {
			//如果屏障错误已被设置，这里只重新标记中断状态
			Thread.currentThread().interrupt();
		}
	}
	//如果屏障错误，直接抛出异常
	if (g.broken)
		throw new BrokenBarrierException();
	//如果不相等，则意味达到了parties，其他线程重置了对象，也解释了上面的疑问
	if (g != generation)
		return index;
	//如果时间到了，还没有等到足够数量，设置屏障错误状态为true，然后唤醒其他线程，然后抛出超时异常
	if (timed && nanos <= 0L) {
		breakBarrier();
		throw new TimeoutException();
	}
}
```

这时候我们就可以回顾下CyclicBarrier设置思路，就是利用锁和condition队列来实现。

我们们知道condition在await时，会释放持有的锁，所以其他的线程才能进来(当前线程还在循环里挂起呢)。


这个时候我们可以看下BrokenBarrierException，设置这个异常设计，假设其中一个线程被中断了，当前线程会抛出InterruptedException，屏障异常状态就会被设置为true，其他线程就会抛出BrokenBarrierException异常。

关于await的很多都在javadoc都有说明，比如调用await方法后，什么时候回返回

1. 最后一个到达
2. 超时
3. 其它线程中断当前线程
4. 其他线程中断其它等待中的线程，当前线程会受波及，如上分析
5. 其他线程超时
6. 其他线程修改了屏障，如调用reset方法

可以看到，虽然看起来CountDownLatch和CyclicBarrier比较类似，但在这些异常等等处理上还是很不一样的。