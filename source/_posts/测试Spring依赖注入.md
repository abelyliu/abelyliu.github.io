---
title: 测试Spring依赖注入
date: 2016-12-15 09:26:51
tags: Spring
category: Java
---
使用spring-test和不使用spring-test测试spring是否正确注入。
<!--more-->
## 不使用spring-test

#### 依赖
添加了依赖注入和junit的依赖。
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.3.4.RELEASE</version>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
</dependency>
```

#### 配置类
配置了自动扫描
```java
package site.abely.config;

import org.springframework.context.annotation.ComponentScan;

@org.springframework.context.annotation.Configuration
@ComponentScan("site.abely.service")
public class Configuration {
}
```

#### 业务代码
```java
package site.abely.service;

public interface Person {
    void sayHello();
}
```

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

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class Speaker {
    @Autowired
    Person person;
    public void say(){
        person.sayHello();
    }
}
```

#### 编写测试用例
测试前先手动注入依赖
```java
package site.abely.service;

import org.junit.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;


public class TestTest {
    private Speaker speaker;

    @Before
    public void init() {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(site.abely.config.Configuration.class);
        speaker = (Speaker) applicationContext.getBean("speaker");
    }

    @org.junit.Test
    public void testSay() {
        speaker.say();
    }
}
```
输出结果`你好`

## 使用spring-test
#### 依赖
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>4.3.4.RELEASE</version>
</dependency>
```
#### 测试代码
```java
package site.abely.service;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes=site.abely.config.Configuration.class)
public class TestTest {
    @Autowired
    Person person;
    @Test
    public void testSay(){
        person.sayHello();
    }
}
```
关于如何载入配置文件@ContextConfiguration可以参考[官方博客](https://spring.io/blog/2011/06/21/spring-3-1-m2-testing-with-configuration-classes-and-profiles)。
