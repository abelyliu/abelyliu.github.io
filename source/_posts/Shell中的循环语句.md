---
title: Shell中的循环语句
tags: Shell
category: Linux
date: 2017-01-11 15:51:35
---

前面曾经学习了if语句的使用，本来是不准备写循环语句的，其语法和if比较相似，但是考虑到其在编程中的常用性还是决定总结一下循环语句的用法。

## while语句
个人感觉While和if是最相似的，只不过一个是循环一个是分支，其格式如下：
```sh 
while test command
do
    other commands
done
```
关于test命令可以参考前一节[Shell中的if语句]中的介绍。

此外还有一点需要注意，while语句后可以定义多个测试命令，但只有最后一个命令的退出状态码会影响循环，这点了解一下即可。平时一般也不会定义多条命令，而且定义多条测试命令的人一般也了解此规则。

<!--more-->

## until语句
until命令和while工作方式相反，`while　true`循环内的命令会被执行(test command 返回值为0)，`until true`会跳出循环，格式如下：
```sh 
until test commands
do 
    other commands
done
```

如果了解Python语言的话应该一眼就知道until的用法，**until就是直到xx时跳出循环**。until和while一样，只会使用最后一个命令的返回值控制循环。

## for语句
for语句的格式如下：
```sh 
for var in list
do 
    commands
done
```
或
```sh 
for (( variable assignment ; condition ; iteration process ))
```
关于第二种的格式需要注意：
1. 变量赋值可以有空格
2. 条件中的变量不以`$`开头
3. 迭代过程中的算式未使用expr命令格式

关于第一种格式有几点可以注意一下：
```sh 
#!/usr/bin/env bash
for test in A B C
do
    echo $test
done
```
上面是一个典型的for循环，输出结果如下：
```sh 
A
B
C
```

我们可以看到默认的分隔符为空格，实际上是
1. 空格
2. 制表符
3. 换行符

如果我们想修改默认分隔符，修改IFS变量即可：
```sh 
#!/usr/bin/env bash
IFS=$'\n'
i=0
for test in $(cat /home/abely/1)
do
    echo $test
done
```

文件1的内容如下：
```text
a
b
c
d e
f g
```
这样d和e，f和g之间就不会被分割，上面代码注意`IFS=$'\n'`中的`$`不能省略，当然你可以试试加和不加的区别。

最后一点关于for语句要说明的是，可以用for命令来自动遍历目录中的文件，在进行此操作时，必须在文件名或路径中使用通配符，它会强制shell使用文件扩展匹配，文件扩展匹配就是生成匹配指定通配符的文件名或路径名的过程。

```sh 
for test in /home/abely/eclipse/*
do
    echo $test
done
```
结果如下：
```sh 
/home/abely/eclipse/artifacts.xml
/home/abely/eclipse/configuration
/home/abely/eclipse/dropins
/home/abely/eclipse/eclipse
/home/abely/eclipse/eclipse.ini
/home/abely/eclipse/features
/home/abely/eclipse/file_
/home/abely/eclipse/icon.xpm
/home/abely/eclipse/p2
/home/abely/eclipse/plugins
/home/abely/eclipse/readme
/home/abely/eclipse/velocity.log
/home/abely/eclipse/workspace
/home/abely/eclipse/无标题文档
```

在上面三种循环中，我们都可以使用break和continue来控制循环的流程，shell中的break和continue后面可以加一个数字，例如`break 2;`，`break;`和`break 1;`是等价的，后面的数字代表着跳出第几层循环，或者继续第几层循环，主要用于多层循环嵌套中。