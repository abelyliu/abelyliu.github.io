---
title: J.U.C之ArrayBlockingQueue(一)
date: 2020-1-4 18:00:00
tags: java
category: J.U.C
---

ArrayBlockingQueue是一个比价常用的并发集合，一般被我们用来实现生产者消费者模式。

我们先来看下各个方法的效果。

方法|	抛出异常|	返回特殊值|	一直阻塞|	超时退出
:---:|:---:|:---:|:---:|:---:|
插入|	add(e)|	offer(e)|	put(e)|	offer(e, time, unit)
移除|	remove()|	poll()|	take()|	poll(time, unit)
检查|	element()|	peek()|	不可用|	不可用

```java
final Object[] items;
int takeIndex;
int putIndex;
int count;
final ReentrantLock lock;
private final Condition notEmpty;
private final Condition notFull;
transient Itrs itrs = null;
```
可以看到ArrayBlockingQueue的底层主要基于ReentrantLock锁，及对应的Condition来实现。

<!--more-->

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
	if (capacity <= 0)
		throw new IllegalArgumentException();
	this.items = new Object[capacity];
	lock = new ReentrantLock(fair);
	notEmpty = lock.newCondition();
	notFull =  lock.newCondition();
}
```

这里的fair就是设置对应的ReentrantLock是公平还是非公平模式。

## 添加方法

```java
public void put(E e) throws InterruptedException {
	//不能放入null元素
	checkNotNull(e);
	final ReentrantLock lock = this.lock;
	//可能抛出中断异常
	lock.lockInterruptibly();
	try {
		//如果已经达到数组的上线，需要阻塞当前线程
		//调用await方法，释放锁，在数组未满的时候得到通知唤醒
		while (count == items.length)
			notFull.await();
		//如果数组还有位置，则添加到数组中
		enqueue(e);
	} finally {
		lock.unlock();
	}
}
```

```java
public boolean offer(E e, long timeout, TimeUnit unit)
	throws InterruptedException {
	//不能放入null元素
	checkNotNull(e);
	//计算等待时长
	long nanos = unit.toNanos(timeout);
	final ReentrantLock lock = this.lock;
	//可能抛出中断异常
	lock.lockInterruptibly();
	try {
		//如果已经达到数组的上线，需要等待timeout时间
		//如果在此段时间内数组不为空，则有机会插入元素
		while (count == items.length) {
			//如果时间到了，则插入失败
			if (nanos <= 0)
				return false;
			//休眠，在数组没满的时候被唤醒，或则时间到了
			nanos = notFull.awaitNanos(nanos);
		}
		//说明数组没有满，加入元素
		enqueue(e);
		return true;
	} finally {
		lock.unlock();
	}
}
```

```java
public boolean offer(E e) {
	checkNotNull(e);
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		//如果数组满了，则直接返回false
		if (count == items.length)
			return false;
		else {
			//未满，则加入元素
			enqueue(e);
			return true;
		}
	} finally {
		lock.unlock();
	}
}
```

```java
//此方法在AbstractQueue中，ArrayBlockingQueue直接调用了父类
public boolean add(E e) {
	//如果加入成功则返回true，否则抛出异常
	if (offer(e))
		return true;
	else
		throw new IllegalStateException("Queue full");
}
```

## 移除方法

```java
public E poll() {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		//如果数组为空，则返回null，否则返回一个元素
		return (count == 0) ? null : dequeue();
	} finally {
		lock.unlock();
	}
}
```

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
	long nanos = unit.toNanos(timeout);
	final ReentrantLock lock = this.lock;
	lock.lockInterruptibly();
	try {
		//如果当前数组数量为0，则尝试等待一段时间
		while (count == 0) {
			//如果时间到了，还是为0，返回null
			if (nanos <= 0)
				return null;
			//休眠，并在数组不为空的时候被唤醒，或则时间到达
			nanos = notEmpty.awaitNanos(nanos);
		}
		//返回一个元素
		return dequeue();
	} finally {
		lock.unlock();
	}
}
```

