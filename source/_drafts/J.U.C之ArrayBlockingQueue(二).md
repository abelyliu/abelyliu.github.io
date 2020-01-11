---
title: J.U.C之ArrayBlockingQueue(二)
date: 2020-1-5 18:00:00
tags: java
category: J.U.C
---

![ArrayBlockingQueue类结构](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/arrayblockingqueueclass.png)

上文我们讨论了ArrayBlockingQueue几个常见代码方法的实现，其实代码分析时避开了一些代码的解释，就是迭代器相关的，如果你观察上面ArrayBlockingQueue的类图，你会发现他也实现了迭代器模式，支持并发修改的情况下迭代元素。虽然不经常使用此方法，我们还是可以学习一下，了解在并发情况下，提供迭代器的复杂性。


```java
//出自dequeue出队方法
if (itrs != null)
	itrs.elementDequeued();
```

为什么需要在出队时要处理修改器呢，今天就来了解下ArrayBlockingQueue的迭代器设计思路。

<!--more-->

## 迭代器使用示例

```java
public static void main(String[] args) throws InterruptedException {
	ArrayBlockingQueue queue = new ArrayBlockingQueue(3);
	//直接将队列放满
	queue.put("1");
	queue.put("2");
	queue.put("3");

	//获取三个迭代器
	Iterator iterator1 = queue.iterator();
	Iterator iterator2 = queue.iterator();
	Iterator iterator3 = queue.iterator();
	
	//遍历迭代器1
	while (iterator1.hasNext()) {
		System.out.print(iterator1.next() + " ");
	}

	System.out.println("");
	//出队两个元素，此时还有3在队列里
	queue.take();
	queue.take();
	while (iterator2.hasNext()) {
		System.out.print(iterator2.next() + " ");
	}

	System.out.println("");
	//继续出队一个元素，并新增一个元素，此时队列里只有4元素
	//尝试考虑交换下面两行代码，先put，再take，看看有什么区别？
	queue.take();
	queue.put("4");
	while (iterator3.hasNext()) {
		System.out.print(iterator3.next() + " ");
	}
}
```

你可以把自己的思考的结果写出来，可以在修改看看会输出什么样的结果。

上面的输出依次是
```tex
1 2 3
1 3
1
//如果交换了take和put，则结果为1 4
```

是不是会感到很奇怪，尤其是交换take和put，明明数组里的元素都是一样的，迭代器的输出完全不同。

## 并发迭代的问题

![问题示意图](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/ArrayBlockingQueue%E8%BF%AD%E4%BB%A3%E5%99%A8.svg)

我们可以思考下上面图里的三种情况，假使获取迭代器时状态为第一行，遍历迭代器时，当时的数组结构为第二三四行，第四行是出队入队一圈后的结果。更一般的问题在于，我们在迭代到一半时数组内容发生了变动。

我们可以带着疑问，阅读下Itrs的源码。

## Itrs

### 结构
```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

	//...	
	
	transient Itrs itrs = null;
	
	//...
}
//可以看到Itrs由node链表组成，node里面封装了Itr，也就是iterator
//简单来说Itrs就是iterator的链表
class Itrs {
	 private Node head;

	private class Node extends WeakReference<Itr> {
			Node next;

			Node(Itr iterator, Node next) {
				super(iterator);
				this.next = next;
			}
		}
	//...	
}

//实现了Iterator接口
private class Itr implements Iterator<E> {
	//...
}
```

itrs实际上是当前数组的迭代器，我们知道ArrayBlockingQueue实现了Collection接口，也就可以使用迭代器模式进行遍历，而在多线程的并发修改下，获取迭代器后，集合里的内容是可能发生改变的，比如出队又入队一圈。

在ArrayBlockingQueue中Itrs只保持弱一致，不会抛出ConcurrentModificationException，并且确保可遍历迭代器构造时存在的元素，此外还可能（但并不保证）反映构造后的所有修改。而且从迭代器的源码可以看出，它只能感知在它创建时有效元素位置上的变化（删除、被替换），而不能感知新增的元素。

接下来我们就分析下它的迭代器是如何实现的，首先从获取开始·

```java
public Iterator<E> iterator() {
	return new Itr();
}
```

