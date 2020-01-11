---
title: J.U.C之SynchronousQueue
date: 2020-1-8 18:00:00
tags: java
category: J.U.C
---

在JDK源码里，有个非常常用的阻塞队列，就是SynchronousQueue，它的put操作必须等到其他线程take出其元素才可以继续往下执行，内部没有使用队列或则数组存储。可能比较类似于ArrayBlockingQueue长度为0的情况，不过我们无法创建长度为0的ArrayBlockingQueue。


## 常用方法

```java
//公平模式使用队列模式，否则使用栈模式
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

```java
//阻塞放
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
  	//放入元素
    if (transferer.transfer(e, false, 0) == null) {
      	//重置中断状态，抛出中断异常
        Thread.interrupted();
        throw new InterruptedException();
    }
}
```

<!--more-->

```java
//非阻塞放，可能放入失败
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
  	//可以看到主要逻辑全部封装到transfer方法
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}
```

```java
//阻塞取方法
public E take() throws InterruptedException {
  	//同样调用transfer方法
    E e = transferer.transfer(null, false, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```

可以看到所有的添加取出方法都被封装到transfer方法中，transfer有两种实现，公平和非公平模式，这里分别介绍。

## 队列实现

```java
//如果是put，则e!=null，如果是take，则e==null
//timed=true代表了设置超时时间，nanos代表超时时间
E transfer(E e, boolean timed, long nanos) {
    QNode s = null; // constructed/reused as needed
  	//判断当前请求是data(put)还是request(take)
    boolean isData = (e != null);
    for (;;) {
        QNode t = tail;
        QNode h = head;
      	//还未初始化，等待初始化完成，为什么呢？
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin
				//队列为空或则前一个和当前都是相同类型(同时take或者put请求)
        if (h == t || t.isData == isData) { // empty or same-mode
            QNode tn = t.next;
          	//队列发生改变，重新循环
            if (t != tail)                  // inconsistent read
                continue;
          	//tail指针未正确指向末尾节点，更新后重试
            if (tn != null) {               // lagging tail
                advanceTail(t, tn);
                continue;
            }
          	//如果超时时间已到，返回null
            if (timed && nanos <= 0)        // can't wait
                return null;
          	//如果请求还未封装成QNode节点，将当前请求包装成QNode节点
            if (s == null)
                s = new QNode(e, isData);
          	//将请求节点挂载到队列末尾
            if (!t.casNext(null, s))        // failed to link in
                continue;
						//修正tail指针
            advanceTail(t, s);              // swing tail and wait
          	//将线程挂起等待
            Object x = awaitFulfill(s, e, timed, nanos);
          	//如果超时或中断等，将节点清理，返回失败
            if (x == s) {                   // wait was cancelled
                clean(t, s);
                return null;
            }
						//如果不是无效节点则处理next == this
            if (!s.isOffList()) {           // not already unlinked
              	//head向后移动
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {                            // complementary-mode
          	//取第一个节点
            QNode m = h.next;               // node to fulfill
          	//如果被更新则重新尝试
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read
            Object x = m.item;
            //isData == (x != null)说明是相同模式
            //x == m 自循环，说明已取消
            // m.casItem(x, e)将m节点的item修改为e
            if (isData == (x != null) ||    // m already fulfilled
                x == m ||                   // m cancelled
                !m.casItem(x, e)) {         // lost CAS
                advanceHead(h, m);          // dequeue and retry
                continue;
            }
            //执行到这里说明item修改成功，尝试修改head头
            advanceHead(h, m);              // successfully fulfilled
            //唤醒对应线程
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}
```

```java
//线程等待挂起代码
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    //自旋次数，如果不是第一个节点，就不用自旋
    //如果定时等待则为maxTimedSpins=(NCPUS < 2) ? 0 : 32;这个值也是经验之谈
    //maxUntimedSpins = maxTimedSpins * 16;
    int spins = ((head.next == s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
      	//如果中断，则标记节点为取消，通过将item标记为自身this实现
        if (w.isInterrupted())
            s.tryCancel(e);
        Object x = s.item;
        //说明被其他线程交换了，返回其他线程的值，如果其他线程是take，则返回null
        if (x != e)
            return x;
      	//超时判断
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
      	//自旋
        if (spins > 0)
            --spins;
      	//将当前线程设置在QNode节点中
        else if (s.waiter == null)
            s.waiter = w;
      	//如果不是超时获取，则挂起向前线程
        else if (!timed)
            LockSupport.park(this);
      	//如果大于等待的阈值，这定时挂起，阈值为1000L
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```



## 栈实现

```java
//item和mode不是volatile类型，下面注释说明了这两个变量会在其他volatile变量前些，在其后读
static final class SNode {
    volatile SNode next;        // next node in stack
    volatile SNode match;       // the node matched to this
    volatile Thread waiter;     // to control park/unpark
    Object item;                // data; or null for REQUESTs
    int mode;
    // Note: item and mode fields don't need to be volatile
    // since they are always written before, and read after,
    // other volatile/atomic operations.
}
```

```java
E transfer(E e, boolean timed, long nanos) {
    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA;

    for (;;) {
        SNode h = head;
      	//如果栈空或则和栈内模式相同
        if (h == null || h.mode == mode) {  // empty or same-mode
            if (timed && nanos <= 0) {      // can't wait
                //超时处理
                if (h != null && h.isCancelled())
                    //弹出取消节点
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null;
            } else if (casHead(h, s = snode(s, e, h, mode))) {//尝试入栈
                //awaitFulfill和上面的思路一致，不在分析
                SNode m = awaitFulfill(s, timed, nanos);
                //节点被取消
                if (m == s) {               // wait was cancelled
                    clean(s);
                    return null;
                }
                //如果等待期间内head被修改，且head和next为当前节点，则修改head头
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        } else if (!isFulfilling(h.mode)) { // try to fulfill
            //如果有线程尝试交换数据
            //如果head是取消节点，则移动头指针
            if (h.isCancelled())            // already cancelled
                casHead(h, h.next);         // pop and retry
            //标记头结点为当前节点，并设置正在交换数据
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                for (;;) { // loop until matched or waiters disappear
                    SNode m = s.next;       // m is s's match
                    //如果没有元素可以交换，修改头指针，跳出循环
                    if (m == null) {        // all waiters are gone
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                    //尝试交换，如果成功，头指针向后移动两位
                    if (m.tryMatch(s)) {
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else                  // lost match
                        //如果交换失败，尝试继续和下一个元素交换，这里移动节点
                        s.casNext(m, mn);   // help unlink
                }
            }
        } else {                            // help a fulfiller
            //进入此方法说明，当前和栈内模式不同，但有其他线程正在交换
            SNode m = h.next;               // m is h's match
            //头指针过时，修改头指针
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                //这里的逻辑和上面的相似，不过不在循环内
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink
            }
        }
    }
}
```