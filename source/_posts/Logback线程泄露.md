# Logback线程泄露

最近发现一个服务出现了线程泄露的问题，线程数从几百缓慢的增加到一千多。获取线程栈，发现是日志线程出现泄露。

通过arthas定位线程栈如下，发现是配置刷新事件触发了日志的重加载

![image-20210408215440613](http://blog.abely.store/1617890080686-image-20210408215440613.png)

<!--more-->

观察KafkaAppender类，发现类在top时是有进行销毁操作的。

![image-20210408220013598](http://blog.abely.store/1617890413640-image-20210408220013598.png)

要理解在销毁时为什么没有释放线程，可以阅读logback重加载的代码，看看销毁的逻辑是什么。如果阅读代码，发现最后都会进入下面这段：

![image-20210408220301049](http://blog.abely.store/1617890581091-image-20210408220301049.png)

这里appenderList实际上是和logger挂钩的，也就是如果一个appender没有被logger引用，在销毁时并不会调用其stop方法。

![image-20210408220721156](http://blog.abely.store/1617890841218-image-20210408220721156.png)

我们的项目恰好存在这种代码，KAFKA_BACKUP没有被直接引用，导致回收时无法被有效回收。我们可以手动注册对象如下：

![image-20210408221124917](http://blog.abely.store/1617891084963-image-20210408221124917.png)

参考链接

- [Logback的深度使用经验和最佳实践 | Zollty's Blog](http://blog.zollty.com/b/archive/the-depth-experience-and-best-practice-of-logback.html)

