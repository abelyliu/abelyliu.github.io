---
title: slf4j选择默认日志实现流程
date: 2019-06-23 00:30:00
tags: 日志
category: 源码
---

我们知道slf4j使用了门面模式，封装了底层的日志实现。今天我们就来看下如slf4j是如何和具体的实现直接进行绑定的。

新建一个项目，依赖如下：
```gradle
dependencies {
    compile group: 'org.slf4j', name: 'slf4j-api', version: '1.7.26'
    compile group: 'ch.qos.logback', name: 'logback-classic', version: '1.2.3'
    compile group: 'org.apache.logging.log4j', name: 'log4j-slf4j-impl', version: '2.11.2'
    compile group: 'org.apache.logging.log4j', name: 'log4j-core', version: '2.11.2'
}
```

主方法如下
```java
public class Main {

    private static Logger logger = LoggerFactory.getLogger(Integer.class);

    public static void main(String[] args) {
        logger.error("hello world");
    }
	
}
//运行结果 
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/C:/Users/abely/.gradle/caches/modules-2/files-2.1/ch.qos.logback/logback-classic/1.2.3/7c4f3c474fb2c041d8028740440937705ebb473a/logback-classic-1.2.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/C:/Users/abely/.gradle/caches/modules-2/files-2.1/org.apache.logging.log4j/log4j-slf4j-impl/2.11.2/4d44e4edc4a7fb39f09b95b09f560a15976fa1ba/log4j-slf4j-impl-2.11.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [ch.qos.logback.classic.util.ContextSelectorStaticBinder]
SLF4J: Detected logger name mismatch. Given name: "java.lang.Integer"; computed name: "Main".
SLF4J: See http://www.slf4j.org/codes.html#loggerNameMismatch for an explanation
23:45:00.039 [main] ERROR java.lang.Integer - hello world
```

进入获取方法：
```java
public static Logger getLogger(Class<?> clazz) {
	Logger logger = getLogger(clazz.getName());
	if (DETECT_LOGGER_NAME_MISMATCH) {
		Class<?> autoComputedCallingClass = Util.getCallingClass();
		if (autoComputedCallingClass != null && nonMatchingClasses(clazz, autoComputedCallingClass)) {
			Util.report(String.format("Detected logger name mismatch. Given name: \"%s\"; computed name: \"%s\".", logger.getName(),
							autoComputedCallingClass.getName()));
			Util.report("See " + LOGGER_NAME_MISMATCH_URL + " for an explanation");
		}
	}
	return logger;
}
```
上面的`if (DETECT_LOGGER_NAME_MISMATCH)`语句是处理当调用log方法的对象和参数中clazz类型不一致时，是否告警的配置。默认关闭，如果想要开启，可以配置`-Dslf4j.detectLoggerNameMismatch=true`，上面main方法就配置了此参数，所以会有`SLF4J: Detected logger name mismatch. Given name: "java.lang.Integer"; computed name: "Main".`



接下来再看下getLogger方法：
```java
public static Logger getLogger(String name) {
	ILoggerFactory iLoggerFactory = getILoggerFactory();
	return iLoggerFactory.getLogger(name);
}
```
getLogger分为两步，第一步获取日志工厂，第二步创建日志。slf4j绑定具体日志就是在获取日志工厂这一步，这里返回logback的日志工厂。我们具体看下是如何绑定的。

```java
public static ILoggerFactory getILoggerFactory() {
	if (INITIALIZATION_STATE == UNINITIALIZED) {
		synchronized (LoggerFactory.class) {
			if (INITIALIZATION_STATE == UNINITIALIZED) {
				INITIALIZATION_STATE = ONGOING_INITIALIZATION;
				performInitialization();
			}
		}
	}
	switch (INITIALIZATION_STATE) {
	case SUCCESSFUL_INITIALIZATION:
		return StaticLoggerBinder.getSingleton().getLoggerFactory();
	case NOP_FALLBACK_INITIALIZATION:
		return NOP_FALLBACK_FACTORY;
	case FAILED_INITIALIZATION:
		throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
	case ONGOING_INITIALIZATION:
		// support re-entrant behavior.
		// See also http://jira.qos.ch/browse/SLF4J-97
		return SUBST_FACTORY;
	}
	throw new IllegalStateException("Unreachable code");
}
```
这里的代码比较简单，判断是否初始化过，如果没有初始化则进行初始化，否则返回日志创建工厂。

这里值得注意的是初始化状态有五种，从代码可以看出，某些情况下未初始化完成，我们也能获取工厂(NOP_FALLBACK_FACTORY,SUBST_FACTORY)。至于为什么状态不是SUCCESSFUL_INITIALIZATION也能获取工厂我们继续往下看`performInitialization`方法。

