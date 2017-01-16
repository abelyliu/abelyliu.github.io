---
title: Shell中的if语句
tags: Shell
category: Linux
date: 2017-01-10 10:23:38
---

无论哪种编程语言，if语句都是十分常用的操作(当然汇编之类的之外啦)。在Shell脚本中if脚本也是非常常用的语句，但是它和其它语言if语句有一些明显的不同。我记得我还写过如下的代码：

```sh 
#!/usr/bin/env bash
if  [ 1==2 ]; then
    echo hello
fi
```

你会发现控制台竟然会奇迹般的输出hello。

我们知道一般语言if语言后面会接一个布尔值，在C语言中也可以使用0和非0数值表示。Shell中的特殊之处在于它是计算if后面语句的返回值，该命令的退出状态码是0，则if后面then语句内的内容会被执行，即命令退出状态码为0(命令成功执行)相当于真，其它状态码为假，这和C语言极其相似。

<!--more-->
#### Shell状态码
Shell中每条命令都有返回值，这里简单看一下命令返回值意味着什么：

状态码|描述
:---:|:---:
0|命令成功结束
1|一般性未知错误
2|不适合的shell命令
126|命令不可执行
127|没找到命令
128|无效的退出参数
128+x|与Linux信号x相关的严重错误
130|通过Ctrl+C终止的命令
255|正常范围之外的退出状态码

#### if语句格式
我们刚刚也解释了if后面其实是一个命令，所以if语句的格式应该为
```sh 
if command
then

fi
```
或者
```sh 
if command; then

fi
```
当然我们也可使用else或elif构建更复杂的逻辑。

那我们该如何实现类似于Java或C中的if判断呢？那就是使用test命令。

test用法为test condition，如果condition条件成立会返回0，否则返回非0值。看起来格式如下：
```sh 
if test condition
then

fi
```
bash shell还为我们提供了另一种写法：
```sh 
if [ condition ]
then

fi
```
下面介绍一下test命令的简单用法，test可以进行三类判断：
- 数值比较
- 字符串比较 
- 文件比较

#### 数值比较

比较|描述
:---:|:---:
n1 -eq n2|检查n1是否和n2相等
n1 -ge n2|检查n1是否大于或等于n2
n1 -gt n2|检查n1是否大于n2
n1 -le n2|检查n1是否小于等于n2
n1 -lt n2|检查n1是否小于n2
n1 -ne n2|检查n1是否不等于n2

这里的数值比较只适用于整数，不要用于浮点数的比较。

#### 字符串比较

比较|描述
:---:|:---:
str1 = str2| 检查str1是否和str2相同
str1 != str2| 检查str1是否和str2不同
str1 < str2| 检查str1是否比str2小
str1 > str2| 检查str1是否比str2大
-n str1| 检查str1的长度是否非0
-z str1| 检查str1的长度是否为0

#### 文件比较

比较|描述
:---:|:---:
-d file|检查file是否存在并是一个目录
-e file|检查file是否存在
-f file|检查file是否存在并是一个文件
-r file|检查file文件是否存在并可读
-s file|检查file文件是否存在并非空
-w file|检查file文件是否存在并可写
-x file|检查file文件是否存在并可执行
-O file|检查file文件是否存在并属当前用户所有
-G file|检查file是否存在并且默认组与当前组相同
file1 -nt file2|检查file1是否比file2新
file1 -ot file2|检查file1是否比file2旧

#### 高级数值比较
刚刚我们用test命令进行数值比较，操作比较有限，我们可以使用双括号来使用高级的数学表达式。格式为`(( expression ))`,expression可以是任意的数学赋值或比较表达式，它除了支持test使用的标准数学运算符外，还可以用以下运算符。

符号|描述
:---:|:---:
val++|后增
val--|后减
++val|先增
--val|先减
!|逻辑求反
~|位求反
**|幂运算
<<|左位移
>>|右位移
&|位布尔和
 &#124;|位布尔或
&&|逻辑和
&#124; &#124;|逻辑或

在双括号里面的`>`和`<`都是不需要转义的。

#### 高级字符串比较
有针对数值运算的高级操作，自然就有针对字符串的高级操作，就是使用双方括号[[ expression ]]。双括号里面的expression使用了test命令中采用的标准字符串比较，但它提供了test命令未提供的另一个特性——模式匹配。

所谓的模式匹配就是可以定义一个正则表达式来匹配字符串，举个栗子：

```sh 
#!/usr/bin/env bash
if [[ $USER == a* ]]
then
    echo "true"
fi
```

上面的代码就可以判断当前登陆用户的用户名是否以a开头。

偶尔我们会用到case语句，下面是case语句的格式，没啥难理解的。

```sh 
case variable in
pattern1 | pattern2) commands1;;
pattern3) commands2;;
*) default commands;;
esac
```
#### 示例
最后解释一下一开始的代码问题是==两边要有空格，在shell编程中尤其要注意其格式，该有的空格不能省略，不然会导致shell执行完全不同的计算。上面之所以返回值为0，1=2是一个赋值语句，把1当做一个变量理解即可，但并不可以使用($1是关键字)。

下面就是利用if语句写的一个简单部署脚本
```sh 
#!/usr/bin/env bash
mvn clean
mvn package
if  [[ $1 = 227 ]]; then
    scp *web/target/*.war root@172.26.100.227:/usr/local/liferay-portal/deploy/
elif [[ $1 = 223 ]]; then
    scp *web/target/*.war root@172.26.100.223:/usr/local/liferay-portal/deploy/
fi
```

参考链接：
1. [shell脚本--if判断（数字条件、字符串条件）](http://blog.csdn.net/yusiguyuan/article/details/17054231)
2. Linux命令行与Shell脚本编程大全