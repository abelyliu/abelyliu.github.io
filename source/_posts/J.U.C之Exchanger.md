---
title: J.U.C之Exchanger
date: 2020-1-6 18:00:00
tags: java
category: J.U.C
---

如果编写多线程代码，会遇到如何进行线程通讯的问题，在JAVA并发包里就提供了一个工具类Exchanger用于交换线程间数据。我们看一个简单的例子：

```java
public static void main(String[] args) {
	Exchanger<String> exchanger = new Exchanger<>();
	new Thread(new Task1(exchanger)).start();
	new Thread(new Task2(exchanger)).start();
}

public static class Task1 implements Runnable {
	Exchanger<String> exchanger;

	public Task1(Exchanger<String> exchanger) {
		this.exchanger = exchanger;
	}

	@Override
	public void run() {
		try {
			String hello = exchanger.exchange("hello");
			System.out.println("hello is " + hello);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

public static class Task2 implements Runnable {
	Exchanger<String> exchanger;

	public Task2(Exchanger<String> exchanger) {
		this.exchanger = exchanger;
	}

	@Override
	public void run() {
		try {
			String world = exchanger.exchange("world");
			System.out.println("world is " + world);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
//output
world is hello
hello is world
```

可以看到A线程打印了B线程的值，B线程打印了A线程的值。

我们今天就来分享下Exchanger是如何实现的。

![单槽交换](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202001023.svg)

![多槽交换](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202001024.svg)


Exchanger的基本思路是共享内存，如果槽位内没有数据则放入数据，等待其他线程来交换，其它线程来了之后，发现槽位不为空，则把其他线程值取出，并设置自己的值在里面，唤醒其它线程。

在线程比较多的情况下，只有一个槽位大大影响交换的效率，这样就会升级为多槽位，也可以看出哪两个线程会交换也是随机的。

我们从exchange方法看起

```java
//可以看到逻辑基本分为三步
//1. 如果参数为null，设置为空，避免传递null
//2. 获取其他线程值，如果中断，则抛出中断异常
//3. 返回其他线程的值
public V exchange(V x) throws InterruptedException {
	Object v;
	Object item = (x == null) ? NULL_ITEM : x; // translate null args
	if ((arena != null ||
		 (v = slotExchange(item, false, 0L)) == null) &&
		((Thread.interrupted() || // disambiguates null return
		  (v = arenaExchange(item, false, 0L)) == null)))
		throw new InterruptedException();
	return (v == NULL_ITEM) ? null : (V)v;
}
```

可以看到主要逻辑全在if中，if由两个条件&&而成
1. `(arena != null || (v = slotExchange(item, false, 0L)) == null)`
2. `((Thread.interrupted() || (v = arenaExchange(item, false, 0L)) == null))`

看到其实主要通过`v = slotExchange(item, false, 0L)`和`v = arenaExchange(item, false, 0L)`来获取其它线程的值。

```java
private final Object slotExchange(Object item, boolean timed, long ns) {
	Node p = participant.get();
	Thread t = Thread.currentThread();
	if (t.isInterrupted()) // preserve interrupt status so caller can recheck
		return null;

	for (Node q;;) {
		if ((q = slot) != null) {
			if (U.compareAndSwapObject(this, SLOT, q, null)) {
				//获取到其它线程设置的值，此时已经取到q的引用，把槽清空，供给其它线程使用
				Object v = q.item;
				//将当前线程参数挂到match字段上
				q.match = item;
				Thread w = q.parked;
				//唤醒对应线程
				if (w != null)
					U.unpark(w);
				return v;
			}
			// 执行到这里，表明出现了竞争，修改为多槽模式
			if (NCPU > 1 && bound == 0 &&
				U.compareAndSwapInt(this, BOUND, 0, SEQ))
				arena = new Node[(FULL + 2) << ASHIFT];
		}
		//说明发生了扩容，返回，通过多槽模式获取
		else if (arena != null)
			return null; // caller must reroute to arenaExchange
		else {
			//此时槽位为空，尝试将当前线程值放到槽位上
			p.item = item;
			if (U.compareAndSwapObject(this, SLOT, null, p))
				break;
			p.item = null;
		}
	}

	// 执行到此处说明已经将当前线程的值放到槽位上了
	int h = p.hash;
	long end = timed ? System.nanoTime() + ns : 0L;
	int spins = (NCPU > 1) ? SPINS : 1;
	Object v;
	while ((v = p.match) == null) {
		//如果cpu数量大于1，则自旋一段时间
		//如果只有一个cpu，当前线程不释放cpu，其他线程是没有机会来交换数据的，很多JDK代码自旋的逻辑都是基于多cpu
		if (spins > 0) {
			h ^= h << 1; h ^= h >>> 3; h ^= h << 10;
			if (h == 0)
				h = SPINS | (int)t.getId();
			else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
				Thread.yield();
		}
		//执行到此处说明其他线程正在交换值，代码还没完全执行完毕，这里重置自旋次数，自旋等待
		else if (slot != p)
			spins = SPINS;
		//如果当前没有中断，没有超时，挂起当前线程
		//这里可以思考下 arena == null有什么含义
		else if (!t.isInterrupted() && arena == null &&
				 (!timed || (ns = end - System.nanoTime()) > 0L)) {
			U.putObject(t, BLOCKER, this);
			p.parked = t;
			if (slot == p)
				U.park(false, ns);
			p.parked = null;
			U.putObject(t, BLOCKER, null);
		}
		//如果超时或者中断，则退出循环
		else if (U.compareAndSwapObject(this, SLOT, p, null)) {
			v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null;
			break;
		}
	}
	//清理线程变量
	U.putOrderedObject(p, MATCH, null);
	p.item = null;
	p.hash = h;
	return v;
}
```