```java
private final static void performInitialization() {
	bind();
	if (INITIALIZATION_STATE == SUCCESSFUL_INITIALIZATION) {
		versionSanityCheck();
	}
}
```
我们可以看到，这个方法做了两件事，绑定slf4j的日志工厂，检查工厂是否正确创建。这个里我们先看下`versionSanityCheck`方法

```java
private final static void versionSanityCheck() {
	try {
		String requested = StaticLoggerBinder.REQUESTED_API_VERSION;

		boolean match = false;
		for (String aAPI_COMPATIBILITY_LIST : API_COMPATIBILITY_LIST) {
			if (requested.startsWith(aAPI_COMPATIBILITY_LIST)) {
				match = true;
			}
		}
		if (!match) {
			Util.report("The requested version " + requested + " by your slf4j binding is not compatible with "
							+ Arrays.asList(API_COMPATIBILITY_LIST).toString());
			Util.report("See " + VERSION_MISMATCH + " for further details.");
		}
	} catch (java.lang.NoSuchFieldError nsfe) {
		// given our large user base and SLF4J's commitment to backward
		// compatibility, we cannot cry here. Only for implementations
		// which willingly declare a REQUESTED_API_VERSION field do we
		// emit compatibility warnings.
	} catch (Throwable e) {
		// we should never reach here
		Util.report("Unexpected problem occured during version sanity check", e);
	}
}
```
这里值得注意的一点是StaticLoggerBinder类并不在slf4j的包里，而是在logback和log4j-slf4j-impl依赖里。

