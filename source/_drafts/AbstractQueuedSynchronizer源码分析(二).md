---
title: AbstractQueuedSynchronizer源码分析(二)
date: 2019-12-24 18:00:00
tags: java
category: AQS
---

我们在使用锁时，jdk提供了await，signal等方法用于线程协作。上文分析了lock等方法本质是对AbstractQueuedSynchronizer的方法封装，其实await，signal等方法核心逻辑也在AbstractQueuedSynchronizer中。

await的休眠逻辑类似于获取锁的逻辑，await也维护了一个队列，用于存储await的线程，当被唤醒时，会将线程转移到CLH队列中，竞争获取执行机会。

```java
public final void await() throws InterruptedException {
	if (Thread.interrupted())
		//如果线程被中断，抛出中断异常
		throw new InterruptedException();
	//将当前线程封装成node节点，添加到condition队列队尾
	Node node = addConditionWaiter();
	//释放当前线程占用的锁
	int savedState = fullyRelease(node);
	int interruptMode = 0;
	//判断当前线程是否在CLH队列中
	while (!isOnSyncQueue(node)) {
	    //如果不在队列中，把自己挂起
		LockSupport.park(this);
		//如果中断则跳出循环
		if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
			break;
	}
	//获取锁
	if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
		interruptMode = REINTERRUPT;
	//清理condition队列，删除取消节点
	if (node.nextWaiter != null) // clean up if cancelled
		unlinkCancelledWaiters();
	//判断是否需要抛出中断异常
	if (interruptMode != 0)
		reportInterruptAfterWait(interruptMode);
}
```

<!--more-->

因为调用await方法时，需要获得锁，否则会抛出IllegalMonitorStateException异常，获得锁的线程一定不再CLH队列中，所以可以用`isOnSyncQueue(node)`来判断是否被唤醒，在signal和signalAll方法中，会把await的线程移除condition队列，转移到CLH队列，这个时候只需要取得锁就可以继续执行。

我们看下signal方法的实现:

```java
private void doSignal(Node first) {
	do {
		if ( (firstWaiter = first.nextWaiter) == null)
			lastWaiter = null;
		first.nextWaiter = null;
	} while (!transferForSignal(first) &&
			 (first = firstWaiter) != null);
}
```

可以看到逻辑主要在transferForSignal中，继续进入观察：

```java
final boolean transferForSignal(Node node) {
	/*
	 * If cannot change waitStatus, the node has been cancelled.
	 */
	if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
		return false;

	//如果对此方法还有印象，就会记得这个方法会把node节点添加到CLH队尾
	Node p = enq(node);
	int ws = p.waitStatus;
	//如果前面节点被取消，或则不能设置唤醒后面线程，则直接唤醒线程(否则就会等待CLH的前驱节点唤醒)
	if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
		LockSupport.unpark(node.thread);
	return true;
}
```

SignalAll的方法会转移condition里所有的节点，其他的和Signal一致。
```java
private void doSignalAll(Node first) {
	lastWaiter = firstWaiter = null;
	do {
		Node next = first.nextWaiter;
		first.nextWaiter = null;
		transferForSignal(first);
		first = next;
	} while (first != null);
}
```
await的方法流程图如下：

![await方法流程](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/await%E9%80%BB%E8%BE%91.svg)

如果仔细观察上面流程图，会发现中断后也会触发队列的转移，具体代码如下：

```java
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    /*
     * If we lost out to a signal(), then we can't proceed
     * until it finishes its enq().  Cancelling during an
     * incomplete transfer is both rare and transient, so just
     * spin.
     */
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

上面的代码逻辑就是把node的节点状态修改为0，如果修改失败则认为已有线程调用signal，进行自旋，等待转移完成。