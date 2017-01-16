---
title: SQL多表查询
date: 2016-11-14 15:49:50
tags: SQL
category: Database
---

尽管对单个表查询并不罕见，但在应用环境下大多数查询都需要针对两个，三个或者更多的表。下面就来聊聊四种连接查询的方法。
- 内连接
- 外连接
- 交叉连接
- 自然连接

<!--more-->

## 内连接
![](/images/24.png)
![](/images/25.png)
```sql
select * from table1 inner join table2 on table1.name = table2.name
```
![](/images/26.png)
我们可以从上面的例子发现内连接有如下特点：
1. table1特有的name和table2特有的name都不会在结果集中(table1的第五条记录，table2的第六条记录)；
2. table1的一条记录对应table2的多条记录，则每种组合方式都会在结果集中显示(table2一条记录对应table1多条记录也是同样)。

当我们连接三个或以上的表时，我们可以将前面连接起来的表当成一张表，然后与后面的连接。
上面我们使用的是相等连接，我们还可以使用不等、大于、小于连接等。下面是一个不等连接的例子，为了易于显示，我们对结果集进行了排序。
```sql
select * from table1 inner join table2 on table1.name <> table2.name order by table1.name , table2.name
```
![](/images/27.png)
## 外连接
#### 左外连接
```sql
select * from table1 left outer join table2 on table1.name = table2.name
```
![](/images/28.png)
#### 右外联接
```sql
select * from table1 right outer join table2 on table1.name = table2.name
```
![](/images/29.png)
从上面我们可以得到外连接的几个特点：
1. 外联接的结果集大于内连接的结果集
2. 左外连接是以第一个表为基准，第二个表中没有的补空
3. 右外连接是以第二个表为基准，第一个表中没有的补空

**如果我们把上述的=改为<>，结果与内连接相同。**

## 交叉连接
交叉连接就是求两张表的笛卡尔积，它本质上就是在不指定任何连接条件的情况下的连接结果，即所有可能的组合。
```sql
select * from table1 cross join table2 
```
![](/images/30.png)

## 自然连接
也还有一种连接方式是自己指定相关表，而让数据库服务器决定需要什么样的连接条件。所谓的自然连接就是指，依赖多表交叉时相同的额列名来推断正确的连接条件。
```sql
select * from table1 natural join table2 
```
![](/images/31.png)
上述连接类似与下面内连接语句
```sql
select table1.uuid,table1.name,table1.age,table2.gender from table1 inner join table2 on table1.uuid=table2.uuid and table1.name=table2.name
```