![enter description here](https://www.github.com/abelyliu/pic/raw/master/小书匠/1.png)

这里slf4j取到实现类的StaticLoggerBinder.REQUESTED_API_VERSION值，判断当前slf4j版本是否兼容现有实现类，如果不兼容则告警，如果实现类没有StaticLoggerBinder.REQUESTED_API_VERSION这个字段，则默认为不需要兼容性检查。

不知道你会不会有这样一个疑问，如果StaticLoggerBinder.REQUESTED_API_VERSION是在实现类中，那么slf4j在编译时要依赖实现类，否则不可能编译通过。

如果你去查询slf4j的官方源代码，其实是有默认实现的，地址为https://github.com/qos-ch/slf4j/tree/1.7-maintenance/slf4j-api/src/main/java/org/slf4j

下面是slf4j里的默认实现

```java
package org.slf4j.impl;
import org.slf4j.ILoggerFactory;

public class StaticLoggerBinder {

    private static final StaticLoggerBinder SINGLETON = new StaticLoggerBinder();

    public static final StaticLoggerBinder getSingleton() {
        return SINGLETON;
    }

    public static String REQUESTED_API_VERSION = "1.6.99"; 

    private StaticLoggerBinder() {
        throw new UnsupportedOperationException("This code should have never made it into slf4j-api.jar");
    }

    public ILoggerFactory getLoggerFactory() {
        throw new UnsupportedOperationException("This code should never make it into slf4j-api.jar");
    }

    public String getLoggerFactoryClassStr() {
        throw new UnsupportedOperationException("This code should never make it into slf4j-api.jar");
    }
}
```

slf4j的pom.xml文件里增加下面的配置

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-antrun-plugin</artifactId>
	<executions>
	  <execution>
		<phase>process-classes</phase>
		<goals>
		 <goal>run</goal>
		</goals>
	  </execution>
	</executions>
	<configuration>
	  <tasks>
		<echo>Removing slf4j-api's dummy StaticLoggerBinder and StaticMarkerBinder</echo>
		<delete dir="target/classes/org/slf4j/impl"/>
	  </tasks>
	</configuration>
</plugin>
```
我们可以看到，在打包的时候会自动移除`org.slf4j.impl`下的源代码。

这个时候我们回过头来看下`bind`方法是如何绑定具体的实现。

```java
private final static void bind() {
	try {
		Set<URL> staticLoggerBinderPathSet = null;
		// skip check under android, see also
		// http://jira.qos.ch/browse/SLF4J-328
		if (!isAndroid()) {
			staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
			reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
		}
		// the next line does the binding
		StaticLoggerBinder.getSingleton();
		INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
		reportActualBinding(staticLoggerBinderPathSet);
		fixSubstituteLoggers();
		replayEvents();
		// release all resources in SUBST_FACTORY
		SUBST_FACTORY.clear();
	} catch (NoClassDefFoundError ncde) {
		String msg = ncde.getMessage();
		if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {
			INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;
			Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");
			Util.report("Defaulting to no-operation (NOP) logger implementation");
			Util.report("See " + NO_STATICLOGGERBINDER_URL + " for further details.");
		} else {
			failedBinding(ncde);
			throw ncde;
		}
	} catch (java.lang.NoSuchMethodError nsme) {
		String msg = nsme.getMessage();
		if (msg != null && msg.contains("org.slf4j.impl.StaticLoggerBinder.getSingleton()")) {
			INITIALIZATION_STATE = FAILED_INITIALIZATION;
			Util.report("slf4j-api 1.6.x (or later) is incompatible with this binding.");
			Util.report("Your binding is version 1.5.5 or earlier.");
			Util.report("Upgrade your binding to version 1.6.x.");
		}
		throw nsme;
	} catch (Exception e) {
		failedBinding(e);
		throw new IllegalStateException("Unexpected initialization failure", e);
	}
}
```

这里可以看到，try块内是正常绑定，catch是出现了问题，绑定时失败。这里值得注意的是`if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg))`这句代表着StaticLoggerBinder类没有找到，也就是当前类路径下没有任何一个slf4j的实现，这里会返回`NOP_FALLBACK_INITIALIZATION`，代表则初始化一个丢弃一切日志的日志工厂，其它异常皆返回创建工厂失败。

我们现在分析一下正常的情况的日志工厂加载，这里不讨论Android情况。

![日志工厂加载流程](https://www.plantuml.com/plantuml/svg/ZP4zJy9G58PtVamd9rWG4oSJ4uk3asa04LC-qjOTY4e_AD5082BI6AqI4vgwG4Cj_9bxphMJVy4BDLqqqUtcvddUzqrEMbO4IJalYuaaMZPgovmeo79DK4w9PrIb8YUB9rjdNAbS4pbU4PHIRgzQB1QaJAcIBqZSXgPlVYgHXEScZaSuqk1fIBpNdp33Fj_ReDwYRiEDlbRKUtWneDsd8mkdZtHu0NCREiqeyCDPqi296LIlGRUeFiDwWmJc_njHwuBfac15UvDrnUMS15rmnhJZVYzth_Z339yztjtUoUuV785w1_w2eU8cYsd01CsCdMXbs3AnxqRmtod4cx8t3cnO3SZ28Fuiyd7oW8P5l7hOCDbOLiO-OolQ_ajws6h7KDZIiRTC9TA5IdvzoIy0)

这里只讨论slf4j流程，所以`StaticLoggerBinder.getSingleton()`里具体怎么初始化的就不讨论了，这需要结合具体的实现。

这里值得注意的有两点
1. `SUBST_FACTORY`的作用
2. 如果有多个`StaticLoggerBinder`会初始化哪个 

我们先来看第一个问题，你是否还记得在获取日志工厂时下面的代码：
```java
public static ILoggerFactory getILoggerFactory() {
	if (INITIALIZATION_STATE == UNINITIALIZED) {
		synchronized (LoggerFactory.class) {
			if (INITIALIZATION_STATE == UNINITIALIZED) {
				INITIALIZATION_STATE = ONGOING_INITIALIZATION;
				performInitialization();
			}
		}
	}
	switch (INITIALIZATION_STATE) {
	case SUCCESSFUL_INITIALIZATION:
		return StaticLoggerBinder.getSingleton().getLoggerFactory();
	case NOP_FALLBACK_INITIALIZATION:
		return NOP_FALLBACK_FACTORY;
	case FAILED_INITIALIZATION:
		throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
	case ONGOING_INITIALIZATION:
		// support re-entrant behavior.
		// See also http://jira.qos.ch/browse/SLF4J-97
		return SUBST_FACTORY;
	}
```

我们可以看到`See also http://jira.qos.ch/browse/SLF4J-97`这样一个bug。这个bug就是在初始化日志工厂的过程中，初始化的代码也依赖了slf4j的日志工厂，这样代码就无法进行下去了，这里slf4j会返回一个SUBST_FACTORY，当真正的日志工厂初始化完成后，将SUBST_FACTORY里的所有logger委托给真正的日志实现。也就是上述流程图的最后三步。这样就支持了日志工厂的重入。

第二个问题和类加载器有关
如果你观察过idea的运行程序，第一行你会发现启动参数`-classpath`，这里面的jar的顺序决定了类加载器先加载哪个。
```xml
compile group: 'ch.qos.logback', name: 'logback-classic', version: '1.2.3'
compile group: 'org.apache.logging.log4j', name: 'log4j-slf4j-impl', version: '2.11.2'
```
如果你调整下这两个依赖的顺序，观察一下idea的-classpath启动参数，你会发现在前面的会被slf4j当作实现类加载。(注意刷新依赖)

一句话总结就是，查找classpath下第一个org.slf4j.impl.StaticLoggerBinder.class，这个就代表要选择的日志实现。

