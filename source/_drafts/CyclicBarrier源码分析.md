---
title: CyclicBarrier源码分析
date: 2019-12-30 18:00:00
tags: java
category: J.U.C
---

在前面的文章我们曾经分析过闭锁CountDownLatch的实现，但是你会发现，当state为0是，我们无法在修改state，因为state只能在CountDownLatch初始化时进行设置。

而今天讨论的CyclicBarrier在达到条件后会自动重置。同样的，下面我们以javadoc中代码为示例代码：

```java
 class Solver {
   final int N;
   final float[][] data;
   final CyclicBarrier barrier;

   class Worker implements Runnable {
     int myRow;
     Worker(int row) { myRow = row; }
     public void run() {
       while (!done()) {
         processRow(myRow);

         try {
           barrier.await();
         } catch (InterruptedException ex) {
           return;
         } catch (BrokenBarrierException ex) {
           return;
         }
       }
     }
   }

   public Solver(float[][] matrix) {
     data = matrix;
     N = matrix.length;
     Runnable barrierAction =
       new Runnable() { public void run() { mergeRows(...); }};
     barrier = new CyclicBarrier(N, barrierAction);

     List<Thread> threads = new ArrayList<Thread>(N);
     for (int i = 0; i < N; i++) {
       Thread thread = new Thread(new Worker(i));
       threads.add(thread);
       thread.start();
     }

     // wait until done
     for (Thread thread : threads)
       thread.join();
   }
 }
```

<!--more-->


