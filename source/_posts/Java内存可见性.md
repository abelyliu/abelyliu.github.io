---
title: Java内存可见性
date: 2017-02-20 10:12:58
tags: Java
category: Java
---
在并发的环境中有三个问题比较常见，原子性，可见性，有序性。原子性比较好理解，也就是代码中的一句话，一行代码，一句赋值在计算机执行时可能需要许多条指令，任意指令之间都有可能中断，这也是最容易理解的问题。

我第一次接触内存可见性问题是在阅读《Java并发编程实战》一书，书中有如下代码：

```java
public class NoVisibility{
    private static boolean ready;
    private static int number;
    
    private static class ReaderThread extends Thread{
        public void run(){
            while(!ready){
                Thread.yield();
            }
            System.out.println(number);
        }
    }
    public static void main(String[] args){
      new ReaderThread().start();
      number=42;
      ready=true;
    }
}
```
书中提到这个程序可能一直循环下去(内存可见性)或者输出0(重排序)，这对我我造成了很大的困惑。直到偶然间看见Java的内存模型来解释，就容易理解了。

上述程序可以一直无法停止的原因在于main线程中对read的修改，在ReaderThread线程中无法读取到，ReaderThread线程中ready一直为false，main线程中为true，这也就是所谓的内存可见性问题。

<!--more-->

## Java内存模型

![](/images/59.png)

对于上述代码而言，主内存中ready为false，main线程中copy一份到线程A然后修改为true，ReaderThread线程中也会从主内存中copy一份到本地内存B，然后执行循环。

出现无限循环的情况，应该是有两种情况，一种是main线程修改变量后未同步到主内存，二是ReaderThread线程直接从本地线程读取，未从主内存读取(第一次从主内存读取时，ready还未修改)。

当然，代码也并不是总是出现这种情况，本人测试上面代码时并未出现无限循环和0，但是在遇到其他相似的代码中遇到过，并发问题有时候就是概率问题，有时候复现确实很困难。

## 重排序
上述代码为什么可能会出现0呢？我们可以观察main方法如下代码：
```java
number=42;
ready=true;
```
变成这样:
```java
ready=true;
//此处发生中断，执行ReaderThread线程中代码
number=42;
```
如果发生了上述的改变，代码输出0是不是就可解释通了。问题在与为什么程序的代码会被系统修改？

我们知道一条指令可以分为许多步骤：
- 取指 IF
- 译码和取寄存器操作数 ID
- 执行或者有效地址计算 EX
- 存储器访问 MEM
- 写回 WB

![](/images/74.png)

我们可以看到一条指令对应许多步骤，上述的每一个步骤都有对应的硬件处理，那么问题来了，是该让指令1执行完所有的步骤后在执行指令2吗？并不是，出于性能的考虑会出现上图的执行方式，也就是当指令1执行了IF后指令2就可以执行了，硬件为了提高效率就会调整代码的结构使其可以并行，所以就会出现代码乱序的现象。

当然硬件也不会随意的排序代码，其会保持其串行执行下的正确性，`number=42;``ready=true;`这两句话在串行的环境下先后顺序并不会影响程序的正确性，重排序不保证在并发条件下语义的正确。关于重排序更多的内容可以参考下面链接。

## 使用volatile关键字
volatile关键字一般是用来解决内存可见性问题，一般用于状态标示变量，要使 volatile 变量提供理想的线程安全，必须同时满足下面两个条件：
- 对变量的写操作不依赖于当前值
- 该变量没有包含在具有其他变量的不变式中

大多数编程情形都会与这两个条件的其中之一冲突，使得 volatile 变量不能像 synchronized 那样普遍适用于实现线程安全。另一方面volatile也会对内存重排序进行限制：

![](/images/75.png)

如果想详细了解可以参考下面的链接。

## 参考链接
1. [深入理解Java内存模型（一）——基础](http://www.infoq.com/cn/articles/java-memory-model-1)
2. Java高并发程序设计
3. Java并发编程实战
4. [Java 理论与实践: 正确使用 Volatile 变量](http://www.ibm.com/developerworks/cn/java/j-jtp06197.html)
5. [Java内存访问重排序的研究](http://tech.meituan.com/java-memory-reordering.html)
