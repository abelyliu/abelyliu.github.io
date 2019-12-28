---
title: J.U.C之AtomicInteger
date: 2019-12-31 18:00:00
tags: java
category: J.U.C
---

其实AtomicInteger只是对Unsafe中CAS模块的封装，不过其中还是有几个值得注意的方法。我们首先看下AtomicInteger的api

```java
//初始化默认值为initialValue
public AtomicInteger(int initialValue) {
    value = initialValue;
}
//初始化默认值为0
public AtomicInteger() {
}
//返回当前值
public final int get() {
    return value;
}
//设置新值
public final void set(int newValue) {
    value = newValue;
}
//设置最终值
public final void lazySet(int newValue) {
    unsafe.putOrderedInt(this, valueOffset, newValue);
}
//设置新值并返回老值
public final int getAndSet(int newValue) {
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}
//当老值为expect时，设置为update，如果设置失败，则返回false
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
//当老值为expect时，设置为update，如果设置失败，则返回false，但是有可能非正常失败
public final boolean weakCompareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
//自增1，并返回老值
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
//自减1，并返回老值
public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}
//自增delta，并返回老值
public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
}
//自增1，并返回自增后的值
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
//自减1，并返回自减后的值
public final int decrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}
//自增delta，并返回自增后的值
public final int addAndGet(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}
//提供一个通过老值计算新值的函数，并把老值修改为新值，方法返回老值
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return prev;
}
//提供一个通过老值计算新值的函数，并把老值修改为新值，方法返回新值
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return next;
}
//提供一个老值和参数x计算新值的方法，并把老值修改为新值，方法返回老值
public final int getAndAccumulate(int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return prev;
}
//提供一个老值和参数x计算新值的方法，并把老值修改为新值，方法返回新值
public final int accumulateAndGet(int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return next;
}
```

如果你仔细观察方法，你会发现`lazySet`和`weakCompareAndSet`这两个方法比较奇怪。

<!--more-->

### lazySet

AtomicInteger已经有了set方法，为什么还需要lazySet方法呢？

>lazySet语义是保证写操作不会与任何先前的写操作重新排序，而是可以与后续操作重新排序（或等效地，可能对其他线程不可见），直到发生其他一些volatile写操作或同步操作为止。


![Java内存模型](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/1775037-20191223140827610-1136016478.png)

简单来说，就是用来保证最终一致性的，直接将结果写入主存，其他线程可能一段时间内看到的还是老值。为什么提供这样一个方法呢？lazySet的速度会比set方法快(少了个内存屏障)，一个典型的场景是，如果一个对象已经失效，我们可以将其设置为null(这里解释下，对象是volatile的，直接设置null会比这种方式开销要大)，这样利于加速GC(需要保证其他线程在一定时间内读取老值没有问题)。

### weakCompareAndSet
如果你仔细阅读AtomicInteger的源码，还会发现一个奇怪的事情，compareAndSet和weakCompareAndSet的实现完全一样，这其中又是怎么一回事呢？
```java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
public final boolean weakCompareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
如果你观察weakCompareAndSet方法javadoc，会有这样一段话`May fail spuriously and does not provide ordering guarantees, so is only rarely an appropriate alternative to compareAndSet`。可以看到这个方法有可能意外的失败，并不提供顺序保证。

这里解释下，在一些平台上，CAS是单指令，如x86平台，这样的平台没有compareAndSet和weakCompareAndSet的区分。但是存在着一些其他平台，如ARM，PowerPC等，CAS指令是由LL/SC两条汇编指令实现的，那么就可能出现在LL 与 SC 两条指令在执行的间期发了上下文切换，或者其他加载和存储操作，这都将导致一个store-conditional的spuriously fail。

可能你现在思考是，这两个代码实现都是一样的，怎么可能会有不一样的行为呢？如你所料，其实weakCompareAndSet的实际行为和compareAndSet是完全相同的，最起码在java6~java8都是一致的。我们看下java9的代码：

```java
public final boolean compareAndSet(int expectedValue, int newValue) {
    return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
}
@Deprecated(since="9")
public final boolean weakCompareAndSet(int expectedValue, int newValue) {
    return U.weakCompareAndSetIntPlain(this, VALUE, expectedValue, newValue);
}
public final boolean weakCompareAndSetPlain(int expectedValue, int newValue) {
    return U.weakCompareAndSetIntPlain(this, VALUE, expectedValue, newValue);
}
```

我们可以看到weakCompareAndSet已经被标记为过时的方法，而内部改造成了调用weakCompareAndSetPlain方法，我们继续跟踪代码

```java
@HotSpotIntrinsicCandidate
public final native boolean compareAndSetInt(Object o, long offset,
                                             int expected,
                                             int x);
@HotSpotIntrinsicCandidate
public final boolean weakCompareAndSetIntPlain(Object o, long offset,
                                               int expected,
                                               int x) {
    return compareAndSetInt(o, offset, expected, x);
}
```

你会发现`weakCompareAndSetIntPlain`方法最后还是调用`compareAndSetInt`，是不是感觉很奇怪。如果你对@HotSpotIntrinsicCandidate这个注解有了解，可能就会明白原因了。

@HotSpotIntrinsicCandidate注解是特定于Java虚拟机的注解。通过该注解表示的方法可能( 但不保证 )通过HotSpot VM自己来写汇编或IR编译器来实现该方法以提供性能。也就是说虽然外面看到的在JDK9中weakCompareAndSet和compareAndSet底层依旧是调用了一样的代码，但是不排除HotSpot VM会手动来实现weakCompareAndSet真正含义的功能的可能性。

参考链接：
1. https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6275329
2. https://stackoverflow.com/questions/36428044/whats-the-difference-between-compareandset-and-weakcompareandset-in-atomicrefer
3. https://www.jianshu.com/p/55a66113bc54
4. https://blog.csdn.net/lzcaqde/article/details/80868854