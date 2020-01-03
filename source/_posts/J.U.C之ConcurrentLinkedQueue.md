---
title: J.U.C之ConcurrentLinkedQueue
date: 2020-1-7 18:00:00
tags: java
category: J.U.C
---

## 再聊ABA问题

前面在学习AtomicStampedReference和AtomicMarkableReference是提过ABA问题，但是没有举出一个例子，今天我们就来了解下ABA与并发队列之间的问题。

在C或C++中，我们知道代码是直接控制指针，指针指向的是内存地址，在C或C++中实现无锁队列时，如果一个元素出队列，内存被回收，又有个节点入队，内存地址恰好和前一个出队地址一致，这样其他线程看来，就可能出现ABA问题。

如果你看过一些JDK的源码，你会发现JDK中无锁队列有这样一个假设，不会出现ABA问题，如下说明


> Note that like most non-blocking algorithms in this package, this implementation relies on the fact that in garbage collected systems, there is no possibility of ABA problems due to recycled nodes, so there is no need to use "counted pointers" or related techniques seen in versions used in non-GC'ed settings.



可以看到，因为垃圾回收的存在，基本可以不用考虑ABA问题，也就不要变更计数等CAS操作。其实原因也比较好理解，一个被持有的引用是不会被垃圾回收器回收，从而不会两个对象分配到同个内存空间。如果对无锁队列和ABA感兴趣，可以参阅相关链接。

## ConcurrentLinkedQueue

### 内部节点结构

```java
//内部的节点很简单，就是一个item和一个next
private static class Node<E> {
    volatile E item;
    volatile Node<E> next;
 	//...
}
```

### 外部结构

```java
//可以看到维护了一个head和一个tail，看起来也比较正常
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {
    //javadoc提到一点，head有可能无法到达tail，这点比较有意思
    private transient volatile Node<E> head;
    private transient volatile Node<E> tail;
    //...
}  
```

### 构造器

```java
//初始化构造器，构造一个节点，内容为空，我们是无法把null通过添加方法加入队列
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}
```

```java
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
      	//基本上大部分的并发集合都不允许空元素
        checkNotNull(e);
        Node<E> newNode = new Node<E>(e);
        if (h == null)
            h = t = newNode;
        else {
          	//用lazySet就是为了提高性能，前面原子AtomicInteger介绍过，这里算一个使用
            t.lazySetNext(newNode);
            t = newNode;
        }
    }
  	//空集合特殊处理
    if (h == null)
        h = t = new Node<E>(null);
    head = h;
    tail = t;
}
```

### 添加方法

```java
//这个方法只会返回true，还可能抛出NullPointerException
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);
	//这是个死循环
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
      	//说明tail后面没有元素，可以尝试添加到链表结尾
        if (q == null) {
          	//添加成功
            if (p.casNext(null, newNode)) {
              	//说明tail已经过时，更新tail指针
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            p = (t != (t = tail)) ? t : head;
        else
            //tail已经过期，这里修正位置
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

#### 单线程执行

这里的方法单看还是比较迷惑的，我们可以先按单线程来分析，offer两个元素进去，观察程序的执行流程



![单线程插入流程](http://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202013161329.svg)



值得注意的是tail的变更，在第一插入的时候，tail并没有更新，在第二次插入的时候才会更新tail节点。



#### 并发执行

```java
if (p != t) // hop two nodes at a time
    casTail(t, newNode); // Failure is OK.
```

从上面单线程分析得知，这是更新tail指针的代码，`casTail(t, newNode);`什么时候会失败呢？

假设一个线程执行到`casTail(t, newNode);`时让出了CPU，此时内存结构如下图所示

![多线程插入](http://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202013163218.svg)



如果此时有个线程调用offer方法，最后也会执行到上面的语句，这个时候只会有一个会更新成功，tail有可能指向第三个元素，也有可能指向第二个元素，但是tail不需要与尾结点强一致，这里也就没有额外处理。



---



```java
else if (p == q)
    p = (t != (t = tail)) ? t : head;
