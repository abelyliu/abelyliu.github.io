---
title: SQL入门
date: 2016-11-10 16:21:06
tags: SQL
category: Database
---
简单总结一下SQL常用的创建查询操作。
<!--more-->

## 创建和使用数据库
#### 创建数据库
```sql
create database test_database
```
#### 创建表
```sql
create table test_table(
    id int PRIMARY KEY,
    name varchar(10)
)
```
[postgres常用数据类型](http://www.cnblogs.com/stephen-liu74/archive/2012/04/30/2293602.html)
[postgres数据类型文档](https://www.postgresql.org/docs/current/static/datatype.html)
#### 插入数据
```sql
insert into test_table (id,name) values(1,'name')
```
#### 更新数据
```sql
update test_table set id=2,name='new name' where id=1
```
#### 删除数据
```sql
delete from test_table where id=2
```

## 查询数据库

#### 查询语句的构成
查询语句由几个子语句构成，下表是查询语句的6个子语句。

子语句名称|使用目的
:---:|:---:
select|确定结果集中应该包含那些列
from|指明所要提取数据的表，以及这些表是如何连接的
where|过滤不需要的数据
group by|对于具有相同列值的行进行分组
having|过滤掉不需要的组
order by|按一个或多个列，对最后结果集中的进行排序

#### select子语句

select子句由于在所有可能的列中，选择查询结果集中要包含那些列，在select子句中，我们可以使用如下内容：
- 字符，比如数字或字符串；
- 表达式，比如transaction.amount*-1；
- 调用内建函数，比如count(*)；
- 用户自定义的函数调用

在select子句中，我们还可以对查询的列起一些别名，使用as可以提高sql的可读性，但不是必须的：
```sql
select name username,id userkey from test_table where id=1
```
```sql
select name as username,id as userkey from test_table where id=1
```
结果如下：
![](/images/12.png)

在某些情况下，查询可能返回重复的行数据，我们可以使用distinct过滤重复行，表数据如下：
![](/images/13.png)
执行查询
```sql
select distinct name from test_table
```
结果如下：
![](/images/14.png)

#### from子语句
from子语句定义了查询中所使用的表，以及连接这些表的方式。from后面可以接三种类型的表：
- 永久表(使用create table语句创建的表)；
- 临时表(子查询返回的表)；
- 虚拟表(使用create view子句创建的视图)；

在单个查询中连接多个表时，需要在select、where、group by、having以及order by子句中指明所引用的是那个表，有两种在from子句之外引用表的方式：
- 使用完整的表名；
- 为每个表指定别名，并在查询中需要的地方使用该别名。

#### where子语句
where子句用于在结果集中过滤掉不需要的行。
where语句可以同时使用and和or操作，为了避免逻辑混乱，建议用`()`增加可读性

#### group by和having子语句
当我们需要在服务器返回结果集之前对数据进行一下提炼，其中一种方式就是使用group by，而having的作用类似于where起过滤作用。

#### order by
order by子句用于对结果集中的原始列数据或是根据列数据计算的表达式结果进行排序，我们可以通过desc和asc指定降序或升序。
举个表达式排序的例子：
```sql
select * from test_table order by right(name,2)
```
该查询使用内建函数right()提取name列的最后三个字段，并根据该值对返回的行排序。

## 过滤
#### 使用圆括号
如果where子句包含3个或更多条件，而且同时使用了and和or操作，那么需要使用圆括号明确意图。
#### 使用not操作符
有些情况下使用not可提高可读性
```sql
/*字段name不为name值的所有行*/
select * from not (name='name')
```
#### 常用条件
符号|描述
:---:|:---:
=|相等
<>|不等
!=|不等
>|大于
<|小于
between ... and| 在...和...之间
in|在...之中
not in|不在...之中

#### 通配符
符号|描述
:---:|:---:
_|正好1个字符
%|任意数目的字符(包括0)

#### 正则匹配
在postgresql中使用正则表达式时需要使用关键字“~”，以表示该关键字之前的内容需匹配之后的正则表达式，若匹配规则不需要区分大小写，可以使用组合关键字“~*”；
mysql可以使用regexp,oracle使用regexp_like函数，sql server允许在like操作符中使用正则表达式。
postgresql可以参考[这篇博客](http://blog.csdn.net/wugewuge/article/details/7704996)

#### null处理
null适合如下场合：
- 没有合适的值
- 值未确定
- 值未定义

当使用null需要注意
- 表达式可以为null，但不能等于null
- 两个null值彼此不能判断为相等
