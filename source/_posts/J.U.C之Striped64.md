---
title: J.U.C之Striped64
date: 2020-1-1 18:00:00
tags: java
category: J.U.C
---

前文我们分享了AtomicInteger的源码，其就是对Unsafe的封装，我们会发现AtomicBoolean，AtomicLong等代码都是类似的。但在atomic包下，还提供了并发程度更高的LongAdder和LongAccumulator。

同样的，我们先观察下LongAdder的api

```java
//自增x
public void add(long x) {
    //...
}
//自增1
public void increment() {
    add(1L);
}
//自减1
public void decrement() {
    add(-1L);
}
//目前累加值
public long sum() {
   //...
}
//重置当前累加值
public void reset() {
    //...
}
//重置当前累加值，并返回重置前的累加值
public long sumThenReset() {
    //...
}
```
从api中，明显可以看出来，这个类一般用来计数，而且在竞争激烈的环境中可以提供比AtomicLong更好的并发效果。
<!--more-->

![Striped64子类](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/Striped64%E5%AD%90%E7%B1%BB.svg)

我们可以看到LongAdder是Striped64子类，下文我们会讨论LongAccumulator。这几个类之所以能提供更好的并发效果，就在Striped64的设计思路。

如果你对JAVA7中的ConcurrentHashMap有所了解，应该知道其中使用了分段锁的策略，而Striped64的思路也是一样，通过分段锁来减少CAS修改失败的概率。

![Striped64结构示意图](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/Striped64%E7%BB%93%E6%9E%84.svg)

从上图我们可以看出，我们可以把不同线程修改分散到不同地方，则样可以大大减少冲突概率。对于LongAdder这种，我们可以吧base value和所有Cell数组里所有的value求和，这样就可以得到最终的值。

