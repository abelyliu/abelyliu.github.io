---
title: AbstractQueuedSynchronizer源码分析(三)
date: 2019-12-25 18:00:00
tags: java
category: AQS
---

前面两篇文章分析了AbstractQueuedSynchronizer中锁和线程协作的代码，这篇来分享一些其他的细节点，主要是我比较好奇的一些点。

#### AQS排队队列为什么叫CLH队列？

不知道你是否有过这个疑问，其实准确的应该是类CLH队列，如果你仔细看过AbstractQueuedSynchronizer中Node的注释，应该明白CLH代表着Craig, Landin, Hagersten，他们当时给出了CLH队列用于实现自旋锁，这里被稍加改造。但基本思路还是一致，前驱节点会提供一些控制信息。更多的会叫sync队列。

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

设置节点状态为取消的有两处地方

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