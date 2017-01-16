---
title: Autowired Resource Inject的区别
date: 2016-12-14 21:05:04
tags: Spring
category: Java
---
我知道在Spring中,可以用@autowired @resource @Inject三种方式进行依赖注入，三种方式的区别在什么地方呢？
<!--more-->
## 定义测试用例
如果没有兴趣可以直接跳到下面的[结论](#结论)。

如果不知道如何测试Spring项目的可以点击这里。

首先我们定义一个Person接口：
```java
package site.abely.service;

public interface Person {
    void sayHello();
}
```
然后定义两个实现类Chinese,American
```java
package site.abely.service;

import org.springframework.stereotype.Component;

@Component
public class Chinese implements Person {
    @Override
    public void sayHello() {
        System.out.println("你好");
    }
}
```

```java
package site.abely.service;

import org.springframework.stereotype.Component;

@Component
public class American implements Person {
    @Override
    public void sayHello() {
        System.out.println("Hello");
    }
}
```


## 结论
#### @Autowired
**先按类型注入，然后按照名字注入，都无法找到唯一的一个实现类出错。**

我们将American类中@Component注释，这样在Spring环境中只有Chinese一个实现类,测试代码如下：
```java
@Autowired
Person person;
@Test
public void testSay(){
    person.sayHello();
}
```
输出`你好`。

因为只有Chinese实现了Person，所以会正确运行(通过类型查找)。

我们取消American类中@Component的注释，运行程序会出现异常：
```
Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'site.abely.service.Person' available: expected single matching bean but found 2: american,chinese
```

因为有American和Chinese两个实现类，Spring不知道该用哪一个注入。

修改测试代码：
```java
@Resource
Person chinese;
@Test
public void testSay(){
    chinese.sayHello();
}
```
结果会输出`你好`。此时变量名chinese等于Chinese默认的Qualifier名字。

如果只有一个实现类Chinese（删除American类），而且Chinese不实现Person接口，此时怎么注入Person chinese都会出错(请求的是Person对象，注入的却不是)。

@Autowired(required=false)中如果设置required为false(默认为true)，则注入失败时不会抛出异常，但`person.sayHello();`调用时会出现空指针异常。

#### @Inject
**在Spring的环境下，@Inject和@Autowired是相同的**，都是使用AutowiredAnnotationBeanPostProcessor来处理依赖注入，@Inject是jsr-330定义的规范，还是比较推荐使用这种方式进行依赖注入，如果使用这种方式，切换到Guice也是可以的。

如果硬要说两个的区别，首先@Inject是Java EE包里的，在SE环境需要单独引入。另一个区别在于@Autowired可以设置required=false而@Inject并没有这个设置选项。

#### @Resource
**先按名字注入，在按类型注入，都无法找到唯一的一个出现异常**。

这是jsr-250定义的规范，相对于jsr-330算是比较老的了。这里不推荐使用这种注入方式，下面讨论一下其注入的问题。

首先我们注释American里的@Component,这样在Spring托管的Bean里只有Chinese实现了Person接口，测试用例如下：
```java
@Resource
Person person;
@Test
public void testSay(){
    person.sayHello();
}
```
输出结果:`你好`。 此时@Resource先按名字person,并未找到person的bean，然后按照类型来找，只有Chinese，注入成功。

取消American中的@Component注释，出现如下异常：
```
Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'site.abely.service.Person' available: expected single matching bean but found 2: american,chinese
```
此时无论通过名字无法确定，然后通过类型还是无法确定，抛出异常。

修改测试代码
```java
@Resource
Person chinese;
@Test
public void testSay(){
    chinese.sayHello();
}
```
输出结果`你好`，此时按名字找到了chinese。

## 特殊情况
我们上面也说了，我们推荐是用@Inject，不会与Spring产生耦合，当然如果有必要也可以使用@Autowired，为什么不推荐使用@Resouce呢？

现在我们让Chinese不实现Person接口，但仍然被Spring管理，测试代码如下：
```java
@Resource
Person chinese;
@Test
public void testSay(){
    chinese.sayHello();
}
```
结果如下：
```java
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'site.abely.service.TestTest': Injection of resource dependencies failed; nested exception is org.springframework.beans.factory.BeanNotOfRequiredTypeException: Bean named 'chinese' is expected to be of type 'site.abely.service.Person' but was actually of type 'site.abely.service.Chinese'
```
结果的问题在于，American是Person的唯一实现类而且被Spring托管，此时却不会被注入。

使用@Autowired输出`Hello`

对比你可以发现@Resource的问题所在，也许你认为@Resouce的结果是合理的，但是你要考虑到@Qualifier或@Named的作用，我们可以用如下代码取得和@Resouce类似的效果：
```java
@Autowired
@Qualifier("chinese")
Person chinese;

@Test
public void testSay() {
    chinese.sayHello();
}
```
出现异常：
```java
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'site.abely.service.Person' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true), @org.springframework.beans.factory.annotation.Qualifier(value=chinese)}
```
这样也会去找一个bean id为chinese的bean，在这种情况下(只有一个实现类)，@Resouce要想获得正确的注入只有一个方式，正确的命名`Person american`或`Person person`(person能成功的原因在于Spring中没有bean的id为person的托管类),如果命名不正确，即使使用`@Qualifier("american")`也不会正确注入(@Resouce不会鸟它的)，这会给开发者带来额外的负担，**即使只有一个实现类，@Resouce也有可能无法注入**。这也是开发中常见的事情，如果你的命名和Sping中某个bean的id相同，@Resouce会出现一些意想不到的问题。
