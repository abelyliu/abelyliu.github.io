---
title: Spring MVC获取前台传递参数
date: 2016-11-2 16:20:00
category: Java
tags: Spring
---
Spring MVC获取前台参数的三种方式。
<!--more-->
## 方式一
```java
@RequestMapping("/login")
public String login(@RequestParam(value = "username",defaultValue = "user1") String username,@RequestParam(value = "password",defaultValue = "123") String password){
    System.out.println("username: "+username+" password: "+password);
    return "index";
}
```
请求url：http://localhost:8080/login?username=%E5%BC%A0%E4%B8%89&password=1111
结果：username: 张三 password: 1111
请求url：http://localhost:8080/login
结果：username: user1 password: 123

## 方式二
```java
@RequestMapping("/user/{userid}")
public String user(@PathVariable("userid") long userid){
    System.out.println(userid);
    return "index";
}
```
请求url:http://localhost:8080/user/52
结果：52
请求url：http://localhost:8080/user/146
结果：146
一般用于Restful接口，表示资源

## 方式三
```java
@RequestMapping(value = "register",method = RequestMethod.GET)
public String registerPage(){
    return "register";
}
```
当我们访问/register时，会出现表单界面，界面代码如下：
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
    <h1>Register</h1>
    <form method="post">
        First Name:<input type="text" name="firstName"><br/>
        Last Name:<input type="text" name="lastName"><br/>
        Username:<input type="text" name="username"><br/>
        Password:<input type="text" name="password"><br/>
        <input type="submit" value="Register">
    </form>
</body>
</html>
```
在form标签中没有action属性，默认还会提交到当前url，以post方式
```java
@RequestMapping(value = "register",method = RequestMethod.POST)
public String register(UserInfo userInfo){
    System.out.println(userInfo);
    return "index";
}
```
UserInfo代码如下：
```java
public class UserInfo {
    String firstName;
    String lastName;
    String username;
    String password;

    @Override
    public String toString() {
        return "UserInfo{" +
                "firstName='" + firstName + '\'' +
                ", lastName='" + lastName + '\'' +
                ", username='" + username + '\'' +
                ", password='" + password + '\'' +
                '}';
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getFirstName() {

        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }
}
```
Userinfo类中，必须有和前台传送过来相同的参数。