```java
//LongAdder的sum实现
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    //如果Cell数组不为空，则将数组里的值和base求和
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

下面我们分析下Striped64中的longAccumulate方法
```java
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) {
        //初始化线程变量，用于哈希，判断锁落在的区间
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        if ((as = cells) != null && (n = as.length) > 0){
            //如果分段区间已经初始化，则判断当前线程所处分段区间，并处理
        }
        else if (cellsBusy == 0 && cells == as && casCellsBusy()){
            //如果分段区间还未初始化，则初始化分段区间
        }
        else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x)))){
            //如果分段区间还在初始化过程中，则尝试修改base值
            break;
        }
    }
}
```

我们分析下初始化Cell数组的代码

```java
//cellsBusy==0代表没有线程进行初始化
//cells == as代表cells目前为空或则长度为0
//casCellsBusy方法则通过cas修改cellsBusy值为1
else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
    //这里可以思考下，为什么需要init状态，为什么需要cells == as的判断
    boolean init = false;
    try {
        if (cells == as) {
            //初始化Cell数组长度为2，并计算当前线程hash后所处的区间，然后设置值为x
            //这里直接将值设置为x，并没有使用LongBinaryOperator fn方法
            Cell[] rs = new Cell[2];
            rs[h & 1] = new Cell(x);
            cells = rs;
            init = true;
        }
    } finally {
        cellsBusy = 0;
    }
    if (init)
        break;
}
```

上面代码需要init的原因在于，如果两个线程同时执行完`cellsBusy == 0 && cells == as`，其中一个线程初始化完成，然后break出循环。这个时候另一个线程唤醒，`casCellsBusy()`也会执行成功，但此时`cells != as`，cells已经不再为空。

接下来问题的核心就在于Cell数组不为空的情况

```java
if ((as = cells) != null && (n = as.length) > 0) {
    //如果当前线程对应的槽为空，还没有线程进来过
    if ((a = as[(n - 1) & h]) == null) {
        //cellsBusy==0意味着cell数组没有在修改
        if (cellsBusy == 0) {       // Try to attach new Cell
            Cell r = new Cell(x);   // Optimistically create
            //cell状态设置为修改中
            if (cellsBusy == 0 && casCellsBusy()) {
                //将对应cell数组中对应位置初始化
                boolean created = false;
                try {               // Recheck under lock
                    Cell[] rs; int m, j;
                    if ((rs = cells) != null &&
                        (m = rs.length) > 0 &&
                        rs[j = (m - 1) & h] == null) {
                        rs[j] = r;
                        created = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (created)
                    break;
                continue;           // Slot is now non-empty
            }
        }
        collide = false;
    }
    //wasUncontended是曾经更新cell值得结果，如果更新过，则wasUncontended一定为false，否则就不需要进入longAccumulate方法
    //如果没有更新过cell，wasUncontended则为true
    //如果更新过，则修改wasUncontended为true，然后执行 h = advanceProbe(h);将当前线程的hash值修改，因为曾经尝试修改cell失败，所以尝试修改hash，分配到其它槽
    else if (!wasUncontended)       // CAS already known to fail
        wasUncontended = true;      // Continue after rehash
    //如果没有尝试过，则尝试修改cell里的值，如果成功则直接返回
    else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                 fn.applyAsLong(v, x))))
        break;
    //如果当前cell数组长度大于cpu数量或则cells在上述判断过程中被修改，则认为不需要扩容，或则扩容已到上限，rehash从新分配槽
    else if (n >= NCPU || cells != as)
        collide = false;            // At max size or stale
    //如果是第一次尝试cas修改cell失败(在for循环内第一次)，则进行rehash从新分配槽
    else if (!collide)
        collide = true;
    //如果不是第一次修改cas失败，cell数组也没有达到数组长度上限，则进行数组扩容
    else if (cellsBusy == 0 && casCellsBusy()) {
        try {
            if (cells == as) {//理由同上
                //cell数组按2倍扩容
                Cell[] rs = new Cell[n << 1];
                //转移老数组中的元素
                for (int i = 0; i < n; ++i)
                    rs[i] = as[i];
                cells = rs;
            }
        } finally {
            cellsBusy = 0;
        }
        collide = false;
        continue;                   // Retry with expanded table
    }
    h = advanceProbe(h);
}
```

![longAccumulate流程图](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/longAccumulate.svg)

这个时候我们在回头看下LongAdder的方法
```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    //如果cell[]不为空，或则base value修改失败
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        //1. cell[]为空
        //2. cell[]长度为0
        //3. 如果当前线程对应的槽位对象为空
        //4. 尝试修改线程对应槽位的值失败
        //上面四种情况则调用父类的longAccumulate修改
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

```java
//方法注释也说明，这个需要保证没有线程并发修改
public void reset() {
    Cell[] as = cells; Cell a;
    //base value置为0
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            //每个非空cell置为0
            if ((a = as[i]) != null)
                a.value = 0L;
        }
    }
}
```

理解了LongAdder的代码，LongAccumulator也是类似的
```java
//可以初始化base值，而且需要一个函数，接收两个long类型参数，返回一个long类型
public LongAccumulator(LongBinaryOperator accumulatorFunction,
                       long identity) {
    this.function = accumulatorFunction;
    base = this.identity = identity;
}
```

```java
public void accumulate(long x) {
    Cell[] as; long b, v, r; int m; Cell a;
    //和LongAdder基本一样
    if ((as = cells) != null ||
        (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended =
               //这个地方使用方法计算更新后的值应该为多少，LongAdder是直接相加
              (r = function.applyAsLong(v = a.value, x)) == v ||
              a.cas(v, r)))
            longAccumulate(x, function, uncontended);
    }
}
```

```java
public long get() {
    Cell[] as = cells; Cell a;
    long result = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                //对每个cell中的值执行function
                result = function.applyAsLong(result, a.value);
        }
    }
    return result;
}
```

```java
//重置方法
public void reset() {
    Cell[] as = cells; Cell a;
    base = identity;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                //这个地方值得注意，不为空的cell直接重置为identity
                a.value = identity;
        }
    }
}
```

我们可以看到LongAdder就是LongAccumulator的一个特列，当LongAccumulator的function为`(x, y) -> x + y`，identity为0时，就是LongAdder。
