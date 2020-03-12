---
title: C语言学习
---

这里主要是学习C Primer Plus的笔记，下面的代码示例中，有些C语言特性有些是后来标准新加的，这里不做特殊说明。


### 使用前需要先声明
C语言中，变量和方法在使用前需要声明
```c
#include <stdio.h>
void butler(void); /* ANSI/ISO C函数原型 */
int main(void)
{
    printf("I will summon the butler function.\n");
    butler();
    printf("Yes. Bring me some tea and writeable DVDs.\n");

    return 0;
}
void butler(void) /* 函数定义开始 */
{
    printf("You rang, sir?\n");
}
```
我们看到在main方法调用了butler方法，而butler方法需要声明在mian方法前面，C语言称之为函数原型。

### 错误检查

```c
printf("%d minus %d is %d\n", ten);  // 遗漏2个参数
```

在C语言中会正确运行，但是后面两个值可能是内存中的任意值，在Java中则会抛出运行时异常。这里体现出C语言和Java的设计理念，C语言基本上正确性完全由写代码的人保证，某些方面也利于性能。Java则是收紧了这方面的限制，减少代码异常运行的风险，提升了安全性。

再举个例子
```c
#include <stdio.h>

int main(void) {
    unsigned short int x;
    unsigned short int y;
    printf("%d\n", x + y);
    return 0;
}
```

在Java的方法中，局部变量必须初始化才能使用，而C语言则不是，这个结果是任意值。

我们在后面介绍其它语言特性时会反复提及这方面的区别。

### 整数定义

```c
long int estine;
long johns;
short int erns;
short ribs;
unsigned int s_count;
unsigned players;
unsigned long headcount;
unsigned short yesvotes;
long long ago;
```

在C语言中，所有整数都是int，对于不同大小，我们可以加上"形容词"，如`long`，`short`，`unsigned`。在某些情况下可以省略int，如：
- `long johns`等价于`long int johns`
- `short int erns`等价于`short erns`
- `long long ago`等价于`long long int ago`

这里解释下`long long`和`long`的区别，一开始出现C语言的时候，那时计算机普遍还是16位，所以一开始int代表的是16位有符号整数，但是随着计算机的发展，到目前基本都是64位计算机。在C语言标准中，int至少为16bits，long至少为32bits，long long至少为64bits。现在普遍的32位机器中，int代表着32位，long也代表32位，long long代表64位，普遍64位机器中，int代表32bits，long代表32或64bits，long long代表64bits。

我们可以发现为了兼容性，C语言的字段大小是不固定的，应该是根据编译器等决定的(猜测)，这点和Java区别蛮大的。如果想在C语言中使用字节固定的类型，类似于Java，可以引入`#include <stdint.h>`，使用`int32_t`或`int64_t`等类型。

这里再提一点和Java的区别，在Java中一个常数默认是int类型，如果常数大于int值，编译会报错，可以在后面加上`L`之类的修饰，表明是一个long类型或者其它。在C语言中这是隐式的，会默认转换，但这里要主要的一点，如果int存不下，会尝试long，如果long存不下会尝试unsigned long，然后尝试long long，unsigned long long。但是八进制和十六进制会在int后尝试unsigned int，为了避免默认类型的问题，最好也使用`L`之类的后缀。

### 字符类型
C语言中的char类型非常类似于Java中的char类型，但是C语言中char是1个字节，也就是8bits，而Java则是2个字节，16个bits。

```c
char z = 'ni';
printf("%c\n", z);
```

在C语言可以正常运行程序，而在Java中则不允许，因为char被限定为一个字符，无法通过编译。

上面我们也说明过，C语言中char是1个字节，所有在C语言中无法编译下面代码，Java中可以正常编译

```c
char z = '你';
```

不过是不是意味着Java就可以表示任意字符呢？很明显是不可以的，因为char为2个字节，最多只能表示65536个字符，比如下面代码就无法通过编译

```java
char y = '𐀀';
```

这里推荐一个查找Unicode编码的网站，可以找到对应编码值对应的字符，http://unicode.scarfboy.com/。无论是C还是Java，char表示的是一个值，所以我们在定义char的时候可以指定设定字符对应的的Unicode编码值，当然一般不推荐用数字，可以使用`char c = '\u0001'`这种格式。

