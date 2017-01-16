---
title: Spring 2.5与JDK8的一个问题
tags: Spring
category: Java
date: 2017-01-09 10:50:28
---

最近公司的服务准备部署到亚马逊云上，对项目做了许多修改，其中有一项是把JDK从7升级到8。在升级过程中有部分项目出现了部署失败的情况，于是又展开了一场调试之旅。

首先需要复现错误，错误有两种：
1. SEVERE: Context [/******] startup failed due to previous errors 
2. Line: 209 - com/opensymphony/xwork2/spring/SpringObjectFactory.java:209:-1

第一个错误提示信息太少，所以从第二个信息开始调试。关于`Line: 209 - com/opensymphony/xwork2/spring/SpringObjectFactory.java:209:-1`这个问题网上解决办法有两个
1. lib中多导入包的大原因：去掉struts2-spring-plugin-2.1.8包即可，因为没有用到spring。
2. 还有的原因是用spring了，却没加监听器，在web.xml里面加上
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext*.xml</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
这两个和我的问题并没有什么关联(这两种方案明显是解决struts2和spring集成的问题)，我的问题很明显是版本导致的，在JDK7下可以正确运行，而JDK8不行，网上也没查询到其它的解决方案，只能暂时放弃这种方式查询。
<!--more-->
这时候我去搜寻jdk版本和spring的版本之间冲突的问题，确实也查到了[Spring4以下的版本不支持JDK8](https://jira.spring.io/browse/SPR-11899)，官方建议用Spring4以上使用JDK8。可以看到官方给出的解决方案是Won'Fix，也就是不会修复了。

但现在的问题是为什么有的项目会部署成功，而只有两三个不成功呢？如果不兼容应该所有的都不兼容才对。

这时候我就对比了问题项目和能部署的项目，发现问题项目使用了注解，我很自然的想到可能是注解导致的问题，但是能部署的项目中有一个也是使用了注解，我去询问告诉升级的人，他告诉我那个项目能正确部署(其实不能)。

这时候我就懵逼了。

不能靠猜想了，于是修改了一下问题项目的log4j的配置，打印struts2和spring的调试日志。

下面是最关键的一句
```sh 
[org.springframework.context.annotation.AnnotationConfigBeanDefinitionParser] are only available on JDK 1.5 and higher
```
查询得知AnnotationConfigBeanDefinitionParser在检测java版本时只会检查到JDK7，JDK8它无法识别。

为什么只有两个项目出现错误呢，因为只有这两个项目在xml中使用了`<context:component-scan base-package="xxx"/>`这句话，换句话说它会使用AnnotationConfigBeanDefinitionParser这个类。我很自然的想到了上面那个也使用注解，却被人告诉我能正确运行的项目，自己测试了一下果然不能正确运行。

现在问题找到了，修复就比较容易了。有三种办法可以修复这个问题：
1. 不使用注解，全部切换为xml配置bean，测试了一下，没什么问题
2. 替换spring中JdkVersion类，JdkVersion不依赖其它类，所以比较好替换，新建一个JdkVersion类，代码如下，编译成class文件替换jar中对应的class文件即可，测试了一下，没什么问题
```java
package org.springframework.core;

/**
 * Created by abely 
 */
public abstract class JdkVersion {
    public static final int JAVA_13 = 0;
    public static final int JAVA_14 = 1;
    public static final int JAVA_15 = 2;
    public static final int JAVA_16 = 3;
    public static final int JAVA_17 = 4;
    //for jre 1.8
    public static final int JAVA_18 = 5;
    private static final String javaVersion = System
            .getProperty("java.version");
    private static final int majorJavaVersion;
    public static String getJavaVersion() {
        return javaVersion;
    }
    public static int getMajorJavaVersion() {
        return majorJavaVersion;
    }
    public static boolean isAtLeastJava14() {
        return true;
    }
    public static boolean isAtLeastJava15() {
        return getMajorJavaVersion() >= 2;
    }
    public static boolean isAtLeastJava16() {
        return getMajorJavaVersion() >= 3;
    }
    static {
        //for jre 1.8
        if (javaVersion.indexOf("1.8.") != -1) {
            majorJavaVersion = 5;
        }else if (javaVersion.indexOf("1.7.") != -1) {
            majorJavaVersion = 4;
        } else if (javaVersion.indexOf("1.6.") != -1) {
            majorJavaVersion = 3;
        } else if (javaVersion.indexOf("1.5.") != -1) {
            majorJavaVersion = 2;
        } else {
            majorJavaVersion = 1;
        }
    }
}
```
3. 替换spring版本

## 参考链接
1. [spring (2.5, 3.2) 在 jre 1.8下的fix](https://my.oschina.net/aruan/blog/210284)
2. [Spring/Java error: namespace element 'annotation-config' … on JDK 1.5 and higher](http://stackoverflow.com/questions/23813369/spring-java-error-namespace-element-annotation-config-on-jdk-1-5-and-high)