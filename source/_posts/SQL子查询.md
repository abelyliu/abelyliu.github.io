---
title: SQL子查询
date: 2016-11-17 16:15:50
tags: SQL
category: Database
---
子查询是指包含在另一个SQL语句内部的查询。子查询总是由括号包围，并且通常在包含语句之前执行。像其它查询一样，子查询也会返回一个如下的结果集：
- 单列单行
- 单列多行
- 多列多行

子查询返回的结果集类型决定了它可能如何被使用记忆包含语句可能使用那些运算符来处理子查询返回的数据。任何子查询返回的数据在包含语句执行完成后都会被丢弃，这使子查询像一个具有作用域的临时表。
<!--more-->

前面说过子查询可以按结果集类型区分，还有一种方式可以划分子查询：
- 非关联子查询
- 关联子查询

示例数据库如下：
![](/images/37.png)
![](/images/38.png)

## 非关联子查询
非关联查询是可以单独执行而不需要引用包含语句中任何内容，这种也是大部分的子查询。
#### 单行单列
如果子查询是非关联和返回单行单列的，这种类型的查询称为标量子查询，并且可以位于常用运算符(=、<>、<、>、<=、>=)的任意一边。如果在等式条件下使用子查询，而子查询又返回多行的结果，那么将会出错。
```sql
select * from table2 where name = (select name from table1 where uuid=1)
```
![](/images/39.png)

#### 多行单列子查询
正如前面所说的，返回多行的子查询不能在等式条件的一边使用，不过另外4个运算符可以用来为这些类型的子查询构建条件。
- in
- not in
- all
- any

```sql
select * from table2 where name in (select name from table1)
```
![](/images/40.png)
```sql
select * from table2 where name not in (select name from table1)
```
![](/images/41.png)
in运算符被用于查看是否能在一个表达式集合中找到某一个表达式，all运算符则用于将某单值与集合中的每个值进行比较。构建这样的条件需要将其中一个比较运算符(=、<>、>、<等)与all运算符配合使用。
```sql
select * from table1 where uuid > all (select uuid from table2 where uuid<3)
```
![](/images/42.png)
当使用not in 或<>运算符比较一个值和一个值集时，必须确保值集中不包含null，否者可能产生未知结果。

与all运算符一样，any运算符允许将一个值与值集中每个成员比较，只要有个比较成立，则条件为真。
```sql
select * from table1 where uuid > any (select uuid from table2 where uuid<3)
```
![](/images/43.png)

#### 多列子查询
 多列子查询可以理解为一张临时表，和数据表的功能相同。
 ```sql
 select * from table1 where ( uuid,name) in (select uuid ,name from table2 )
 ```
![](/images/44.png)

## 关联子查询
前面的子查询语句都是独立于包含语句的，这意味着这些子查询可以被单独执行，并可以检验结果，而关联子查询则不同。
```
select * from table1 where  name  not in (select name from table2 where table1.name = table2.name )
```
![](/images/45.png)
如果我们直接允许括号内的子查询会发现出现错误，子查询和外部的SQL无法分离开。上述的子查询因为有了table1.name使之具有了关联性，先会从table1中得到所有数据，然后为每一行数据执行一次子查询。
 
我们在使用关联子查询时，exists运算符是非常常用的。当我们之关系存在的关系而不在乎数量，就可以使用exists运算符。
```sql
select name,uuid from table2 where not exists (select name from table1 where table1.name = table2.name )
```
![](/images/46.png)

