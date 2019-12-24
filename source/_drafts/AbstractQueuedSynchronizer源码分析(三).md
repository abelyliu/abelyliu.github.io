---
title: AbstractQueuedSynchronizer源码分析(三)
date: 2019-12-25 18:00:00
tags: java
category: AQS
---

前面两篇文章分析了AbstractQueuedSynchronizer中锁和线程协作的代码，这篇来分享一些其他的细节点，主要是我比较好奇的一些点。

#### AQS排队队列为什么叫CLH队列？

不知道你是否有过这个疑问，其实准确的应该是类CLH队列，如果你仔细看过AbstractQueuedSynchronizer中Node的注释，应该明白CLH代表着Craig, Landin, Hagersten，他们当时给出了CLH队列用于实现自旋锁，这里被稍加改造用于实现`blocking synchronizers`。但基本思路还是一致，前驱节点会提供一些控制信息。有时候也会叫sync队列。

#### Node节点会有哪些状态？

|   常量表示  |   值  |  含义   |
| :---: | :---: | :---: |
|   SIGNAL  |   -1  |  当前节点需要唤醒后继节点   |
|   CANCELLED  |   1  |   当前节前取消状态  |
|   CONDITION  |   -2  |    当前节点在condition队列里初始化状态 |
|   PROPAGATE  |   -3  |   当CLH队列最后一个元素获得共享锁时设置，让下个获取共享锁的线程可直接获取到  |
|   0  |  0   |   进入CLH队列默认状态，等待获取锁  |

1. 只有取消状态CANCELLED的值大于0，所以在代码里有很多地方判断状态是否大于0，就是判断是否为取消状态
2. 在独占锁的请求中，状态只会有0，CANCELLED，SIGNAL三种
3. 在共享锁的请求中，状态只会有0，CANCELLED，SIGNAL，PROPAGATE四种
4. condition队列里只有CONDITION和CANCELLED两种状态
5. 代码里很多地方用状态是否小于0来判断是否需要唤醒后继线程，结合上面很容易得到
<!--more-->

#### 什么时候会node节点会设置为取消状态？

不知道你是否有过疑问，代码里有很多处理node节点为取消的情况，确没有看到什么地方可以把状态设置为取消。

其实这个如果你仔细阅读代码其实是能发现问题的，而我在前两篇文章中并没有提及那块代码。一开始也说过原因，前两篇文章主要分析核心流程，抛开一些细枝末节，如异常处理(虽然也很重要)，这样更容易抓住重点。

设置节点状态为取消的有两处地方，不知道你是否还记得一共有两种队列，一种sync队列，一种condition队列，正好对应这两种情况，我们一一来分析：

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

![fullyRelease调用链](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/Snipaste_2019-12-24_20-57-32.png)

方法的入口在await的各种重载方法中，可以看到，如果在释放锁失败，或者出现异常时，我们会把刚刚添加到condition队列中的节点设置为删除。

举个最简单的例子，如果没有在锁的范围内调用await方法，我们知道会抛出IllegalMonitorStateException，这个时候node节点已经添加到condition队列里，需要移除，这里会将状态设置为取消，待下一个进入condition队列的线程在添加到condition队列时会发现取消的节点，然后会把其移除。

接下来我们看下sync队列里的取消状态是如何设置的

```java
private void cancelAcquire(Node node) {
    //巴拉巴拉
    node.waitStatus = Node.CANCELLED;
   //巴拉巴拉
}
```
![cancelAcquire调用链](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/Snipaste_2019-12-24_21-11-48.png)

随便找个方法看看
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
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
        if (failed)
            cancelAcquire(node);
    }
}
```

可以看到在自旋休眠获取锁的过程中，出现了异常，failed值则会为true，然后会将当前节点设置为取消状态。

这里以ReentrantLock的FairSync(公平锁)举例说明，我们可以看到上面代码中会掉用 tryAcquire(arg)，我们看下FairSync里的实现
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

我们可以看到代码里有可能会抛出`ERROR`，为什么nextc小于0会抛出超过最大锁上线的错误？不知道有没有想过`Integer.MAX_VALUE + 1`的值是多少。

在这种情况下和上面场景一样，需要将错误和异常传递到方法的调用方，但在异常传递的过程中需要将节点处理掉，不然会影响后续节点的锁资源竞争。

#### cancelAcquire代码分析
上面提到了在异常情况下，会调用cancelAcquire方法，取消node节点，这里简单分析cancelAcquire的代码

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;
    node.thread = null;
    //设置当前的节点的前驱为一个非取消状态节点(如果前驱本来就不是取消状态，则不会有任何变化)
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    
    //获取前驱节点的后继节点，如果当前节点的前驱本来就不是取消状态，那么preNext还是节点本身
    Node predNext = pred.next;
    
    //标记当前节点为已删除
    node.waitStatus = Node.CANCELLED;
    
    if (node == tail && compareAndSetTail(node, pred)) {
        //如果当前节点本来就是尾节点，将tail标记为前驱即可，并将前驱的后继节点设置为空
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        //pred == head则意味着当前接单为sync中第一个节点(或者前驱都是删除状态)，这样直接唤醒后继节点即可
        //如果当前节点要取消，则需要把当前节点需要唤醒后继节点的任务交给前驱节点，所以需要判断前驱节点的状态是否是SIGNAL
        //(ws = pred.waitStatus) == Node.SIGNAL || ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))
        //这里可以看到如果不是SIGNAL则将其置为SIGNAL，如果设置失败，则直接唤醒后继节点，后继节点会自己修改前驱节点的状态(相当于将设置任务转移)
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                //这里将当前节点的前驱的next指向当前节点的next，这样前驱获得锁后，就能唤醒当前节点的后继
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

![获取锁取消队列示意图](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/%E8%8E%B7%E5%8F%96%E9%94%81%E5%8F%96%E6%B6%88%E9%98%9F%E5%88%97%E7%A4%BA%E6%84%8F%E5%9B%BE.svg)

上面图片中，上半部分是初始队列假设示意图，下半部分是代码执行后示意图

如果你观察上面图会发现，node4的前驱还是取消节点，cancelAcquire方法里并没有把node4的前驱修改，cancelAcquire的目的是把node1只想node4，这样node1获取锁后，能正确唤醒node4线程，当node4被唤醒后，自己会在shouldParkAfterFailedAcquire方法内部修改前驱节点

#### AbstractQueuedSynchronizer和AbstractQueuedLongSynchronizer是什么关系？