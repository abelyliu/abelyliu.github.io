---
title: Java集合概览
date: 2016-09-08 10:49:51
tags: Java
category: Java
---
总结一下集合的用法
<!--more-->
Java集合定义了两种基本的数据结构，Collection和Map，基本架构如下图。  
![](/images/1.jpg)


## Set接口
set是无重复对象的集合，即任意两个元素满足下面两个条件(null除外，Set中最多有一个null)：
1. a!=b
2. a.equals(b)!=true

类|数据结构|元素顺序|限制|基本操作|迭代性能|备注
:---:|:---:|:---:|:---:|:---:|:---:|:---:
HashSet|哈希表|无|无|O(1)|O(capacity)|最佳通用实现
LinkedHashSet|哈希链表|插入的顺序|无|O(1)|O(n)|保留插入的顺序
EnumSet|位域|枚举声明|枚举类型的值|O(1)|O(n)|只能保存不是null的枚举值
TreeSet|红黑树|升序排列|可比较|O(log(n))|O(n)|元素要实现Comparable或Comparator接口
CopyOnWriteArraySet|数组|插入的顺序|无|O(n)|O(n)|不能使用同步方法，也不保证线程安全

## List接口
类|表示方式|随机访问|备注
:---:|:---:|:---:|:---:|
ArrayList|数组|能|最佳全能实现
LinkedList|双向链表|否|高效的插入和删除
CopyOnWriteArrayList|数组|能|线程安全；遍历块，修改慢
Vector|数组|能|过时的类；同步的方法，不要使用
Stack|数组|能|扩展Vector类；过时了，用Deque替代

## Map接口
类|表示方式|null键|null值|备注
:---:|:---:|:---:|:---:|:---:
HashMap|哈希表|是|是|通用实现
ConcurrentHashMap|哈希表|否|否|通用的线程安全实现
ConcurrentSkipListMap|哈希表|否|否|专用的线程安全实现
EnumMap|数组|否|是|键是枚举类型
LinkedHashMap|哈希表加列表|是|是|保留插入或访问顺序
TreeMap|红黑树|否|是|按照键排序，耗时O(log(n))
IdentityHashMap|哈希表|是|是|比较时使用==，而不使用equals
WeakHashMap|哈希表|是|是|不会阻止垃圾回收键
Hashtable|哈希表|否|否|过时的类，同步的方法，不建议使用
Properties|哈希表|否|否|使用String类的方法扩展Hashtable接口
