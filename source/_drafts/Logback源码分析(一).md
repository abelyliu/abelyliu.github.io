---
title: Logback源码分析(一)
date: 2019-12-30 18:00:00
tags: java
category: logback,日志
---

前文我们分析过slf4j是如何绑定日志实现类的，我们今天来分析下logback日志类的具体实现。实现我们分两部分分析，第一部分为如何初始化，第二部分为日志打印流程。

我和slf4j一样，还是从获取Log代码开始分析，这里会跳过slf4j的逻辑，如有需要，可以翻阅前文查看。
```java
private static Log logger = LogFactory.getLog(this.getClass());
```

我们以前分析过，绑定具体日志实现类的行为发生在getILoggerFactory里，返回的是`return StaticLoggerBinder.getSingleton().getLoggerFactory()`。
```java
public static Logger getLogger(String name) {
    ILoggerFactory iLoggerFactory = getILoggerFactory();
    return iLoggerFactory.getLogger(name);
}
```

我们就从logback的getLoggerFactory方法入手，先看是如何获取Logger。

<!--more-->

```java
public ILoggerFactory getLoggerFactory() {
    //如果日志还没有初始化，返回默认的日志上下文
    if (!initialized) {
        return defaultLoggerContext;
    }

    if (contextSelectorBinder.getContextSelector() == null) {
        throw new IllegalStateException("contextSelector cannot be null. See also " + NULL_CS_URL);
    }
    //获取去日志上下文
    return contextSelectorBinder.getContextSelector().getLoggerContext();
}

```