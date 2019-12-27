---
title: Java安全模型
date: 2019-12-30 18:00:00
tags: java
category: java
---
如果你阅读过一些框架代码，或则JDK的源码，会经常发现类似下面的代码
```java
//下面代码取自java.sql.DriverManager，初始化jdbc驱动代码
private static void loadInitialDrivers() {
	String drivers;
	try {
		drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
			public String run() {
				return System.getProperty("jdbc.drivers");
			}
		});
	} catch (Exception ex) {
		drivers = null;
	}
	
	//...
}
```
很多方法会被封装在AccessController.doPrivileged中，不知道你是否有过疑问，为什么代码要这样写呢？

<!--more-->

不知道你是否还记得Java Applet程序，它一般运行在支持Java 的Web 浏览器内，从网络加载Java代码执行。因为Java虚拟机需要访问操作系统提供一些服务，如果网络上包含一些恶意代码，则会对本地系统造成破坏。

Java的安全模型的设置就是为了限制代码的运行，而我们在平时的开发中可能并没有感觉到，这是因为默认情况下，虚拟机是不会代码进行限制的。

如果我们把安全功能打开：
```java
public static void main(String[] args) {
    System.setSecurityManager(new SecurityManager());
    System.out.println(System.getProperty("jdbc.drivers"));
}
//output
Exception in thread "main" java.security.AccessControlException: access denied ("java.util.PropertyPermission" "jdbc.drivers" "read")
	at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
	at java.security.AccessController.checkPermission(AccessController.java:884)
	at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
	at java.lang.SecurityManager.checkPropertyAccess(SecurityManager.java:1294)
	at java.lang.System.getProperty(System.java:717)
	at com.example.demo.DemoApplication.main(DemoApplication.java:9)
```

可以看到，如果我们把安全功能打开，则一个获取环境变量的方法也无法执行。可以点开getProperty方法，查看实现逻辑
```java
public static String getProperty(String key) {
    checkKey(key);
    //获取安全配置类
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        //如果不为空，则检查当前操作是否有权限
        sm.checkPropertyAccess(key);
    }
    //真正获取环境配置的代码
    return props.getProperty(key);
}
```

在JDK源码里，很多和系统资源交互的代码都加入了权限检查。既然有权限检查，就会有配置权限的地方，我们可以在启动时加入-Djava.security.policy=my.policy (文件的路径)，默认的权限配置文件为`$JAVA_HOME/jre/lib/security/java.policy `。

- 没有配置的权限表示没有。
- 只能配置有什么权限，不能配置禁止做什么。
- 同一种权限可多次配置，取并集。
- 统一资源的多种权限可用逗号分割。

刚刚我们说过，一开始JAVA的一个主要方向就是运行Applet，为了安全性，设计了SecurityManager，而默认情况下，本地本地代码是没有安全限制的，那么JVM又是如何区分本地代码和从网络代码，分别设置不同的权限呢?

答案就在classloader，本地代码和网络代码是从不同的classloader进行加载的，对不同的classloader有着不同的安全限制(安全模型从一直在演化，如有兴趣，可以参考尾部链接)。

![安全模型](https://abelyliu.oss-cn-shanghai.aliyuncs.com/blog/4190914-2e86c9c0fdc174a3.webp)

我们现在理解了SecurityManager的设置初衷和实现原理，但是有个问题还没有阐述，`AccessController.doPrivileged`方法的作用是什么？回到刚才的例子，我们把代码稍加修改
```java
public static void main(String[] args) {
    System.setSecurityManager(new SecurityManager());
    String drivers;
    try {
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }
    System.out.println(drivers);
}
```

我们会发现，虽然我们开启了权限检查，我们还是能正常的执行代码，这是因为`AccessController.doPrivileged`能用来临时提升权限级别。

我们设计SecurityManager的目的就是提升安全性，为什么还需要提供这样的功能呢？你可以简单思考下。

考虑一下这种场景，我们设计了一个组件，第三方可以使用我们的代码，但是第三方代码可能使用不同的classloader，安全域也不一样，我们自己会定义一些安全检查，判断第三方是否可以调用，如果可以，那么我们就可以使用特权模式，即使第三方代码没有权限也可以获得相应的服务。

比如上面的java.sql.DriverManager，即使你的没有读取环境变量的权限，但依然不会影响你加载JDBC驱动。

参考链接：
1. https://www.jianshu.com/p/0ce92da36eba
2. https://www.cnblogs.com/iceAeterNa/p/4876503.html
3. https://www.cnblogs.com/MyStringIsNotNull/p/8268351.html
4. https://stackoverflow.com/questions/2233761/when-should-accesscontroller-doprivileged-be-used/2234080#2234080



