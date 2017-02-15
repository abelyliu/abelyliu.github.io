---
title: Java函数式编程
date: 2017-02-14 08:11:04
tags: Functional Programming
category: Java
--
Java8相比较以前的JDK最主要的变化就是添加了对Lam表达式的支持，换而言之，从Java8开始，JDK提供了某种程度上的函数式编程的支持。

函数式编程思想之所以这两年流行开来，个人感觉有如下三个主要原因：
1. 硬件性能问题——如果一种语言的运行速度过慢，一般来说是不适合大规模应用。
2. 没有状态——严格的函数式编程所有变量是不可变的，这样同一个方法任何情况下都会执行相同的操作，这样便于测试，符合现在潮流的测试驱动开发。
3. 并行优势——随着摩尔定律的失效，服务器朝着多核基本是可以预见的，因为上面没有状态的原因，所以函数式代码在程序并行方面有很大的优势。

关于具体什么是函数式编程及其特点可以参考下面的参考链接。

下面我们了解一下Java8中一些常用的函数式操作。

## Map
Map目的是将一个值映射为另一个值
```java 
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
List<Integer> squareNums = nums.stream().
                                map(n -> n * n).
                                collect(Collectors.toList());
```

## FlatMap
 FlatMap可以将一个值映射为多个值
 ```java 
 List<Integer> list = new ArrayList<>();
 for(int i=1;i<10;i++){
     list.add(i);
 }
 List<List<Integer>> lists = new ArrayList<>();
 lists.add(list);
 lists.add(list);
 List<Integer> lists1 = lists.stream().flatMap(s->s.stream()).collect(Collectors.toList());
 lists1.stream().forEach(System.out::println);
 ```

## Filter
Filter可以过滤一部分元素
```java 
List<Integer> list = new ArrayList<>();
for(int i=1;i<10;i++){
    list.add(i);
}
List<Integer> list1 = list.stream().filter(s->s>5).collect(Collectors.toList());
list1.stream().forEach(System.out::println);
```

## ForEach 
ForEach就是遍历所有元素，执行某些操作，`list1.stream().forEach(System.out::println);`就是将集合所有元素打印。peek和foreach的功能是相同的，但一个foreach是terminal操作，一个是peak是Intermediate操作。

一个流只能有一个 terminal 操作，当这个操作执行后，流就被使用“光”了，无法再被操作，一个流可以后面跟随零个或多个 intermediate 操作。

## Reduce
这个方法的主要作用是把 Stream 元素组合起来，下面是一个求和的例子：
```java 
List<Integer> list = new ArrayList<>();
for(int i=1;i<10;i++){
    list.add(i);
}
int list1 = list.stream().reduce(0,(a,b)->a+b).intValue();
System.out.println(list1);
```

## Limit/Skip
limit 返回 Stream 的前面 n 个元素；skip 则是扔掉前 n 个元素。
```java 
List<Integer> list1 = list.stream().filter(s->s>5).limit(2).collect(Collectors.toList());
List<Integer> list1 = list.stream().filter(s->s>5).skip(2).collect(Collectors.toList());
```

## Match
match和filter类似，对每个元素执行判断，filter返回元素，match返回匹配结果。
1. allMatch 每个元素都返回true，allMatch返回true
2. anyMatch 有一个元素返回true，anyMatch返回true
3. noneMatch　没有一个元素为true，noneMatch返回true

```java 
boolean bool = list.stream().filter(s->s>5).anyMatch(s->s<7);
boolean bool1 = list.stream().filter(s->s>5).allMatch(s->s<7);
boolean bool2 = list.stream().filter(s->s>5).noneMatch(s->s<7);
System.out.println(bool+" "+bool1+" "+bool2);
//output 
true false false
```



## 参考链接
1. [Functional Programming For The Rest of Us](http://www.defmacro.org/ramblings/fp.html)
2. [傻瓜函数式编程](https://github.com/justinyhuang/Functional-Programming-For-The-Rest-of-Us-Cn/blob/master/FunctionalProgrammingForTheRestOfUs.cn.md)
3. [函数式编程初探](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html)
4. [函数式编程](http://coolshell.cn/articles/10822.html)
5. [Java 8 中的 Streams API 详解](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/)
