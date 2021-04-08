
---
title: 如何计算JAVA对象占用内存
date: 2020-01-01 00:00:00
categories: 
- Java
tags:
- Java内存
---

# 如何计算JAVA对象占用内存

## Java对象内存布局

首先我们需要了解下对象内存结构的基础知识，这里不过多展开(这里对象指非数组对象)

![enter description here](https://abelytmp.oss-cn-hangzhou.aliyuncs.com/小书匠/2020381581824087331-015a6de3-07ca-4400-99cd-bb97e241ed67.svg)

一个对象由四部分组成

- mark world 元数据，如对象的hash code，锁偏向等信息
- klass ref 当前对象对应的class对象的引用，非JVM规范，这里是hotspot的优化
- data 当前对象的成员变量等信息(代码等信息存放在方法区中)
- padding 对齐到8字节的倍数(这里注意，也有对齐到16字节，和具体虚拟机有关，为了解释简洁，默认8字节)

在32位的虚拟机中，引用大小都是32位，也就是4字节，对应的mark world也是4字节。即mark world 和klass ref都是4字节。

在64位虚拟机中，引用大小是64位，也就是8个字节，对应的mark world也是8字节。即mark world 和klass ref都是8字节。

<!--more-->

## 指针压缩

如果我们把计算机内存看作是一个一维数组，我们知道引用代表的是内存的地址，32位的范围为`[0000 0000,ffff ffff]`，64位范围为`[0000 0000 0000 0000,ffff ffff ffff ffff]`。

32位最大的内存为4G，当我们内存大于4G后我们则不得不修改JVM为64位虚拟机。

如果你仔细思考，你会发现一个问题，同样的程序在32位虚拟机需要的内存是小于64位虚拟机的(假设程序需要内存小于4g)，对于内存0这个位置，32位表示为`0000 0000` 4个字节，64位表示为`0000 0000 0000 0000` 8字节，我们知道java中有很多引用，这样会导致引用类型占用空间扩大一倍。

为了解决这个问题，就出现了指针压缩机制。

我们上面的图也说过，Java对象占用的内存一定是8的倍数，如果不是，则需要padding到8的倍数。

![enter description here](https://abelytmp.oss-cn-hangzhou.aliyuncs.com/小书匠/2020381579000372485-c0ff5f2a-84e8-44f1-b35b-e2dac3703503.svg)


上图对象1和对象2都是8个字节，对象3可能是8个字节或者8n个字节。我们观察可以发现所有非8n的字节地址是不可能成为对象的地址，所以我们可以人为规定0->0，8->1，16->2 ... 8n->n。在实际寻址的时候再转化为对应实际地址，这样我们就可以用32位表示`4G*8=32G`内存大小的堆。换种方式理解，对象的内存地址十六进制表示，最后三位一定是0，这样我们就没有必要表示这三位，访问内存时通过将地址末尾增加3bit的0(就是乘8)，得到实际内存地址，这样就可以访问32G内存，这种思路在CPU分段，分页寻址时也有利用。

在JDK1.8中，指针压缩是默认开启的，即相当于`-XX:+UseCompressedOops`。所以在JDK1.8中，即使是64位虚拟机，只要没有修改启动参数，且内存小于32G，那么引用类型占用的大小就是32bit，即4个字节。

回到上面的内存布局，在64位压缩模式下，所有引用类型为4字节，即导致klass ref为4个字节而非8个字节。

## 计算示例

```java
public static void main(String[] args){
  List<A> list = new ArrayList<>(10000);
  for (int i = 0; i < 10000; i++) {
    list.add(new A(i, new B(i)));
  }
  sleep(600000);
}

public static class A {
  int a;
  B b;

  public A(int a, B b) {
    this.a = a;
    this.b = b;
  }
}

public static class B {
  long b;

  public B(long b) {
    this.b = b;
  }
}
```

这里我们来分析下A对象占用的内存大小，注意和list大小的区别。

按照我们上面的内存布局来计算`8(mark world)+4(klass ref)+4(int)+4(ref)+4(padding)=24B`。 

同理我们计算B的大小，`8(mark world)+4(klass ref)+4(padding)+8(long)=24B`。

![enter description here](https://abelytmp.oss-cn-hangzhou.aliyuncs.com/小书匠/2020381581748382902-213ecfdf-f8f7-4971-94ae-d46781e94b98.png)

上图中的单位是kB，注意和KB区分开，可以参考相应的维基百科[^1]

上面计算的公式有两点值得注意

1. A对象在计算大小时，并没有加上B对象的大小，只是增加了一个引用的大小(对象的shallow size)

2. 在计算B对象大小时，我们先加padding，然后再加8，这个不是顺序写错了，而是内存布局就是这样

如果你分析过JVM内存泄漏问题，那么应该看到过shallow size和retained size。shallow size代表的是A对象本身的大小，retained size代表A对象本身大小及递归其持有对象大小之和。这里A对象的retained size为24+24=48B。

对于第二个问题，简单解释就是padding不仅仅存在对象末尾，在klass ref和data之间，data内部之间，也是可能存在padding，这里不过多展开，如果想了解更多内存布局的知识，可以参考open jdk的jol工具及其示例[^2]。



## 运行时计算内存占用

上面我们通过手动分析对象结构，从而计算占用大小，在实际开发中通过此种方式计算则比较困难。

- 使用多少位虚拟机，是否开启指针压缩，对齐到8字节还是16字节等等需要事先知晓
- 对象层级结构复杂，持有其它对象比较常见
- String这种动态大小对象很难事先评估



### java.lang.instrument

如果你观察过instrument的api，你会发现有个getObjectSize方法，但是这里也没说明是shallow size还是retained size。这里做个简单的实验

```java
public static void main(String[] args) {
    //直接拿取Instrumentation比较麻烦，这里用ByteBuddy简化代码
    Instrumentation install = ByteBuddyAgent.install();
    System.out.println(install.getObjectSize(new A(1, new B(1))));
}
//output 24
```

可以看到，instrument返回的是shallow size。



### Unsafe

我们可以通过Unsafe拿到对象的内存地址，这样我们就可以通过分析内存地址，计算出对象的实际占用内存。可以参考http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/代码，主要代码如下，思路就是取data里最大的内存偏移地址，然后向8的倍数取整：

```java
public static long sizeOf(Object o) throws NoSuchFieldException, IllegalAccessException {
  Field field = Unsafe.class.getDeclaredField("theUnsafe");
  field.setAccessible(true);
  Unsafe u = (Unsafe) field.get(null);
  HashSet<Field> fields = new HashSet<>();
  Class c = o.getClass();
  while (c != Object.class) {
    for (Field f : c.getDeclaredFields()) {
      if ((f.getModifiers() & Modifier.STATIC) == 0) {
        fields.add(f);
      }
    }
    c = c.getSuperclass();
  }

  // get offset
  long maxSize = 0;
  for (Field f : fields) {
    long offset = u.objectFieldOffset(f);
    if (offset > maxSize) {
      maxSize = offset;
    }
  }

  return ((maxSize / 8) + 1) * 8;   // padding
}
```

我们可以稍微改造下此方法，虚拟机有可能对齐到16字节，所以我们可以动态计算需要对齐多少字节，`Integer.valueOf(System.getProperty("sun.arch.data.model"))/8`;

我们使用此方法再次计算下A内存的大小

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
  System.out.println(sizeOf(new A(1, new B(1))));
}
//output 24
```

我们同样计算出了对象的shallow size。

计算的思路主要就上面两种，很多计算内存占用内存分析的工具思路也是上面，网上还有使用Runtime.getRuntime().totalMemory()和Runtime.getRuntime().freeMemory()计算前后内存查，算近似值，这里只适合较大对象，且还需要确保GC不会影响。接下来我们看下类库中的实现：

### RamUsageEstimator

RamUsageEstimator是 lucene-core 里面的一个工具类，他提供了一个计算对象shallow size的方法，基本原理就是通过反射拿到所有字段，计算私有类型和引用类型占用大小，并对齐字节(实质上就是unsafe方式)

```java
public static void main(String[] args) {
    long l = RamUsageEstimator.shallowSizeOf(new A(1, new B(1)));
    System.out.println(l);
}
//output 24
```



### MemoryMeasurer

https://github.com/DimitrisAndreou/memory-measurer，这个项目提供了一个测量对象大小的方法

```java
public static void main(String[] args) {
    long memory = MemoryMeasurer.measureBytes(new A(1, new B(1)));
    System.out.println(memory);
}
//output 48
```

可以看到，这个工具是直接计算对象的retained size。这个项目因为是agent方式，所以也没有放到maven仓库中，可以拉下来本地编译[^3]。

看到是使用agent方式，应该就能猜出来本质上和instrument方法是一样的，使用了反射遍历对象去计算实际大小。 

### 其它类库

1. https://github.com/fracpete/sizeofag

2. https://github.com/apache/wicket/tree/master/wicket-objectsizeof-agent

3. https://github.com/arturmkrtchyan/sizeof4j
4. https://github.com/jbellis/jamm/

5. https://github.com/phatak-dev/java-sizeof
6. https://github.com/ehcache/sizeof
7. https://mvnrepository.com/artifact/com.carrotsearch/java-sizeof
8. https://openjdk.java.net/projects/code-tools/jol/



前4个和上面的两个例子区别不大。

第5个也类似，不过返回的是retained size，使用Scala写成，按项目文档上描述是从spark项目抽取出来。

第6个实现方式也是通过instrument，它和MemoryMeasurer不同点在于不需要增加agent启动参数，也不是通过我们例子中的ByteBuddy方式attach，而是调用java attach api，但是和ByteBuddy一样，只能运行在JDK的环境中，这里额外说下[EA Agent Loader](https://github.com/electronicarts/ea-agent-loader)这个项目可以在jre环境attach，不过不能是本地类库，而且已经停更了。另一个重要的区别在于这个可以计算shallow size也可以计算retained size。

第7个没有找到对应的主页，这个项目使用unsafe，同时支持shallow size和retained size。

如果你也需要使用运行时计算内存，很明显地7类库是比较方便的。因为没有主页，这里提供一个使用示例，具体使用方法可以阅读RamUsageEstimator的代码。

```java
public static void t1 () {
    System.out.println(RamUsageEstimator.sizeOf(new A(1, new B(1))));
    System.out.println(RamUsageEstimator.shallowSizeOf(new A(1, new B(1))));
}
//output 48 24
```

第8个类库，这个前文已经提过了，很推荐大家把官方的例子运行一遍，涉及到很多内存布局的知识，计算shallow size和retained size也只是其中的一个小功能，这个类库是你理解内存布局的不二之选。



[^1]: https://zh.wikipedia.org/wiki/%E5%8D%83%E5%AD%97%E8%8A%82

[^2]: https://openjdk.java.net/projects/code-tools/jol/

[^3]: https://segmentfault.com/a/1190000007183623










