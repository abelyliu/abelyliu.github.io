---
title: J.U.C之TransferQueue
date: 2020-1-9 18:00:00
tags: java
category: J.U.C
---

上文我们聊了SynchronousQueue，如果你观察J.U.C，你会发现TransferQueue可以实现和SynchronousQueue相同的功能，而且提供更好的并发性。
TransferQueue目前只有一个子类LinkedTransferQueue，我们观察下面的类图

![LinkedTransferQueue类图](http://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202014222212.png)

```java
public interface TransferQueue<E> extends BlockingQueue<E> {
    //尝试交换
    boolean tryTransfer(E e);
    //尝试交换，如果没有线程交换则阻塞
    void transfer(E e) throws InterruptedException;
    //在一定时间内尝试交换
    boolean tryTransfer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
    //是否有等待交换的消费者
    boolean hasWaitingConsumer();
    //有多少个等待的消费者
    int getWaitingConsumerCount();

}
```

<!--more-->

LinkedTransferQueue的主要和上文SynchronousQueue类类似，poll，offer等方法都实质上是xfer的封装。

我们主要分析下xfer方法的实现逻辑

```java
private E xfer(E e, boolean haveData, int how, long nanos) {
    if (haveData && (e == null))
        throw new NullPointerException();
    Node s = null;                        // the node to append, if needed

    retry:
    for (;;) {                            // restart on append race

        for (Node h = head, p = h; p != null;) { // find & match first node
            boolean isData = p.isData;
            Object item = p.item;
            //item != p说明没有自引用，是有效节点
            //(item != null) == isData说明节点没有被交换
            if (item != p && (item != null) == isData) { // unmatched
                //如何和当前进入线程模式相同，则跳出循环
                if (isData == haveData)   // can't match
                    break;
                //尝试和头结点交换
                if (p.casItem(item, e)) { // match
                    //如果线程发现第一个元素(head)正在被交换，他会尝试和第二个元素交换
                    //这样q!=h
                    for (Node q = p; q != h;) {
                        Node n = q.next;  // update by 2 unless singleton
                        //修改head头
                        if (head == h && casHead(h, n == null ? q : n)) {
                            //将老head标记为取消next指向自己
                            h.forgetNext();
                            break;
                        }                 // advance and retry
                        //!q.isMatched()代表正在q正在被交换，继续循环，寻找下一个头结点
                        if ((h = head)   == null ||
                            (q = h.next) == null || !q.isMatched())
                            break;        // unless slack < 2
                    }
                    //将p节点对应的线程唤醒
                    LockSupport.unpark(p.waiter);
                    return LinkedTransferQueue.<E>cast(item);
                }
            }
            //如果p节点没有被取消，则下移节点，否则重置头结点
            Node n = p.next;
            p = (p != n) ? n : (h = head); // Use head if p offlist
        }
        //执行到这里说明没有节点可以配对
        //如果是poll，tryTransfer方法，则直接返回，不需要等待
        if (how != NOW) {                 // No matches available
            //封装当前请求成为Node节点
            if (s == null)
                s = new Node(e, haveData);
            //添加到队列末尾,tryAppend里面值得注意的是，如果一个offer，一个take进来，只能初始化一种队列
            Node pred = tryAppend(s, haveData);
            //如果添加失败，重新尝试
            if (pred == null)
                continue retry;           // lost race vs opposite mode
            //如果不是put，offer，add方法，直接返回，否则挂起
            //上述四个方法,offer有个重载方法，队列时无界的，不会阻塞添加
            if (how != ASYNC)
              	//挂起，等待其他线程前来匹配
                return awaitMatch(s, pred, e, (how == TIMED), nanos);
        }
        return e; // not waiting
    }
}
```


```java
private E awaitMatch(Node s, Node pred, E e, boolean timed, long nanos) {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    //检查之后才会初始化
    int spins = -1; // initialized after first item and cancel checks
    ThreadLocalRandom randomYields = null; // bound if needed

    for (;;) {
        Object item = s.item;
        if (item != e) {                  // matched
            // assert item != s;
            s.forgetContents();           // avoid garbage
            return LinkedTransferQueue.<E>cast(item);
        }
        if ((w.isInterrupted() || (timed && nanos <= 0)) &&
                s.casItem(e, s)) {        // cancel
            //清理节点
            unsplice(pred, s);
            return e;
        }
		//初始化自旋
        if (spins < 0) {                  // establish spins at/near front
            if ((spins = spinsFor(pred, s.isData)) > 0)
                randomYields = ThreadLocalRandom.current();
        }
        //自旋
        else if (spins > 0) {             // spin
            --spins;
            if (randomYields.nextInt(CHAINED_SPINS) == 0)
                Thread.yield();           // occasionally yield
        }
        else if (s.waiter == null) {
            s.waiter = w;                 // request unpark then recheck
        }
        //定时挂起
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos > 0L)
                LockSupport.parkNanos(this, nanos);
        }
        else {
            //非定时挂起
            LockSupport.park(this);
        }
    }
}
```

```java
final void unsplice(Node pred, Node s) {
    s.forgetContents(); // forget unneeded fields
    /*
     * See above for rationale. Briefly: if pred still points to
     * s, try to unlink s.  If s cannot be unlinked, because it is
     * trailing node or pred might be unlinked, and neither pred
     * nor s are head or offlist, add to sweepVotes, and if enough
     * votes have accumulated, sweep.
     */
    //pred==null，不用处理，当前节点已删除，head会重新定位
    //pred==s 说明s是首节点，也不需要处理
    //pred.next != s 说明pred节点已处理完毕
    if (pred != null && pred != s && pred.next == s) {
        Node n = s.next;
        if (n == null ||
            //将当前节点的前驱和后继链接，如果前节点已经匹配，则需要移动head头
            (n != s && pred.casNext(s, n) && pred.isMatched())) {
            for (;;) {               // check if at, or could be, head
                Node h = head;
                if (h == pred || h == s || h == null)
                    return;          // at head or list empty
                if (!h.isMatched())
                    break;
                Node hn = h.next;
                if (hn == null)
                    return;          // now empty
                if (hn != h && casHead(h, hn))
                    h.forgetNext();  // advance head
            }
          	//上面有可能没有修改head头，这样next引用还在，这里到达一定次数修正链表
            if (pred.next != pred && s.next != s) { // recheck if offlist
                for (;;) {           // sweep now if enough votes
                    int v = sweepVotes;
                    if (v < SWEEP_THRESHOLD) {
                        if (casSweepVotes(v, v + 1))
                            break;
                    }
                    else if (casSweepVotes(v, 0)) {
                        sweep();
                        break;
                    }
                }
            }
        }
    }
}
```


