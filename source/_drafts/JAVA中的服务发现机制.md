---
title: JAVA中的SPI机制
date: 2019-06-25 23:00:00
tags: java
category: java
---

我们知道如果我们的服务只依赖接口，那么替换实现将变得非常容易。但是我们又如何注入或者选择接口的实现呢？其实JDK已经内置了一种服务发现机制，Java Service Provider Interface 简称SPI。

先从一个简单的例子看起：
```java
package site.abely;

import java.util.Iterator;
import java.util.ServiceLoader;

public class Main {
    public static void main(String[] args) {
        ServiceLoader<DemoInterface> load = ServiceLoader.load(DemoInterface.class);
        Iterator<DemoInterface> iterator = load.iterator();
        while (iterator.hasNext()) {
            System.out.println(iterator.next().say());
        }
    }
}
```

<!--more-->

```java
package site.abely;

public interface DemoInterface {
    String say();
}
```

上述代码的中的main方法就是依赖`DemoInterface`接口进行编程，没有涉及具体实现，但是ServiceLoader.load会返回DemoInterface的实现类。

这个时候你运行程序，你会发现什么不会输出，因为并没有实现类。

这时候我们写两个实现类：

```java
package site.abely;

public class DemoInterfaceImpl1 implements DemoInterface {
    @Override
    public String say() {
        return "DemoInterfaceImpl1";
    }
}
```

```java
package site.abely;

public class DemoInterfaceImpl2 implements DemoInterface {
    @Override
    public String say() {
        return "DemoInterfaceImpl2";
    }
}
```

这个时候运行结果还是一样，并不会发现实现类。

我们在`resources`目录下新建`META-INF/services/site.abely.DemoInterface`文件，内容为`site.abely.DemoInterfaceImpl1`。

这个时候再运行程序则会使用DemoInterfaceImpl1的实现，如果内容为`site.abely.DemoInterfaceImpl2`，则会使用DemoInterfaceImpl2实现。

我们真正在使用这种机制的时候，实现类和`META-INF/services/package.interface`文件都会在第三方jar内，这个我们就可以动态替换实现类的jar或者主动选择达到动态切换实现的功能。
不过值得注意的时使用ServiceLoader，会初始化所有实现类，而且所有实现类必须有无参构造器。

## 应用举例
### JDBC驱动加载

下面代码基于java8，java11略微有点区别

修改pom.xml文件，增加mysql的依赖

```xml
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>8.0.16</version>
</dependency>
```

不知道你是否还记得以前我们连接数据库驱动的时候，使用代码`Class.forName(xxx)`，而现在无需执行此代码，只需要如下显示即可：

```java
public static void main(String[] args) throws SQLException {
	DriverManager.getConnection("","","");
}
```

在java8中，DriverManager中有如下初始化代码：

```java
/**
 * Load the initial JDBC drivers by checking the System property
 * jdbc.properties and then use the {@code ServiceLoader} mechanism
 */
static {
	loadInitialDrivers();
	println("JDBC DriverManager initialized");
}
```

点进去看你会发现下面代码
```java
drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
	public String run() {
		return System.getProperty("jdbc.drivers");
	}
});

....

String[] driversList = drivers.split(":");
for (String aDriver : driversList) {
	Class.forName(aDriver, true,ClassLoader.getSystemClassLoader());
}
```
我们可以看到DriverManager在初始化的时候会加载环境变量`jdbc.drivers`的值，并用调用`Class.forName`注册驱动。

上述省略的代码还有下面这样一段
```java
AccessController.doPrivileged(new PrivilegedAction<Void>() {
	public Void run() {
		ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
		Iterator<Driver> driversIterator = loadedDrivers.iterator();
		while(driversIterator.hasNext()) {
			driversIterator.next();
		}
		return null;
	}
});
```
你会发现这里也会使用ServiceLoader加载所有的驱动并注册。

