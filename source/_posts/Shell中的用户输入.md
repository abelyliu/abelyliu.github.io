---
title: Shell中的用户输入
date: 2017-01-16 14:15:57
tags: Shell
category: Linux
---
前面学习了条件语句(if)和循环语句(for,while,until)的使用方法，如果想要写出比较实用的脚本一般还需要学会处理用户的输入，大部分的脚本都需要在启动的时候输入点什么。
今天学习下面三种输入的用法：
1. 命令行参数
2. 命令行选项
3. 运行时输入

举个例子：`ls -l /home/abely`中-l就是选项，/home/abely就是参数。运行时输入就是在程序运行是通过命令行或者文件等动态输入数据供程序使用。

<!--more--> 

#### 命令行参数
test.sh代码如下：
```sh 
#!/usr/bin/env bash
echo "$0 $1 $2 $3"
```
在终端运行`sh test.sh a b c`，结果如下：
```text
test.sh a b c
```
我们可以看到$0代表程序名称，$1代表第一个参数，$2代表第二个参数,$3代表第三个参数，依似类推。注意第十位及其以后的参数需要用{}，例如${10},${11}。

上面的关于$0表达的并不准确，$0代表什么和你如何运行此脚本有很大关联。

![](/images/55.png)

上面需要注意的是bin目录被我配置在PATH中，我自定义脚本一般都放入此目录，Ubuntu中默认将添加用户bin目录添加到PATH中，只需创建文件夹即可。如果不将test.sh放入到PATH配置的路径中，那么直接运行test.sh会找不到命令。

为了解决上述问题，我们可以使用basename命令`name=$(basename $0)`，这样无论你如何运行脚本$0得到的永远是test.sh。

这里给个建议，在使用参数时可以先检查参数是否存在，如果启动脚本时未传入参数，直接使用程序可能会出现错误。

关于参数还有两个四个特殊的变量：`$#`，`!#`，`$*`，`$@`。

$#代表参数的个数，不输入参数为0。如果你想获得最后一个输入参数，${ $# }是不可以的，在{}中是不能使用$的，这时候就需要是${ !# }来得到最后一个参数。

$\*和$@是用来一次性获得所有用户输入的参数，$\*会把所有参数当成一个单词，$@是每个参数当成一个单词，我们可以使用for循环遍历每个单词。

最后关于参数还有一个操作符`shift`，如果在脚本中执行一下shift命令，那么$1会被删除，$1=$2，$2=$3...，所有的参数都会左移，如果shift后面加上一个数值，如shift 3，则代表向左移动三个参数，$1，$2，$3会被$4，$5，$6代替，关于shift了解即可。

## 命令行选项
关于命令行选项处理有如下难点：
1. 有的选项会带有参数，有些选项可能就不需要参数；
2. 如何界定参数和选项

在代码中手动实现上述功能在简单的脚本中还行的通，稍微复杂一些可能就不行了。这时候我们可以使用getopt命令，它已经替我们实现好了解析选项的功能，其格式如下：
```sh 
getopt optstring parameters
```
optstring定义了有效的选项字母，还定义了那些选项字母需要参数值。

定义方法是在optstring中列出你要在脚本中使用到的每个命令行选项字母，在需要参数的选项后面加一个冒号。

```sh 
getopt ab:cd -a -b test1 -cd test2 test3
```

我们来分析一下上述代码：首先定义了四个选项`-a`，`-b`，`-c`，`-d`，且b必须带参数，后面是用户输入的选项和参数。了解上述代码的意义，现在可以看看如何在脚本中使用了。

简单来说就是我们把用户输入的选项和参数信息先格式化然后再供用户使用。

```sh 
#!/usr/bin/env bash
set -- $(getopt -q ab:cd "$@")
while [ -n "$1" ]
 do
    case "$1" in
        -a) echo "found a";;
        -b) param="$2"
        echo "found b with para value $param"
        shift ;;
        -c) echo "found c";;
        --) shift
            break  ;;
        *) echo"$1 not found" ;;
    esac
    shift
done
count=1
for param in "$@"
do
    echo "param #$count: $param"
    count=$[ $count + 1]
done
```
运行参数为`-a -b haha 1 2 3`，输出结果为：
```sh 
found a
found b with para value 'haha'
param #1: '1'
param #2: '2'
param #3: '3'
```

除了getopt之外，getopts也可以处理选项，这里就不讨论其区别，下面仅给出其一个示例,如想了解，可以[戳这里](http://www.cnblogs.com/FrankTan/archive/2010/03/01/1634516.html)。

```sh 
#!/usr/bin/env bash
while getopts :ab:c opt
do
    case "$opt" in
        a) echo "found a" ;;
        b) echo "found b with value $OPTARG" ;;
        c) echo "found c" ;;
        *) echo "unkown option:$opt" ;;
    esac
done
```
输入参数为`-a -b 1 -c `,输出结果为：
```text
found a
found b with value 1
found c
```

下面表格中是一些约定的选项释义，可以更方便的记忆的使用：

选项|描述
:---:|:---:
-a|显示所有对象
-c|生成一个计数
-d|指定一个目录
-e|扩展一个对象
-f|指定读入数据的文件
-h|显示命令的帮助信息
-i|忽略文本大小写
-l|产生长格式版本
-n|使用非交互模式(批处理)
-o|将所有的输出重定向到指定的文件
-q|以安静模式运行
-r|递归地处理目录和文件
-s|以安静模式运行
-v|生成详细输出
-x|排除某个对象
-y|对所有的问题回答yes

## 运行时输入
如果想要在代码运行的某个时间内动态的输入某些信息，我们可以使用read命令。

read命令从键盘读取变量的值，通常用在shell脚本中与用户进行交互的场合。该命令可以一次读取多个变量的值，变量和输入的值都需要使用空格隔开。在read命令后面，如果没有指定变量名，读取的数据将被自动赋值给特定的变量REPLY。

```sh 
#!/usr/bin/env bash
echo "please enter your name:"
read name
echo "hello $name"
```
运行结果如下：
```text
please enter your name:
abely
hello abely
```

关于read的其它高级用法，可以参考下面的参考连接。

## 参考链接
1. Linux命令行与Shell脚本编程大全
2. [read命令](http://man.linuxde.net/read)
