---
title: AbstractQueuedSynchronizer源码分析(一)
date: 2019-12-23 18:00:00
tags: java
category: java
---

AbstractQueuedSynchronizer是jdk内实现很多并发工具的基础类，如下

![AbstractQueuedSynchronizer子类](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/Snipaste_2019-12-23_14-27-04.webp)

AbstractQueuedSynchronizer用来实现锁，信号量等常见并发控制工具，今天就来分析下AQS的代码实现。

在分析代码前，先简单介绍下实现原理，这里以独占锁(ReentrantLock)为业务场景。

我们在代码里获取锁时，如果能获取到，则立即返回，否则会等到锁可用时得到锁。

AQS内部维护了一个链表，每一个没有获取锁的线程，会被封装成一个一个node节点，添加到链表的尾部，然后会将自己休眠挂起，结构如下图所示：

![clh链表](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/aqsclh%E7%BB%93%E6%9E%84.svg)

<!--more-->
1. head node是个空节点，并不关联任何线程，只用来关联node节点
2. 每次锁释放时，只有head node的指向的node节点会被唤起，这里可以先简单理解为，node节点会顺序获取锁

如果查看ReentrantLock的lock方法，最终会发现指向AbstractQueuedSynchronizer的acquire方法(这里以ReentrantLock的公平模式为例，非公平差别并不大)
```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```

1. `tryAcquire(arg) `尝试获取锁，如果获取成功，则直接返回(此方法需要子类实现自己的业务,此处不做分析)。
2. `addWaiter(Node.EXCLUSIVE)`将当前线程封装成node节点，并插入CLH链表。
3.  `acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`判断当前节点是否是首节点，如果不是，则挂起
4.  `selfInterrupt()` 如果挂起过程中发生了中断，这里设置下中断标志位。


addWaiter方法比较简单，这里看下acquireQueued方法

acquireQueued的方法主要分为两个部分：
```java
//获取node节点的前驱节点
final Node p = node.predecessor();
//如果当前节点是首节点，则尝试获取锁，如果获取到锁，返回在等待锁的过程中是否发生过线程中断
if (p == head && tryAcquire(arg)) {
    //修改头节点，设置为刚刚获取锁的节点
	setHead(node);
	p.next = null; // help GC
	failed = false;
	return interrupted;
}
```

```java
//是否需要挂起当前线程，如果需要，调用parkAndCheckInterrupt方法将当前线程挂起
 if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
	interrupted = true;
```

完整代码如下：
```java
final boolean acquireQueued(final Node node, int arg) {
	boolean failed = true;
	try {
		boolean interrupted = false;
		//进入for循环后，只有获取锁或则执行代码出现了异常，否则不会退出循环
		for (;;) {
			final Node p = node.predecessor();
			if (p == head && tryAcquire(arg)) {
				setHead(node);
				p.next = null; // help GC
				failed = false;
				return interrupted;
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
	    //如果获取锁过程中，出现异常，则将节点移除队列
		if (failed)
			cancelAcquire(node);
	}
}
```

这里我们再看下，什么时候需要将自己挂起呢？
```java
//每个node里会维护一个状态waitStatus，如果waitStatus为SIGNAL，则代表当前node获取到锁后，需要将后序节点唤醒，去竞争锁
//这里会判断前驱节点是否为SIGNAL，如果是SIGNAL，则可以进入睡眠，如果不是则设置为SIGNAL，然后再去尝试锁获取，没有获取到会再进入此方法，然后会判断需要睡眠。这里还会把前驱节点状态为删除的从CLH队列里移除
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	int ws = pred.waitStatus;
	if (ws == Node.SIGNAL)
	    //如果已经设置为唤醒，则认为当前线程可以休眠
		return true;
	if (ws > 0) {
	    //如果前驱状态为已删除，则修改前驱为一个非删除的节点
		do {
			node.prev = pred = pred.prev;
		} while (pred.waitStatus > 0);
		pred.next = node;
	} else {
	    //设置当前节点前驱会唤醒当前线程
		compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
	}
	//再去尝试获取锁
	return false;
}
```

现在看下锁的请求流程
![锁请求流程](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/CLH%E6%B5%81%E7%A8%8B.svg)

我们可以看到，最后绝大部分的线程都会挂起，那么什么时候会被唤醒呢？毫无疑问，这个需要观察下释放锁的代码：

```java
//如果释放锁成功，如果队列不为空，则唤醒队列的第一个元素
public final boolean release(int arg) {
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h);
		return true;
	}
	return false;
}
```

可以看到，逻辑主要再unparkSuccessor方法中，主要方法如下：

```java
Node s = node.next;
//如果next节点为空，或则取消，则从tail(尾节点)向前查询，找到一个非取消节点
if (s == null || s.waitStatus > 0) {
	s = null;
	for (Node t = tail; t != null && t != node; t = t.prev)
		if (t.waitStatus <= 0)
			s = t;
}
//唤醒s线程
if (s != null)
	LockSupport.unpark(s.thread);
```

当s.thread线程被唤醒时，就会从上面睡眠的地方被唤醒，继续去尝试获取锁。





