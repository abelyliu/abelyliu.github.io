---
title: Josephus问题
date: 2016-11-04 13:51:39
tags: Java
category: 算法
---
据说着名犹太历史学家Josephus有过以下的故事：在罗马人占领乔塔帕特后，39 个犹太人与Josephus及他的朋友躲到一个洞中，39个犹太人决定宁愿死也不要被敌人到，于是决定了一个自杀方式，41个人排成一个圆圈，由第1个人 开始报数，每报数到第3人该人就必须自杀，然后再由下一个重新报数，直到所有人都自杀身亡为止。

然而Josephus 和他的朋友并不想遵从，Josephus要他的朋友先假装遵从，他将朋友与自己安排在第16个与第31个位置，于是逃过了这场死亡游戏。

使用Java代码模拟如下：
<!--more-->
```java
int maxperson=41;
int dienum=3;
int num=1;
int dieperson=0;
List<Integer> diepersons = new ArrayList<>();
for(int i=0;dieperson!=maxperson;i++){
    //当到达最后一个人时，从开头继续报数
    if(i==maxperson) i=0;
    //如果当前人死亡，跳过
    if(diepersons.contains(i)){continue;}
    //报数，如果报数和死亡数字吻合，打印死亡这序列号,并记录
    if(num==dienum){
        System.out.print(i+" ");
        diepersons.add(i);
        dieperson++;
        num=1;
        continue;
    }
    num++;
}
```
- num是每个人报的数，从1开始
- i是开始时每个人的序列号
- dieperson是死亡人数，diepersons则记录了哪些人死亡和其顺序，diepersons.size是等于dieperson的。

还有一种可以直接求解最后一个死亡的方法：
```java
public static int Josephus(int n, int m)
{
    if (n == 1)
    {
        return 0;
    }
    else
    {
        return (Josephus(n - 1, m) + m) % n;
    }
}
public static void main(String[] args)
{
    int i = Josephus(41, 3);
    System.out.println(i);
}
```
这里逻辑用到了一些数学证明，如有兴趣，[可以移步。](http://blog.carpela.me/2016/02/22/josephus-problem-and-recursion/)