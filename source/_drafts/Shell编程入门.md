---
title: Shell编程入门
tags: Shell
category: Linux
---
如果大家工作在Linux环境下，大家应该了解Shell的强大之处，可以帮助我们快速完成很多工作。Shell编程的复杂处在于它有一些非常特殊的语法规则，刚接触时容易踩坑。

这里介绍一下刚刚接触Shell脚本时一些注意事项和技巧。

## 变量
##### 变量区分大小写
##### 变量在赋值时等号左右不能有空格 
var1 = 10（错误）；var1=10（正确）
##### 可以使用set命令查看系统的环境变量
##### 使用变量时加上$
$var1; ${var1}
##### 将命令结果赋值给变量
var1=echo "hello"（错误）  
var1=\`echo "hello"\`（使用反引号`，tab键上面的）  
var1=$(echo "hello") （使用$()格式）

## 执行数学运算
操作符|描述
:---:|:---:
ARG1 &#124; ARG2 | 如果ARG1既不是null也不是零值，返回ARG1；否则返回ARG2
ARG1 & ARG2|如果没有参数是null或零值，返回ARG1；否则返回0
ARG1 < ARG2|如果ARG1小于ARG2，返回1；否则返回0
ARG1 <= ARG2|如果ARG1小于等于ARG2，返回1；否则返回0
ARG1 = ARG2|如果ARG1等于ARG2，返回1；否则返回0
ARG1 != ARG2|如果ARG1不等于ARG2，返回1；否则返回0
ARG1 >= ARG2|如果ARG1大于等于ARG2，返回1；否则返回0
ARG1 > ARG2|如果ARG1大于ARG2，返回1；否则返回0
ARG1 + ARG2|如果ARG1和ARG2算术运算和
ARG1 - ARG2|如果ARG1和ARG2算术运算差
ARG1 * ARG2|如果ARG1和ARG2算术运算积
ARG1 / ARG2|如果ARG1被ARG2除的算术商
ARG1 % ARG2|如果ARG1被ARG2除的算术余数
STRING : REGEXP|如果REGEXP匹配到了STRING中的某个模式，则返回该模式匹配


