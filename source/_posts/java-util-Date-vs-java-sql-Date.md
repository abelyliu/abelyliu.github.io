---
title: java.util.Date vs java.sql.Date
tags: Java
category: Java
date: 2017-01-03 13:53:19
---

在使用JDBC访问数据库时，当我遇到日期类型时会使用java.util.Date，当时刚刚接触数据库，如何使用JDBC就把我搞得焦头烂额，也并没有关心这个问题，只是按照别人的代码抄写。今天在[stackoverflow](http://stackoverflow.com/questions/2305973/java-util-date-vs-java-sql-date)看到关于这两个之间区别的讨论，特此学习一下。

## 区别与联系
我们知道每种数据库都有自己的数据类型，但就时间类型而言一般至少有三种类型：`date`,`time`和`timestamp`类型。查阅JDK的文档我们发现java.sql包下面也有三种对应的类型。
1. java.sql.Date 只包含年月日
2. java.sql.Time 只包含时分秒
3. java.sql.Timestamp 包含年月日时分秒

上面的三种类型都是继承自java.util.Date，java.util.Date和java.sql.Timestamp的区别在于Date精度为毫秒，Timestamp精度为纳秒而且不包含时区。

<!--more-->

回答者也提到了不建议用上面三种类型的任何一种，建议使用时间戳，然后转换成任何想要的格式。

为什么回答者不建议使用上面三种类型呢？

因为对上面三种类型如果不够理解那么容易写出有问题的代码，比如使用java.sql.Date时并不会包含时区信息，使用java.sql.Time时并不会带有年月日的信息，当你读取数据时会发现出现问题，如果使用时间戳则不会出现上述问题。

#### 时间戳
这里简单介绍一下时间戳，**时间戳是指格林威治时间1970年01月01日00时00分00秒(北京时间1970年01月01日08时00分00秒)起至现在的总秒数。**

这里要注意的是**时间戳是和时区无关的**，比如时间戳`1`就表示格林威治时间1970年01月01日00时00分01秒或者北京时间1970年01月01日08时00分01秒。

![](/images/52.jpg)

还有一点是在java.util.Date中getTime()返回的时间戳是格林威治时间1970年01月01日00时00分00秒(北京时间1970年01月01日08时00分00秒)起至现在的*总毫秒数*，精度不同用法类似。

## 使用java.sql.*
我们知道JDBC中有一个PreparedStatement用来预编译SQL语句，如果我们使用PreparedStatement可能就需要上述三种类型了，PreparedStatement中有如下三个方法(还有[重载的方法](http://docs.oracle.com/javase/8/docs/api/index.html?java/sql/PreparedStatement.html))：
1. void	setDate(int parameterIndex, Date x)
2. void	setTime(int parameterIndex, Time x)
3. void	setTimestamp(int parameterIndex, Timestamp x)

其中的参数类型就是上述类型。下面是一个例子：

```java
package site.abely;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.util.Date;

public class Dao {
    public static void main(String[] args) {
        try {
            Class.forName("org.postgresql.Driver").newInstance();
            String url = "jdbc:postgresql://localhost:5432/test";
            Connection con = DriverManager.getConnection(url, "postgres", "admin");
            PreparedStatement st = con.prepareStatement("INSERT INTO date_time (\"date\", \"time\", \"timestamp\") VALUES (?,?,?)");
            st.setDate(1, new java.sql.Date(new Date().getTime()));
            st.setTime(2, new java.sql.Time(new Date().getTime()));
            st.setTimestamp(3, new java.sql.Timestamp(new Date().getTime()));
            st.execute();
            st.close();
        } catch (Exception ee) {
            System.out.println(ee.getMessage());
        }

    }
}
```

数据库中插入结果如下：
![](/images/53.png)

如果在Java8的环境中可以考虑java.time包中新的日期API, 在Java8之前如果不使用上述类型存储对应的类型如time需要自定义Date的解析格式，如果存储date和timestamp则可以直接用java.util.Date。

```java
String sql ="INSERT INTO date_time (\"date\", \"time\", \"timestamp\") VALUES ('"+new Date()+"','"+ new java.sql.Time(new Date().getTime())+"','"+new Date()+"')";
```

![](/images/54.png)

我们可以看到如果使用java.util.Date替代java.sql.Timestamp会丢失精度，java.sql.Date不会有影响。回答者之所以建议使用时间戳也是因为时间戳不会丢失信息，使用起来也相对方便。







## 参考连接
1. [java.util.Date vs java.sql.Date](http://stackoverflow.com/questions/2305973/java-util-date-vs-java-sql-date)
2. [百度百科](http://baike.baidu.com/view/354827.htm)
3. [程序员的日常：时间戳和时区的故事](http://codingpy.com/article/programmer-daily-story-about-timestamp-and-timezone/)