构造器
```java
Itr() {
	// assert lock.getHoldCount() == 0;
	lastRet = NONE;
	final ReentrantLock lock = ArrayBlockingQueue.this.lock;
	lock.lock();
	try {
		if (count == 0) {
			//如果队列里没有数组，设置为DETACHED模式
			cursor = NONE;
			nextIndex = NONE;
			prevTakeIndex = DETACHED;
		} else {
			//获取当前take index并存储，因为是内部类所以这种写法
			final int takeIndex = ArrayBlockingQueue.this.takeIndex;
			prevTakeIndex = takeIndex;
			//获取taken index对应的元素并存储，所以迭代器一定会输出当时获取迭代器存在的第一个元素
			nextItem = itemAt(nextIndex = takeIndex);
			//获取下一个元素索引，如果到了put index(存储边界)， 则返回-1
			cursor = incCursor(takeIndex);
			//将当前迭代器挂载到itrs
			if (itrs == null) {
				itrs = new Itrs(this);
			} else {
				itrs.register(this); // in this order
				//清理无用的迭代器
				itrs.doSomeSweeping(false);
			}
			prevCycles = itrs.cycles;
		}
	} finally {
		lock.unlock();
	}
}
```

看到这里，我们首先会有个印象，知道为什么一开的示例，为什么永远会输出1。

接下来我们看下hasNext和next方法

```java
public boolean hasNext() {
	// assert lock.getHoldCount() == 0;
	//只要不是在获取迭代器是队列为空，这里nextItem就不会为空
	if (nextItem != null)
		return true;
	//如果没有下个元素，说明迭代器迭代完毕，需要删除
	noNext();
	return false;
}
```

```java
public E next() {
	//获取nextItem
	final E x = nextItem;
	if (x == null)
		throw new NoSuchElementException();
	final ReentrantLock lock = ArrayBlockingQueue.this.lock;
	lock.lock();
	//设置下个nextItem，供下个hasNext方法使用
	try {
		if (!isDetached())
			//需要判断队列有没有改变，如果改变，需要更新迭代器
			incorporateDequeues();
		//这里修改了lastRet值为nextIndex
		lastRet = nextIndex;
		//cursor是下个元素的下标，可以参考初始化的值
		final int cursor = this.cursor;
		if (cursor >= 0) {
			//设置nextItem值，供下次调用hasNext和next使用
			nextItem = itemAt(nextIndex = cursor);
			//更新cursor值，供下次使用
			this.cursor = incCursor(cursor);
		} else {
			//上面incCursor方法如果没有下个元素了，会将cursor设置为NONE，从而进入此方法
			nextIndex = NONE;
			nextItem = null;
		}
	} finally {
		lock.unlock();
	}
	return x;
}
```

如果你阅读了noNext和next方法，比较复杂的逻辑在于`incorporateDequeues();`中，也就是更新迭代器的逻辑，我们着重分析一下

```java
private void incorporateDequeues() {
	final int cycles = itrs.cycles;
	final int takeIndex = ArrayBlockingQueue.this.takeIndex;
	final int prevCycles = this.prevCycles;
	final int prevTakeIndex = this.prevTakeIndex;
	//cycles != prevCycles代表着从获取迭代器到现在，takeIndex一定回到过0，回过得次数就是cycles-prevCycles
	//takeIndex!=prevTakeIndex代表着有元素出队过
	//可以思考下为什么不能交换两个条件
	if (cycles != prevCycles || takeIndex != prevTakeIndex) {
		final int len = items.length;
		//计算获取迭代器到现在一共出队多少次
		long dequeues = (cycles - prevCycles) * len
			+ (takeIndex - prevTakeIndex);

		//检查这些值得有效性，后面分析怎么进行检查
		if (invalidated(lastRet, prevTakeIndex, dequeues, len))
			lastRet = REMOVED;
		if (invalidated(nextIndex, prevTakeIndex, dequeues, len))
			nextIndex = REMOVED;
		if (invalidated(cursor, prevTakeIndex, dequeues, len))
			cursor = takeIndex;
		//如果无效，则标记状态
		if (cursor < 0 && nextIndex < 0 && lastRet < 0)
			detach();
		else {
			//否则，重置prevCycles和prevTakeIndex的值
			this.prevCycles = cycles;
			this.prevTakeIndex = takeIndex;
		}
	}
}
```

我们接下来分析加如何检查状态

