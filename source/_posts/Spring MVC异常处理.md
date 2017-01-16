---
title: Spring MVC异常处理
date: 2016-11-03 14:23:31
category: Java
tags: Spring
---
在Spring中，有三种方式可以将异常转换为响应。

- 特定的Spring异常会自动映射为指定的HTTP状态码;
- 使用@ResponseStatus注解，将其映射为一个HTTP状态码;
- 在方法上添加@ExceptionHandler注解，使其用来处理异常。

<!--more-->

## 自动转换的异常
下面的异常，spring会自动将其转换为对应的http状态码

Spring异常|HTTP状态码
:--:|:--:
BindException|400 - Bad Request
ConversionNotSupportedException|500 - Internal Server Error
HttpMediaTypeNotAcceptableException|406 - Not Acceptable
HttpMediaTypeNotSupportedException|415 - Unsupported Media Type
HttpMessageNotReadableException|400 - Bad Request
HttpMessageNotWritableException|500 - Internal Server Error
HttpRequestMethodNotSupportedException|405 - Method Not Allowed
MethodArgumentNotValidException|400 - Bad Request
MissingServletRequestParameterException|400 - Bad Request
MissingServletRequestPartException|400 - Bad Request
NoSuchRequestHandlingMethodException|404 - Not Found
TypeMismatchException|400 - Bad Request

## 自定义的异常转换
```java
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(value = HttpStatus.BAD_REQUEST,reason = "no reson")
public class MyException extends RuntimeException {
}
```
```java
@RequestMapping("exception")
public String exception(HttpServletRequest request)  {
    System.out.println("hello");
    throw new MyException();
}
```
结果如下图所示：
![](/images/7.png)

## 自定义异常处理

控制器中添加如下代码：
```java
@ExceptionHandler(MyException.class)
public String handleMyException(){
    return "error";
}
```
这样就会返回对应的error.jsp,这个方法只对这个类中的出现的MyException生效。
