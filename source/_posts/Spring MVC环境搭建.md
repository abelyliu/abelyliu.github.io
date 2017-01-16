---
title: Spring MVC环境搭建
date: 2016-11-2 11:40:00
category: Java
tags: Spring
---
快速搭建一个spring mvc web项目。
<!--more-->
## 添加依赖
pox.xml添加如下依赖
```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webmvc</artifactId>
  <version>4.3.3.RELEASE</version>
</dependency>
<dependency>
  <groupId>javax.servlet</groupId>
  <artifactId>javax.servlet-api</artifactId>
  <version>4.0.0-b01</version>
</dependency>
```
spring-webmvc依赖其它一些spring jar，所以此处方便起见并没有单独引进，项目中可以单独引进。
spring-webmvc的依赖关系如下：
![](/images/9.png)
## 在web中启用spring mvc
```java
package com.ouo.config;

import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;

public class Init extends AbstractAnnotationConfigDispatcherServletInitializer {

    //spring 的配置
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[]{RootConfig.class};
    }
    //springmvc 的配置
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{WebConfig.class};
    }
    //拦截所有的请求
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```
1. 为什么Init类能配置spring和spring mvc
我们知道在servlet3.0以后，web.xml就不再是必需的，容器会在类路径中查找实现`javax.servlet.ServletContainerInitializer`接口的类，会用实现类配置Servlet容器。Spring中，`SpringServletContainerInitializer`实现了`ServletContainerInitializer`接口，`SpringServletContainerInitializer`又会委托实现`WebApplicationInitializer`的类来完成初始化。在上述的配置文件中，`AbstractAnnotationConfigDispatcherServletInitializer`实现了`WebApplicationInitializer`接口,或者说继承了接口。
2. 为什么会有spring和spring mvc两个配置
spring mvc配置文件是用来配置Web组件bean（控制器，视图解析器，请求映射关系等），spring配置文件是用来配置后台其它bean，这些bean通常是Service，Dao等。

----
如果不习惯使用java config，可以使用xml，如下：
```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/spring/root-context.xml</param-value>
  </context-param>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>

  <servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>appServlet</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
```

## 配置spring mvc
```java
package com.ouo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.config.annotation.DefaultServletHandlerConfigurer;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import org.springframework.web.servlet.view.InternalResourceViewResolver;

@Configuration
@EnableWebMvc
@ComponentScan("com.ouo.controller")
public class WebConfig extends WebMvcConfigurerAdapter {
    //配置视图解析器
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setExposeContextBeansAsAttributes(true);
        return resolver;
    }

    //配置静态资源不会被拦截
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```
到现在我们就可以启动服务器，RootConfig暂时不需要配置，因为现在关注点不再后台，还有一点是别忘了在`WEB-INF/views/`目录下创建`index.jsp`