```java
//构造器里初始化lastRet=NONE(-1)
if (invalidated(lastRet, prevTakeIndex, dequeues, len))
	lastRet = REMOVED;

private boolean invalidated(int index, int prevTakeIndex,
							long dequeues, int length) {
	//第一次进入index<0，直接返回false
	//上面说过，构造器里初始化-1
	if (index < 0)
		return false;
	//如果你还记得上面next方法中这样一行lastRet = nextIndex;
	//nextIndex代表着被迭代器输出的下个值的下标
	//也就是第一个调用next后，lastRet就不为0了
	int distance = index - prevTakeIndex;
	if (distance < 0)
		distance += length;
	//这里在判断出队长度是否大于输出数量
	return dequeues > distance;
}
```

这里解释起来比较麻烦，我们先来看个迭代器使用示例

```java
ArrayBlockingQueue queue = new ArrayBlockingQueue(10);
queue.put("1");
queue.put("2");
queue.put("3");
queue.put("4");
queue.put("5");
queue.put("6");
int i = 1;
Iterator iterator1 = queue.iterator();
while (iterator1.hasNext()) {
	System.out.print(iterator1.next() + " ");
	if (i++ == 1) {
		queue.poll();//1出队
		queue.poll();//2出队
		queue.poll();//3出队
		queue.poll();//4出队
		queue.poll();//5出队
	}
}
````

上面程序输出的结果是1 2 6，问题在于2为什么也会被输出？

![检查示意图](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202001021.svg)

上图是描述上面这种场景的数组结构示意图，可以看到lastRet是此次迭代输出的下标，那么理解lastRet就理解迭代器是如循环的。

1. 首先lastRest在Itr的构造器被初始化`lastRet = NONE;`
2. 在迭代next方法里`lastRet = nextIndex;`
3.  这个时候我们需要理解下nextIndex这个变量，lastRest在Itr的构造器被初始化`nextItem = itemAt(nextIndex = takeIndex);`，可以看到nextIndex的初始值就是获取迭代器时的takeIndex
4.  在迭代next方法里`nextItem = itemAt(nextIndex = cursor);`，可以看到nextIndex被赋值为cursor
5.  同样，我们看下cursor在Itr构造器里初始化代码`cursor = incCursor(takeIndex);`，incCursor可以理解为自增1，因为是个数组环，所以需要做一些特殊处理
6.  next方法中`this.cursor = incCursor(cursor);`，同样是自增1

结合上面的图应该可以理解lastRet+1=nextIndex，nextIndex+1=cursor

理解了上面的变量关系，我们再回头看下invalidated方法的代码

```java
if (invalidated(lastRet, prevTakeIndex, dequeues, len))
	lastRet = REMOVED;

//结合上图和代码理解，第二次进入迭代器时
//dequeues=5(出队5个元素)
//lastRet=index=1，prevTakeIndex=上次获取迭代器时的takeIndex=0
//distance=1-0=1,dequeues>distance 返回true
//这里distance<0是为了处理环的情况
//后面两个校检逻辑是一样的，只不过index换成了nextIndex,cursor
private boolean invalidated(int index, int prevTakeIndex,
							long dequeues, int length) {
	if (index < 0)
		return false;
	int distance = index - prevTakeIndex;
	if (distance < 0)
		distance += length;
	return dequeues > distance;
}
```

```java
//可以看到，这里返回true后会订正cursor的值
if (invalidated(cursor, prevTakeIndex, dequeues, len))
	cursor = takeIndex;

//在next方法中，会设置nextItem为最新值，这样下次迭代就能跳过被删除的元素
nextItem = itemAt(nextIndex = cursor);
```

我们继续分析
```java
//如果三个都小于0，说明迭代器已经失效，标记
//nextIndex和lastRet小于0比较好理解，什么时候cursor会小于0
if (cursor < 0 && nextIndex < 0 && lastRet < 0)
	detach();
