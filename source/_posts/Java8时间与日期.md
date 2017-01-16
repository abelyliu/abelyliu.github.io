---
title: Java8日期与时间
tags: Java
category: Java
date: 2017-01-04 10:14:45
---

我们知道Java8一个比较大的改动就是引入了新的表示日期和时间的API，还是很有必要抽点时间了解一下基本的用法。

## Java8之前的问题
在讨论Java8的API之前，我们先看看原来API的问题。我们知道Java8之前只有java.util.Date和java.util.Calendar来表示和处理时间，它们的一些常见的问题如下：
1. Date返回getMonth返回值为0~11，分别对应1~12月；
2. Calendar处理夏令时可能会引起问题；
3. 这两个类以及DateTimeFormat都不是线程安全的；
4. 缺乏很多时间概念，如无时区日期，时间，间隔等等；
5. 缺乏很多常用的日期操作，如计算两个日期相差天数

JSR 310就是这样应运而生，Java8的API也是依据此规范而来。
>This JSR will tackle the problem of a complete date and time model, including dates and times (with and without time zones), durations and time periods, intervals, formatting and parsing.

<!--more-->

## 概览
包名|功能
:---:|:---:
java.time|表示日期、时间、瞬时和期间的公共类
java.time.chrono|非ISO日历系统的API
java.time.format|格式化类
java.time.temporal|使用fields,units,adjusters访问日期和时间
java.time.zone|支持时区及其规则

## 常用类

类|功能
:---:|:---:
Instant|自1970年1月1日以来的一个时间点，以纳秒为单位
Duration|时长，以纳秒为单位
Calendrical|与底层API有关
DateTimeFields|存储域——值对的一个映射，不要求一致
DayOfWeek|星期中的一天
LocalDate|本地日期(年月日)，没有调整
LocalTime|本地时间(时分秒)，没有调整
LocalDateTime|LocalDate和LocalTime的组合
MonthDay|月日
OffsetTime|带时区的偏移时间，不包括日期或时区值
OffsetDateTime|带时区偏移的日期时间，不包括时区值
Period|时间长度的描述，如两个月零三天
ZonedDateTime|带时区和偏移的日期时间
Year|年
YearMonth|年月
ZoneOffset|从UTC开始的时间偏移(时分秒)
ZoneId|定义一个时区，如Canada/Eastern，及其转换规则

如果对哪个类不是特别理解，打开IDE，简单的操作一下即可。

## 简单用例

#### 查看当天日期
```java
System.out.println(LocalDate.now());
System.out.println(LocalTime.now());
System.out.println(LocalDateTime.now());

output:
2017-01-03
16:24:11.520
2017-01-03T16:24:11.520
```

#### 格式化日期和时间
下面是DateTimeFormatter类提供的格式化样式，只需简单了解一下即可，需要时可以查询[DateTimeFormatter文档](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html)。
还有一个值得注意的是，模式中字母的重复次数对应这字符串预期长度。例如"MMM"对应"Jan"，""MMMM"对应"January"。你可能会好奇为什么"MMMM"为什么不是对应"Janu"，这点可以参考上面的文档，如何对应在文档中也有说明。

符号|含义|显示形式|例子
:---:|:---:|:---:|:---:
G|公元|文本|AD;Anno Domini
y|公元年|年份|2004;04
u|公元年|年份|公元年和y相同，公元前3年，y返回3,u返回-2
D|一年中的第几天|数字|189
M/L|月|数字/文本|7;07;Jul;July;J
d|一月中的第几天|数字|10
Q/q|一年中的第几季|数字/文本|3;03;Q3,3rd quarter
Y|年|年份|1996;96
w|一年中的某一周|数字|27
W|一个月中的某一周|数字|4
e/c|星期中的本地天数|数字/文本|2;02;Tue;Tuesday;T
E|星期中的第几天|文本|Tue;Tuesday;T
F|某月中星期几出现的序数(第几个)|数字|3
a|AM/PM表示|文本|PM
h|12小时制的小时数(1~12)|数字|12
K|12小时制的小时数(0~11)|数字|0
k|24小时制的小时数(1~24)|数字|24
H|24小时制的小时数(0~23)|数字|0
m|小时内第几分|数字|30
s|一分内第几秒|数字|55
S|秒的小数(毫秒)|小数|978
A|千分之一天|数字|1234
n|纳秒|数字|987654321
N|十亿分之一(纳)天|数字|1234000000
V|时区ID值|文本|America/Los_Angeles;Z;-08:30
z|时区名|文本|Pacific Standard Time;PST
X|时区偏移Z|Offset-X|Z;-08;-0830;-08:30;-083015;-08:30:15
x|时区偏移|Offset-x|+0000;-08;-0830;-08:30;-083015;-08:30:15
Z|时区的GMT偏移|Offset-Z|+0000;-0800;-08:00
O|本地时区偏移|Offset-O|GMT+8；GMT+08:00;UTC-08:00
p|填充下一个|填充符|1

```java 
DateTimeFormatter df = DateTimeFormatter.ofPattern("yy/LL/dd");
System.out.println(df.format(LocalDate.now()));
System.out.println(LocalDate.parse("17/01/04",df));

output:
17/01/04
2017-01-04
```

#### 日期和时间戳转换
```java 
System.out.println(ZonedDateTime.now());
System.out.println(ZonedDateTime.now().toInstant().getEpochSecond());
Instant instant = Instant.ofEpochSecond(1483492172L);
ZonedDateTime zonedDateTime = ZonedDateTime.ofInstant(instant, ZoneId.of("Asia/Shanghai"));
System.out.println(zonedDateTime);

output:
2017-01-04T09:13:39.624+08:00[Asia/Shanghai]
1483492419
2017-01-04T09:09:32+08:00[Asia/Shanghai]
```

#### 字符串解析为日期
```java 
//默认解析格式为ISO 8601
LocalDate localDate = LocalDate.parse("2016-01-04");
System.out.println(localDate);

DateTimeFormatter pattern = DateTimeFormatter.ofPattern("dd LLL yyyy");
System.out.println(pattern.format(LocalDate.now()));

//注意LocalDate是和本地相关的，所以要用一月，上面就是看看本地输出格式
DateTimeFormatter df = DateTimeFormatter.ofPattern("dd MMM yyyy");
System.out.println(LocalDate.parse("04 一月 2011", df));

output:
2016-01-04
04 一月 2017
2011-01-04
```

#### 日期间的计算
```java 
LocalDate last = LocalDate.of(2016, 1, 1);
LocalDate now = LocalDate.now();
Period diff = Period.between(last, now);
System.out.println(diff);

//今天是2017/1/4
Period p = Period.ofDays(1);
System.out.println(now.plus(p));
output:
P1Y3D
2017-01-05
```
## 参考链接
1. [What's wrong with Java Date & Time API?](http://stackoverflow.com/questions/1969442/whats-wrong-with-java-date-time-api)
2. [ISO_8601](https://zh.wikipedia.org/zh-cn/ISO_8601)