![mysql的meta-inf](https://www.github.com/abelyliu/pic/raw/master/小书匠/1561472446281.png)

mysql依赖里的配置文件夹下java.sql.Driver里面的值就是com.mysql.cj.jdbc.Driver

这里可能有些同学还是不理解驱动怎么注册的，这里简单介绍下，我们先看下DriverManager怎么返回的驱动

```java
for(DriverInfo aDriver : registeredDrivers) {
	// If the caller does not have permission to load the driver then
	// skip it.
	if(isDriverAllowed(aDriver.driver, callerCL)) {
		try {
			println("    trying " + aDriver.driver.getClass().getName());
			Connection con = aDriver.driver.connect(url, info);
			if (con != null) {
				// Success!
				println("getConnection returning " + aDriver.driver.getClass().getName());
				return (con);
			}
		} catch (SQLException ex) { 
			if (reason == null) {
				reason = ex;
			}
		}

	} else {
		println("    skipping: " + aDriver.getClass().getName());
	}

}
```

我们可以看到所有注册的驱动都在registeredDrivers列表里。
这个时候我们再看下mysql com.mysql.cj.jdbc.Driver的代码：
```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    //
    // Register ourselves with the DriverManager
    //
    static {
        try {
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }

    /**
     * Construct a new driver and register it with DriverManager
     * 
     * @throws SQLException
     *             if a database error occurs.
     */
    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
```
这里的static块就会将当前驱动注册到DriverManager里面去。 

### SLF4J实现加载
前文我们分析过slf4j 1.8版本前是如果绑定日志的，在slf4j1.8版本后，也改用java的spi服务发现机制。这里注意不仅要升级slf4j依赖，也要升级对应匹配的实现类。
我这里使用的是

```xml
<dependency>
	<groupId>org.slf4j</groupId>
	<artifactId>slf4j-api</artifactId>
	<version>2.0.0-alpha0</version>
</dependency>
<dependency>
	<groupId>ch.qos.logback</groupId>
	<artifactId>logback-classic</artifactId>
	<version>1.3.0-alpha4</version>
	<scope>test</scope>
</dependency>
```
我们直接看bind()方法
```java
private final static void bind() {
	try {
		List<SLF4JServiceProvider> providersList = findServiceProviders();
		reportMultipleBindingAmbiguity(providersList);
		if (providersList != null && !providersList.isEmpty()) {
			PROVIDER = providersList.get(0);
			PROVIDER.initialize();
			INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
			reportActualBinding(providersList);
			fixSubstituteLoggers();
			replayEvents();
			// release all resources in SUBST_FACTORY
			SUBST_PROVIDER.getSubstituteLoggerFactory().clear();
		} else {
			INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;
			Util.report("No SLF4J providers were found.");
			Util.report("Defaulting to no-operation (NOP) logger implementation");
			Util.report("See " + NO_PROVIDERS_URL + " for further details.");

			Set<URL> staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
			reportIgnoredStaticLoggerBinders(staticLoggerBinderPathSet);
		}
	} catch (Exception e) {
		failedBinding(e);
		throw new IllegalStateException("Unexpected initialization failure", e);
	}
}
```

这里代码也比较简单，逻辑如下

![代码流程](https://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUAoKdWsV-cBzOkASz9CifxFQdcwRjxplWtFD-wsvifCqu3pdkpeVR9Zr_ELkpGL54eoKlCKD2fJYpMv5830wcd7tAVBkv_sJ7k-PisJ7GrFTgo2QBE6I0UN9XMNP9QKbgJwvAUdfnQv9IQNv1TLlcpl0LhtRFhIf_kdlzWt-MdxBY1wDdVf-pqzJtTkUx9xyVC9RTPSgJd5gGeGSrxid_9qzhod7REVxjxrPCUYftkQWPOzxMY7605o-lQbJ_kQddTjUzRG2DIPbvAPniNb0AI17WK0)

我们再看下如何查询实现类的
```java
private static List<SLF4JServiceProvider> findServiceProviders() {
	ServiceLoader<SLF4JServiceProvider> serviceLoader = ServiceLoader.load(SLF4JServiceProvider.class);
	List<SLF4JServiceProvider> providerList = new ArrayList<SLF4JServiceProvider>();
	for (SLF4JServiceProvider provider : serviceLoader) {
		providerList.add(provider);
	}
	return providerList;
}
```
下面是logback中meta-inf里的内容

![logback配置](https://www.github.com/abelyliu/pic/raw/master/小书匠/1561474509078.png)

这里就是典型的SPI机制。