```java
public E take() throws InterruptedException {
	final ReentrantLock lock = this.lock;
	lock.lockInterruptibly();
	try {
		//如果数组没有元素，则挂起，等待非空通知
		while (count == 0)
			notEmpty.await();
		//返回元素
		return dequeue();
	} finally {
		lock.unlock();
	}
}
```

```java
public boolean remove(Object o) {
	//因为不允许加入null元素，所以不可能删除null元素
	if (o == null) return false;
	final Object[] items = this.items;
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		if (count > 0) {
			final int putIndex = this.putIndex;
			int i = takeIndex;
			do {
				//如果元素相同，则删除
				//可以看到从takeIndex开始查找，找到第一个就结束，不会遍历整个数组
				if (o.equals(items[i])) {
					removeAt(i);
					return true;
				}
				//如果达到数组边界，则移动到头结点，相当于一个环形队列
				if (++i == items.length)
					i = 0;
			} while (i != putIndex);
		}
		//如果数量等于0，则删除必然失败
		return false;
	} finally {
		lock.unlock();
	}
}
```

```java
void removeAt(final int removeIndex) {
	// assert lock.getHoldCount() == 1;
	// assert items[removeIndex] != null;
	// assert removeIndex >= 0 && removeIndex < items.length;
	final Object[] items = this.items;
	//如果删除的元素刚好是下个出队元素，则出队
	if (removeIndex == takeIndex) {
		// removing front item; just advance
		items[takeIndex] = null;
		if (++takeIndex == items.length)
			takeIndex = 0;
		count--;
		if (itrs != null)
			itrs.elementDequeued();
	} else {
		//如果出队的元素在中间，则需要将数组removeIndex+1到putIndex之间的元素向前移动一位
		final int putIndex = this.putIndex;
		for (int i = removeIndex;;) {
			int next = i + 1;
			if (next == items.length)
				next = 0;
			if (next != putIndex) {
				items[i] = items[next];
				i = next;
			} else {
				items[i] = null;
				this.putIndex = i;
				break;
			}
		}
		count--;
		if (itrs != null)
			itrs.removedAt(removeIndex);
	}
	notFull.signal();
}
```

## 检查方法
```java
public E peek() {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		//返回数组即将取出的元素，如果队列为空，则返回null
		return itemAt(takeIndex); // null when queue is empty
	} finally {
		lock.unlock();
	}
}
```

```java
//此方法在父类AbstractQueue中
public E element() {
	E x = peek();
	if (x != null)
		return x;
	else
		//如果为空，直接抛出异常
		throw new NoSuchElementException();
}
```

## 内部方法分析
### enqueue
入队方法本质上都是调用enqueue方法
```java
private void enqueue(E x) {
	// assert lock.getHoldCount() == 1;
	// assert items[putIndex] == null;
	final Object[] items = this.items;
	//将元素x放到putindex位置上
	items[putIndex] = x;
	//如果下个要放的达到数组长度，则回到数组开头
	if (++putIndex == items.length)
		putIndex = 0;
	//数组中元素数量自增1
	count++;
	//发出数组非空信号
	notEmpty.signal();
}
```
### dequeue
出队方法本质上都是dequeue方法
```java
private E dequeue() {
	// assert lock.getHoldCount() == 1;
	// assert items[takeIndex] != null;
	final Object[] items = this.items;
	@SuppressWarnings("unchecked")
	//获取出队元素，然后将原位置置空
	E x = (E) items[takeIndex];
	items[takeIndex] = null;
	//将takeIndex移到下一个位置，如果达到数组上限，回到数组开头
	if (++takeIndex == items.length)
		takeIndex = 0;
	//元素数量自减
	count--;
	//如果有迭代，则修改迭代信息
	if (itrs != null)
		itrs.elementDequeued();
	//发出数组未满信号
	notFull.signal();
	return x;
}
```

可以看到ArrayBlockingQueue几个方法的源码还是比较简单的，就是一个循环数组和condition队列。




## 参考链接
1. https://blog.csdn.net/csdn_xpw/article/details/78400225