可以看到单槽获取的代码还是很简单的。这里我们思考几个小问题：

arena的数组长度`arena = new Node[(FULL + 2) << ASHIFT];`，这个长度是多少呢？

```java
//NCPU代表当前cpu个数
//MMASK << 1等于510
//NCPU >>> 1等于NCPU/2
 static final int FULL = (NCPU >= (MMASK << 1)) ? MMASK : NCPU >>> 1;
//假设我们是8核处理器，Full=4，(FULL + 2) << ASHIFT，ASHIFT等于7，也就是768
arena = new Node[(FULL + 2) << ASHIFT];
```

第二个问题就是自旋次数的问题

```java
//自旋次数初始化为2的10次方
private static final int SPINS = 1 << 10;

//只要spins大于0就会自旋，可以看到代码里有个--spins，spins的值会越来越小
//h的作用又是什么呢，首先是避免空循环被JIT优化，还有个作用是引入随机
//h < 0 时才会执行--spins，一般来说会循环SPINS*2次，h<0的概率是0.5
//什么时候会执行Thread.yield();呢？一般来说是spins=SPINS/2和spins=0时，会尝试让出cpu
if (spins > 0) {
	h ^= h << 1; h ^= h >>> 3; h ^= h << 10;
	if (h == 0)
		h = SPINS | (int)t.getId();
	else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
		Thread.yield();
}
```

第三个问题在等待时，为什么需要加`arena == null`判断

我们考虑这样一种情况，当线程执行到`arena == null`前发生了中断，有两个线程进入，其中一个失败，修改为多槽模式，这个时候当前线程自旋就可以了，就不该进行休眠，这里也只是减少冲突带来的影响，并不能完全避免。


接下来我们分析下多槽位的代码

