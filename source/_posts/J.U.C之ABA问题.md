---
title: J.U.C之ABA问题
date: 2020-1-2 18:00:00
tags: java
category: J.U.C
---

## AtomicStampedReference

前面我们利用AtomicInteger等类可以非常简单的实现CAS功能，但使用CAS时，有个问题是无法绕过去的，那就是ABA问题。

如果在我们获取的值，假设为A，在执行CAS操作前，其他线程先设置为B，再设置为A，那么我们的CAS操作还是能执行成功，因为预期值没有发生变化。但是某些情况，这是不允许的，比如维基百科中举了一个例子：

A代表的是一个内存地址，但A里的内容已经发生了改变，地址相同，内容不同，这个时候如果执行CAS变更，会带来一些意想不到的问题。

JDK里面内置了一个AtomicStampedReference类，就是为了解决ABA问题。

```java
//AtomicStampedReference中的静态内部类
private static class Pair<T> {
	final T reference;
	final int stamp;
	private Pair(T reference, int stamp) {
		this.reference = reference;
		this.stamp = stamp;
	}
	static <T> Pair<T> of(T reference, int stamp) {
		return new Pair<T>(reference, stamp);
	}
}
```
我们可以看到，AtomicStampedReference类不仅持有对象引用，还持有一个计数，我们可以在每次cas变更的时候都修改stamp，这样即使reference还是相同，我们可以通过stamp是否发生变化来判断是否有被其他线程修改过。

<!--more-->

我们看起其中一些关键方法

```java
//stampHolder至少长度我1
//会将当前stamp设置到数组中，并返回当前值
public V get(int[] stampHolder) {
	Pair<V> pair = this.pair;
	stampHolder[0] = pair.stamp;
	return pair.reference;
}
//只有当对象引用和stamp都匹配时，才有可能执行cas
//如果要更新的对象和源对象相同，且stamp相同时，不执行cas操作
public boolean compareAndSet(V   expectedReference,
							 V   newReference,
							 int expectedStamp,
							 int newStamp) {
	Pair<V> current = pair;
	return
		expectedReference == current.reference &&
		expectedStamp == current.stamp &&
		((newReference == current.reference &&
		  newStamp == current.stamp) ||
		 casPair(current, Pair.of(newReference, newStamp)));
}
```

## AtomicMarkableReference

网上很多文章都分析AtomicMarkableReference也是用来解决ABA问题，但是我感觉有了AtomicStampedReference，没必要在搞个AtomicMarkableReference，而且用AtomicMarkableReference解决ABA问题来，我总感觉有点问题。

```java
//AtomicMarkableReference静态内部类
//和AtomicStampedReference的区别主要在于一个是int类型的stamp，一个boolean类型的mark
private static class Pair<T> {
	final T reference;
	final boolean mark;
	private Pair(T reference, boolean mark) {
		this.reference = reference;
		this.mark = mark;
	}
	static <T> Pair<T> of(T reference, boolean mark) {
		return new Pair<T>(reference, mark);
	}
}
```

其它的实现类似于AtomicStampedReference，这里就不在介绍。你也可以思考下，用这个解决ABA问题。


```java
private static class Test {
	private AtomicMarkableReference<String> ref =
			new AtomicMarkableReference<>(null, false);

	public void setData(String data) {
		ref.compareAndSet(null, data, false, true);
	}
}
```

上面的例子是，我们需要对data初始化，我们就可以用AtomicMarkableReference，boolean值用来表示是data值是否可用等。即使有多个线程，也只会有一个能初始化data。

参考链接：
1. https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2
2. https://www.logicbig.com/tutorials/core-java-tutorial/java-multi-threading/atomic-markable-reference.html