```

这个代码也比较有意思，我们知道`Node<E> q = p.next;`，那么为什么p在什么情况下会等于q呢？这个需要了解下出队的逻辑，这个待会分析。



## 获取方法

```java
public E peek() {
  restartFromHead:
  for (;;) {
    for (Node<E> h = head, p = h, q;;) {
      E item = p.item;
      //如果当前节点由内容，或则没有后继节点，则返回当前节点内容
      if (item != null || (q = p.next) == null) {
        //如果h!=p，则尝试更新头节点，不一定会更新成功
        updateHead(h, p);
        return item;
      }
      else if (p == q)
        continue restartFromHead;
      else
        //向后移动一位指针
        p = q;
    }
  }
}
```

```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
          	//如果节点包含元素，尝试取出，如果成功取出则返回
            if (item != null && p.casItem(item, null)) {
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
          	//已经没有后继节点了，返回null
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
            else
              	//向后移动节点
                p = q;
        }
    }
}
```



和添加方法类似，取出方法会修改head头，同样我们按照单线程的模式思考是如何出队的。

第一个线程会把head指向的节点置位null，因为`p==h`，所以会直接返回item。

第二个线程进入后会发现head的节点item=null，然后会执行`p = q;`，将节点向后移，然后`p.casItem(item, null)`成功，但此时发现`p != h`为true，这时候会修正head节点。



可以看到在出队操作时也会有`p == q`的判断，了解这个之前，先了解下如何修正head头结点

```java
//可以看到h的next会设置为其本身，为什么要这样做呢，应该是为了获取后继节点方便，如果后继节点设置为空，就无法知道是节点的末尾还是一个无效的节点
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h);
}
//获取后继节点方法
final Node<E> succ(Node<E> p) {
  Node<E> next = p.next;
  return (p == next) ? head : next;
}
```

上面的代码可以看出，调用updateHead方法时，h节点(老的head头)会把自己的next指向自己，这个时候我们再来回顾下poll方法。



上面说了第二个线程会修正头结点，假设第二个线程B和第三个线程C同时进入poll方法，B线程还是按照刚刚的逻辑刚执行完updateHead方法，此时C线程开始执行，此时C线程中的h，p还是老的头结点，其next元素已经被修改为自己，这样C线程就会执行到`if (p == q)`且返回true。关于offer方法中判断可以构造类似的条件，当p恰好是头结点，这样如果同时被出队，就会发生p.next=p的情况。

## 迭代器

ConcurrentLinkedQueue的迭代器构造器如下

```java
private class Itr implements Iterator<E> {
    //下个节点
    private Node<E> nextNode;
		//下个节点的item
    private E nextItem;
  	//最后一个节点
    private Node<E> lastRet;
		//构造器直接调用了advance方法
    Itr() {
        advance();
    }

    //直接判断是下个节点是否为空
    public boolean hasNext() {
        return nextNode != null;
    }
		//和构造器一样，直接调用advance方法
    public E next() {
        if (nextNode == null) throw new NoSuchElementException();
        return advance();
    }
		//为什么要定义个l变量呢
    public void remove() {
        Node<E> l = lastRet;
        if (l == null) throw new IllegalStateException();
        // rely on a future traversal to relink.
        l.item = null;
        lastRet = null;
    }
}
```



可以看到逻辑主要是在advance方法中

```java
private E advance() {
  	//如果是构造器进来nextNode，nextItem为默认值，也就是null
    lastRet = nextNode;
    E x = nextItem;

    Node<E> pred, p;
    if (nextNode == null) {
      	//构造器进来会查找首节点
        p = first();
        pred = null;
    } else {
        pred = nextNode;
        p = succ(nextNode);
    }

    for (;;) {
        if (p == null) {
          	//说明已经到队列末尾，全部标记为空
            nextNode = null;
            nextItem = null;
            return x;
        }
        E item = p.item;
        if (item != null) {
          	//设置nextnode和nextitem
            nextNode = p;
            nextItem = item;
            return x;
        } else {
            // skip over nulls
          	//跳过空节点，可能正在出队，或已经出队
            Node<E> next = succ(p);
            if (pred != null && next != null)
                pred.casNext(p, next);
            p = next;
        }
    }
}
```



## 奇怪的问题

这里在阅读源码的时候遇到一个比较诡异的问题

![未执行查询时的值](http://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/202013235341.png)

可以看到head=tail=512也就是空节点

![查询head值](http://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/20201323539.png)

可是如果你执行语句查询head值，或者在poll方法里打断点(360行)，你会发现head竟然莫名指向了第一个元素。

这里有知道的可以解答下，着实想不明白。





## 参考链接

1. https://juejin.im/post/5d4789ca51882519ac307a6f
2. http://ifeve.com/%e5%b9%b6%e5%8f%91%e9%98%9f%e5%88%97-%e6%97%a0%e7%95%8c%e9%9d%9e%e9%98%bb%e5%a1%9e%e9%98%9f%e5%88%97concurrentlinkedqueue%e5%8e%9f%e7%90%86%e6%8e%a2%e7%a9%b6/