```java
private final Object arenaExchange(Object item, boolean timed, long ns) {
	Node[] a = arena;
	Node p = participant.get();
	for (int i = p.index;;) {                      // access slot at i
		int b, m, c; long j;                       // j is raw array offset
		//q为数组第一个元素
		Node q = (Node)U.getObjectVolatile(a, j = (i << ASHIFT) + ABASE);
		//如果q不为null，则当前线程可以尝试和q线程交换数据
		if (q != null && U.compareAndSwapObject(a, j, q, null)) {
			Object v = q.item;                     // release
			q.match = item;
			Thread w = q.parked;
			if (w != null)
				U.unpark(w);
			return v;
		}
		//如果q为空，当前线程尝试放入
		else if (i <= (m = (b = bound) & MMASK) && q == null) {
			p.item = item;                         // offer
			//交换成功，等待交换
			if (U.compareAndSwapObject(a, j, null, p)) {
				long end = (timed && m == 0) ? System.nanoTime() + ns : 0L;
				Thread t = Thread.currentThread(); // wait
				for (int h = p.hash, spins = SPINS;;) {
					Object v = p.match;
					//如果v不为空，则意味着其它线程已经交换了值，当前线程取出即可
					if (v != null) {
						U.putOrderedObject(p, MATCH, null);
						p.item = null;             // clear for next use
						p.hash = h;
						return v;
					}
					//自旋，和单槽相同，这里还有注释o(╥﹏╥)o
					else if (spins > 0) {
						h ^= h << 1; h ^= h >>> 3; h ^= h << 10; // xorshift
						if (h == 0)                // initialize hash
							h = SPINS | (int)t.getId();
						else if (h < 0 &&          // approx 50% true
								 (--spins & ((SPINS >>> 1) - 1)) == 0)
							Thread.yield();        // two yields per wait
					}
					//这里也是如果不为p则意味着其它线程正在交换
					else if (U.getObjectVolatile(a, j) != p)
						spins = SPINS;       // releaser hasn't set match yet
					//挂起当前线程，等待交换	，这里注意只有在m==0是才挂起线程，待会解释m等变量的作用
					else if (!t.isInterrupted() && m == 0 &&
							 (!timed ||
							  (ns = end - System.nanoTime()) > 0L)) {
						U.putObject(t, BLOCKER, this); // emulate LockSupport
						p.parked = t;              // minimize window
						if (U.getObjectVolatile(a, j) == p)
							U.park(false, ns);
						p.parked = null;
						U.putObject(t, BLOCKER, null);
					}
					//超时或者中断返回，或者自旋完毕还是等不到交换需要处理
					else if (U.getObjectVolatile(a, j) == p &&
							 U.compareAndSwapObject(a, j, p, null)) {
						//前移一位
						if (m != 0)                // try to shrink
							U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1);
						p.item = null;
						p.hash = h;
						i = p.index >>>= 1;        // descend
						if (Thread.interrupted())
							return null;
						if (timed && m == 0 && ns <= 0L)
							return TIMED_OUT;
						//重新去竞争
						break;                     // expired; restart
					}
				}
			}
			else
				//放入失败，删除p的改变
				p.item = null;                     // clear offer
		}
		else {
			if (p.bound != b) {                    // stale; reset
				p.bound = b;
				p.collides = 0;
				i = (i != m || m == 0) ? m : m - 1;
			}
			else if ((c = p.collides) < m || m == FULL ||
					 !U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)) {
				p.collides = c + 1;
				i = (i == 0) ? m : i - 1;          // cyclically traverse
			}
			else
				i = m + 1;                         // grow
			p.index = i;
		}
	}
}
```

默认所有的线程第一次进入都会选择数组的第一个元素，当线程发现无法设置值，也无法读取值(有两个线程在使用)，他就会去数组的第二个里执行相同的逻辑(实际上是数组最后一个有元素的地方，这里便于理解)。

接下来我们从代码分析下是如何实现上述功能的

全局bound变量代表着最后一个存在元素的地方
局部m变量代表代表着最后一个存在元素的下标，从bound计算得来

我们先来分析m和b这两个变量，首先观察下这两个变量是怎么修改的

`m = (b = bound) & MMASK`  
`MMASK=0xff=1111 1111`，bound初始为SEQ，在slotExchange方法中设置的，说明m初始值也为0

首先看下b的修改
```java
//SEQ=256=1 0000 0000
U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1)
!U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)
```

而m只有一处赋值，结合b的修改地方，可以得出`U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)`执行后，m会加1，`U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1)`执行后，m会减1。

```java
//所有线程如果是第一次执行到此处，则p.bound != b一定true,m=0,i=0
//初始化线程的bound和collides
if (p.bound != b) {                    // stale; reset
	p.bound = b;
	p.collides = 0;
	//如果数组的第一个槽位被占用，线程无法设置也无法获取，数组的第二个槽位被设置，等待线程交换，
	//这时候线程会执行到此处，m=1,i=0，这里就会将i修改为1，然后重新循环，和第二个槽位线程交换，如果有n个槽位有值，这里i=m=n，直接跳转到第n个槽位
	//m-1会被执行是前面恰好有槽位释放，这里不再解释
	i = (i != m || m == 0) ? m : m - 1;
}
```
![enter description here](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202001025.svg)

假设为上图所示，当再有个线程进来时，就会执行`if (p.bound != b)`的代码，`p.collides = 0;i=m=3`，他还是无法获取3号槽位的元素(数组第四个)，则会执行下面代码

```java
//首先p.collides会自增，i会自减，当c>=m时，会执行U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)，再去扩容，如果成功i=m+1，失败则多轮询一次
//简单来说，越到后面扩容越难，需要挨个尝试前面的槽位，p.collides机制
else if ((c = p.collides) < m || m == FULL ||
		 !U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)) {
	p.collides = c + 1;
	i = (i == 0) ? m : i - 1;          // cyclically traverse
} else
	i = m + 1;
```

读到这里如果你仔细想就会发现一个问题，如果我申请个新的槽位，如果没有线程来交换不就尴尬了吗

还记得上面挂起线程的if语句为什么有个m == 0吗，保证只有在0号位置的槽位可以挂起，其他的只能自旋，最后一个else if语句中`U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1);`是否还有印象，当自旋一定次数后，就会向前移动一位，重新去选择和竞争槽位。

