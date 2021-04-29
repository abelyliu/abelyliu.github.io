---
title: Spring循环依赖
tags:
  - Spring
category:
  - Java
date: 2021-04-29 11:04:13
---

spring的循环依赖主要理解`singletonObjects`，`earlySingletonObjects`，`singletonFactories`三个缓存是如何使用的。

<!--more-->

![](https://blog.abely.store/image-20210429095224398.png)

![](https://blog.abely.store/image-20210429095643931.png)

![image-20210429101135433](https://blog.abely.store/image-20210429101135433.png)

![](https://blog.abely.store/12.svg)



populateBean有可能依赖其它bean，这一步可能会调用doGetBean获取创建其它bean。



#### 为什么需要singletonFactories

为什么不将new出来的对象放到earlySingletonObjects，而放到singletonFactories，然后获取时，从singletonFactories获取后，再放到earlySingletonObjects。

这里先解释下singletonFactories存放的是创建对象的方法，如果A依赖B，且B需要被AOP增强，那么直接new出来的B对象不能直接被A使用，我们在从singletonFactories获取对象的时候，方法里会判断B需要被增强，这个时候就不直接返回B，而是创建B的代理对象并返回。这样earlySingletonObjects里存放的就是代理对象。如果B不需要被增强，那么singletonFactories里直接获取原始new出来的对象，放到earlySingletonObjects里。



#### earlySingletonObjects和singletonObjects区别

earlySingletonObjects是用来解决循环依赖的，如果没有循环依赖则earlySingletonObjects没有意义，earlySingletonObjects里的对象还没有对属性填充，还没有执行一些回调方法，对象还没有创建完成。singletonObjects里的对象已经创建完成。