---
title: Spring循环依赖与Async
tags:
  - Spring
category:
  - Java
date: 2021-04-29 15:09:58
---


如果bean使用了@Async方法，并且bean进行了循环依赖，那么在项目有可能出现BeanCurrentlyInCreationException。本文简单聊聊，@Async是如何影响循环依赖的。测试代码如下：

```java
@Service
public class A {
    //这里A依赖B
    @Autowired
    private B b;

    //注意，这是一个异步方法
    @Async
    public void funA() {
    }
}

@Service
public class B  {
    //这里B依赖A
    @Autowired
    private A a;
}

@SpringBootApplication
@EnableAsync
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

<!--more-->

Spring创建完对象，需要执行applyBeanPostProcessorsAfterInitialization，@Async注解就是在这里进行AOP处理，生成代理类。

![](https://blog.abely.store/image-20210429140529488.png)

![](https://blog.abely.store/image-20210429141957782.png)

这里可看到，执行初始化回调方法后，会进行判断，注入其它Component里的对象bean，和最后处理完毕的对象exposedObject是否是同一个对象。

这里解释下`earlySingletonReference`，`exposedObject`，`bean`三者之间的关系。

bean就是new出来的对象引用，最原始的对象。

exposedObject是bean执行过initializeBean方法后的对象。initializeBean方法里主要做下面几件事

- 回调Aware方法
- 执行postProcessBeforeInitialization处理
- 调用init方法
- 执行postProcessAfterInitialization处理

大部分情况下，exposedObject和bean是相同的，也存在上文中某些BeanProcessor会改变对象，如返回代理对象

earlySingletonReference，如果在当前创建对象的过程中，没有被调用doGetBean方法，那么earlySingletonReference为空，因为第二个参数为false，不会从singletonFactories里获取。如果不为空，要么和bean相同，要么是AOP处理过的代理对象。

如果没有循环依赖，那么AOP的处理就会在initializeBean方法里，如果对象有循环依赖，那么会提前到`getSingleton(beanName,true)`里面执行。



这里值得注意的是，上述代码并不一定能100%复现问题，取决于具体的Spring Bean加载顺序，如果先加载B，那么项目可以正常启动，所以我们可以用下面办法解决

- @Lazy
- @DependsOn
- allowRawInjectionDespiteWrapping配置为true，这会导致注入的对象和实际对象不一致，所以这个不太建议

本身循环依赖就容易来带问题，生产中还是建议分离业务。