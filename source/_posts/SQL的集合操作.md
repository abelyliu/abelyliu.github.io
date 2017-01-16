---
title: SQL的集合操作
date: 2016-11-11 15:00:40
tags: SQL
category: Database
---
尽管可以在与数据库交互时一次只处理一行数据，但实际上关系数据库通常处理的都是数据集合。
<!--more-->
## 基本概念
集合之间的三种操作我们都比较熟悉(并，交，补)：
- A union B
- A intersect B
- A expect B 

在数据库中我们要使用这些操作，我们要满足以下条件６：
- _两个数据集合必须具有相同数目的列_
- _两个数据集中对应列的数据类型必须是一致的(或者服务器能够将其中的一种转换为另一种)_

## union操作
union与union all操作符可以连接多个数据集，他们的区别在于union对于连接后的集合排序并去除重复项，而union all保留重复项。
table1和table2数据如下
![](/images/15.png)
![](/images/16.png)
```sql
select uuid,name from table1  union select uuid,name from table2
```
![](/images/17.png)
```sql
select uuid,name from table1  union all select uuid,name from table2
```
![](/images/18.png)

## intersect操作
intersect操作符去除了交集区域中的所有的重复行，除此之外，ANSI SQL规范还定义了并不删除重复行的intersect all操作符，不过大部分数据库都没有支持。
```sql
select uuid,name from table1  intersect  select uuid,name from table2
```
![](/images/19.png)

## except操作
A except B 就是Ａ集合减去Ｂ集合
```sql
select uuid,name from table1  except  select uuid,name from table2
```
![](/images/20.png)
except all操作是按Ｂ中重复次数在Ａ中减去，例如如下两个表：
![](/images/21.png)
![](/images/22.png)
```sql
select id from table3  except all select id from table4
```
![](/images/23.png)

## 注意事项
1. 当我们对数据集执行集合操作后进行排序，order by后需要使用第一个表的列名(如果两个表字段名不同)，否者会出现错误。
2. 在执行多个集合操作时注意顺序，默认是自顶向下被解析和执行，但有两点需要注意
    - 根据ANSI SQL标准，在调用集合操作时，intersect操作符比其他操作符具有更高的优先级；
    - 可以使用圆括号对多个查询进行封装，以明确他们的执行顺序。