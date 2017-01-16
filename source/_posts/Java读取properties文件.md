---
title: Java读取properties文件
date: 2016-11-23 10:12:58
tags: Java
category: Java
---
读取properties文件的三种方式
<!--more-->

## 使用ResourceBundle
我么知道ResourceBundle是用来做国际化的，国际化的配置文件也是properties格式的，所以我们可以直接使用ResourceBundle读取配置文件。
这种方式还有一个好处就是会自动缓存，也就是值在一开始读的时候就固定了，在程序运行过程中，值修改了也无法实时更新。
```java
ResourceBundle rb = ResourceBundle.getBundle("config");
String userName = rb.getString("username");
String password = rb.getString("password");
String ip = rb.getString("proxyhost");
String port = rb.getString("port");
```
值得注意的是文件后缀名不用加上，而且文件默认从classpath下查找。

## 使用File IO
```java
Properties prop = new Properties();  
prop.load(new FileInputStream("config/config.properties"));  
String userName = prop.getProperty("username");  
String password = prop.getProperty("password");  
String ip = prop.getProperty("proxyhost"); 
String port = prop.getProperty("port"); 
```
因为我是使用maven构建的项目，所以文件读取的使用的相对路径，而上一种方法会直接从classpath读取。

## 使用Class Loader
```java
Properties prop = new Properties();
prop.load(Main.class.getResourceAsStream("config.properties"));
```
这种方法和上一种的方法区别在于加载文件的方式不同，这种方式读取文件是从classpath路径下读取文件。