else {
	//如果迭代器还正常，重置当前的prevCycles和prevTakeIndex，用来判断迭代过程中是否发生变化
	this.prevCycles = cycles;
	this.prevTakeIndex = takeIndex;
}
```
那么什么时候cursor < 0呢？

```java
//next方法
this.cursor = incCursor(cursor);
//可以看到index==putIndex会变成-1
private int incCursor(int index) {
	if (++index == items.length)
		index = 0;
	if (index == putIndex)
		index = NONE;
	return index;
}
```
这个时候仔细想下`if (cursor < 0 && nextIndex < 0 && lastRet < 0)`会为true。

```java
ArrayBlockingQueue queue = new ArrayBlockingQueue(10);
queue.put("1");
queue.put("2");
queue.put("3");
queue.put("4");
queue.put("5");
queue.put("6");
int i = 1;
Iterator iterator1 = queue.iterator();
while (iterator1.hasNext()) {
	System.out.print(iterator1.next() + " ");
	if (i++ == 5) {
		queue.put("7");
		queue.put("8");
		queue.put("9");
		queue.put("10");
		queue.poll();
		queue.poll();
		queue.poll();
		queue.poll();
		queue.poll();
		queue.poll();
		queue.poll();
		queue.poll();
	}
}
```

在第五次迭代时，cursor已经被设置为-1，这个时候数组发生了特定的变动就会进入`if (cursor < 0 && nextIndex < 0 && lastRet < 0)`，在这种情况下依然将迭代器标记为迭代完成。

标记完成迭代器的代码简单看下

```java
//prevTakeIndex一开始为获取迭代器时的takeIndex下标，如果下标为-1则表明已完成，我们着重看下doSomeSweeping方法
private void detach() {
	if (prevTakeIndex >= 0) {
		prevTakeIndex = DETACHED;
		itrs.doSomeSweeping(true);
	}
}
```

```java
void doSomeSweeping(boolean tryHarder) {
	//参数tryHarder只是控制下面的循环次数，一个4次一个16次
	int probes = tryHarder ? LONG_SWEEP_PROBES : SHORT_SWEEP_PROBES;
	Node o, p;
	final Node sweeper = this.sweeper;
	boolean passedGo;   // to limit search to one full sweep
	//如果sweeper不为空，则上次没有遍历完，继续上次的遍历
	if (sweeper == null) {
		o = null;
		p = head;
		passedGo = true;
	} else {
		o = sweeper;
		p = o.next;
		passedGo = false;
	}

	for (; probes > 0; probes--) {
		if (p == null) {
			//如果未true代表迭代器为空，不需要处理
			if (passedGo)
				break;
			//否则从head开始查找	
			o = null;
			p = head;
			passedGo = true;
		}
		final Itr it = p.get();
		final Node next = p.next;
		//如果当前迭代器为空或者已完成，则移除链表
		if (it == null || it.isDetached()) {
			// found a discarded/exhausted iterator
			probes = LONG_SWEEP_PROBES; // "try harder"
			// unlink p
			p.clear();
			p.next = null;
			if (o == null) {
				head = next;
				if (next == null) {
					// We've run out of iterators to track; retire
					itrs = null;
					return;
				}
			}
			else
				o.next = next;
		} else {
			//处理下一个节点
			o = p;
		}
		p = next;
	}
	//如果循环到达还没有处理完，则记录用于下次继续循环
	this.sweeper = (p == null) ? null : o;
}
```

这个时候我们在回头看下dequeue中的`itrs.elementDequeued();`

```java
//分为两种情况，如果数组无元素或出队游标循环一圈
void elementDequeued() {
	if (count == 0)
		queueIsEmpty();
	else if (takeIndex == 0)
		takeIndexWrapped();
}
void queueIsEmpty() {
	//清空所有的迭代器
	for (Node p = head; p != null; p = p.next) {
		Itr it = p.get();
		if (it != null) {
			p.clear();
			//修改cursor，lastRet等值，终止迭代器的循环
			//shutdown不会修改nextItem的值，保证hasNext为true时，一定能获取到值
			it.shutdown();
		}
	}
	head = null;
	itrs = null;
}

void takeIndexWrapped() {
	//循环次数+1
	cycles++;
	for (Node o = null, p = head; p != null;) {
		final Itr it = p.get();
		final Node next = p.next;
		//和上面区别在于it.takeIndexWrapped()方法
		if (it == null || it.takeIndexWrapped()) {
			p.clear();
			p.next = null;
			if (o == null)
				head = next;
			else
				o.next = next;
		} else {
			o = p;
		}
		p = next;
	}
	if (head == null)   // no more iterators to track
		itrs = null;
}

//可以看到如果迭代器的prevCycles和当期的cycles不一样，则停止迭代
boolean takeIndexWrapped() {
	if (isDetached())
		return true;
	if (itrs.cycles - prevCycles > 1) {
		shutdown();
		return true;
	}
	return false;
}
```

最后还有个删除方法也会触发Itrs的removedAt方法，removedAt的主要逻辑在于，如果删除的元素下标和lastRet，nextIndex，cursor比较，如果cursor等比其大，需要自减1，前移一位。

![移除方法示意图](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202001022.svg)

如果删除的是数组第一个元素，则lastRet,nextIndex,cursor都向前移一位，代码这里就不贴了。