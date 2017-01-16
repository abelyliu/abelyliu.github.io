---
title: Spring MVC文件上传
date: 2016-11-03 10:47:31
category: Java
tags: Spring
---
利用spring mvc实现文件上传
<!--more-->
使用StandardServletMultipartResolver，这种方式是Servlet3.0支持的方式，需要在Servlet3.0容器中。

启用文件上传功能
```java
package com.ouo.config;


import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;

import javax.servlet.MultipartConfigElement;
import javax.servlet.ServletRegistration;

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
    // 启用文件上传
    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration){
        registration.setMultipartConfig(new MultipartConfigElement("/home/work/"));
    }

}
```
此种方法是利用servlet3.0的功能，所以需要在servlet容器中指定multipart的配置。我们的初始化类继承了`AbstractAnnotationConfigDispatcherServletInitializer`,我们可以直接重载customizeRegistration方法来配置multipart属性。如果我们继承`AbstractAnnotationConfigDispatcherServletInitializer`可以用同样的方法。
如果是实现`WebApplicationInitializer`的话，类似如下配置：
```java
DispatcherServlet ds = new DispatcherServlet();
ServletRegistration.Dynamic servletRegistration = context.addServlet("appServlet",ds);
registration.addMapping("/");
registration.setMultipartConfig(new MultipartConfigElement("/home/work/"));
```
无论那种方法，都需要该目录存在，否则会出现异常。

下面是配置multipart解析器
```java
package com.ouo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.multipart.MultipartResolver;
import org.springframework.web.multipart.support.StandardServletMultipartResolver;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.config.annotation.DefaultServletHandlerConfigurer;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import org.springframework.web.servlet.view.InternalResourceViewResolver;

import java.io.IOException;

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

    //配置multipart解析器
    @Bean
    public MultipartResolver multipartResolver() throws IOException{
        return new StandardServletMultipartResolver();
    }


}
```
如果使用web.xml配置multipart属性，如下：
```xml
<servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
    <multipart-config>
        <location>/home/work</location>
        <max-file-size>123</max-file-size>
        <max-request-size>1244</max-request-size>
    </multipart-config>
</servlet>
```
控制器代码如下：
```java
@RequestMapping("/upload")
public String upload(@RequestPart("file") String file){
    System.out.println(file);
    return "index";
}
```
这里注意的是参数的类型，我们可以使用我们想要的参数类型，常见的有`byte[]`,`String`,`MultipartFile`,`Part`。
`org.springframework.web.multipart.MultipartFile`接口如下
![](/images/5.png)
`javax.servlet.http.Part`接口如下：
![](/images/6.png)
两个接口基本上没有什么区别，只不过方法名略微不同。

前台页面代码：
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
    <h1>Register</h1>
    <form method="post" action="/upload" enctype="multipart/form-data">
        First Name:<input type="text" name="firstName"><br/>
        Last Name:<input type="text" name="lastName"><br/>
        Username:<input type="text" name="username"><br/>
        Password:<input type="text" name="password"><br/>
        file:<input type="file" name="file">
        <input type="submit" value="Register">
    </form>
</body>
</html>
```

如果无法在Servlet3.0以上服务器运行，可以使用CommonsMultipartResolver替代使用StandardServletMultipartResolver。