### 布尔类型
在C语言中，0代表假，非零代表真，一般为1。在C99标准中，新增`_Bool`类型，1bit表示真假。在Java中boolean类型取值则为true或false，这里考你一个小问题，在Java中，一个boolean占用多大内存呢?

### 浮点类型
C语言里的浮点类型类似于整数类型，其具体取值范围和有效位数只定义了最低限制，感兴趣可以自行了解。

```c
double dip1 = 1e1;//10.0      1*10^1
double dip2 = 0x1p10;//1024.0 1*2^10
printf("%f\n", dip1);
printf("%f\n", dip2);
```

C语言支持科学计数法，一种是十进制，一种是16进制，在Java中两者也都可以哦。

### 具体大小
可以用sizeof来查询类型具体占用多少字节
```c
/* C99为类型大小提供%zd转换说明 */
printf("Type int has a size of %zd bytes.\n", sizeof(int));
printf("Type char has a size of %zd bytes.\n", sizeof(char));
printf("Type long has a size of %zd bytes.\n", sizeof(long));
printf("Type long long has a size of %zd bytes.\n",
        sizeof(long long));
printf("Type double has a size of %zd bytes.\n",
        sizeof(double));
printf("Type long double has a size of %zd bytes.\n",
        sizeof(long double));
```

在我的电脑中
```
Type int has a size of 4 bytes.
Type char has a size of 1 bytes.
Type long has a size of 8 bytes.
Type long long has a size of 8 bytes.
Type double has a size of 8 bytes.
Type long double has a size of 16 bytes.
```

### 类型转换
在Java中，如果一个数据类型从大转小，则需要强制指明，在C语言则不需要，比如`int cost = 12.99;`则cost会被初始化为12。

### 字符串
C语言中字符串和Java中字符串都是由char[]存储，在C语言中，所有字符串都以`\0`结束。如果执行`sizeof ""`，你会得到1而非0。如果你想得到正确的char长度，可以使用`strlen`函数，这里注意是char字符串的长度，如果你是一个中文字符，返回的并不是1，这里是和Java不同的。

### 定义常量

```c
#define TAXRATE 0.015
```

一般C语言中常量用全大写字母标识，还有个不太常用的约定，名称前带`c_`或`k_`前缀标识常量。

这里比较有趣的一点关于预处理，也就是#define。

```c
/* 错误的格式 */
#define TOES = 20
```

如果我们这样定义TOES，则意味着如果代码出现`digits = fingers + TOES;`，则实际上会变成`digits = fingers + = 20;`。可以看到，预处理就是个字符串替换的过程。

### 定义不变量
这里的不变量就是Java中的final变量，一般我们定义常量都会设置其为final，在C语言中，关键字为const。

### 数组

数组的初始化

```c
int oxen[SIZE] = {5,3,2,8};
```

虽然Java也可以这样初始化数组，但更一般的是我们使用`int[] oxen= new int[SIZE];`或则`int[] oxen={5,3,2,8}`这种形式。

```c
 int oxen[SIZE] = {5,3,2,8};        /* 初始化没问题 */
int yaks[SIZE];
yaks[SIZE] = {5,3,2,8};        /* 不起作用 */
yaks = oxen;                   /* 不允许 */
```

在C语言中，上面的后两行代码不能编译，不允许这样初始化数组。

C语言中不检查数组下标，如下代码可以运行，如果不仔细编写代码，可能导致未知问题
```c
int main(void)
{
    int name[40];
    name[41]=1;
    return 0;
}
```

> C语言为何会允许这种麻烦事发生？编译器没必要捕获所有的下标错误， 这要归功于C信任程序员的原则。不检查边界，C程序可以运行更快。编译器没有必要捕获所有的额下标错误，因为在程序运行之前，数组的下标值可能尚未确定。因此，为安全起见，编译器必须在运行时添加额外代码检查数组的每个下标值，这会降低程序的运行速度。C相信程序员能编写正确的代码，这样的程序运行速度更快。但并不是所有的程序员都能做到这一点，所以就出现了下标越界的问题。 

### 指针

```c
flizny == &flizny[0]; // 数组名是该数组首元素的地址
```

&放在变量后表示取址，flizny代表的就是数组的第一个元素，也就是数组第一个元素的地址。

关于指针以后有时间单独学习一下。



参考链接：
1. https://stackoverflow.com/questions/6462439/whats-the-difference-between-long-long-and-long
2. https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html