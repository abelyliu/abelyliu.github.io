---
title: Alibaba Druid使用
date: 2016-11-10 14:30:54
tags: Connection pool
category: Java
---
Druid是阿里巴巴开源平台上的一个项目，整个项目由数据库连接池、插件框架和SQL解析器组成。该项目主要是为了扩展JDBC的一些限制，可以让程序员实现一些特殊的需求，比如向密钥服务请求凭证、统计SQL信息、SQL性能收集、SQL注入检查、SQL翻译等，程序员可以通过定制来实现自己需要的功能。 

本文参照官方说明，但使用java进配置。

<!--more-->
## 配置依赖
```xml
<!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.0.26</version>
</dependency>
```
## 配置StatFilter
Druid内置提供一个StatFilter，用于统计监控信息。
```java
@Bean
public DataSource dataSource() {
    DruidDataSource dataSource = new DruidDataSource();
    dataSource.setUsername("yourname");
    dataSource.setPassword("yourpass");
    dataSource.setUrl("jdbc:postgresql://127.0.0.1:5432/mesportal");
    try {
        dataSource.setFilters("stat");
    } catch (SQLException e) {
        e.printStackTrace();
    }
    return dataSource;
}
```
StatFilter的别名是stat，这个别名映射配置信息保存在druid-xxx.jar!/META-INF/druid-filter.properties。

## 配置展示信息页
Druid内置提供了一个StatViewServlet用于展示Druid的统计信息。

这个StatViewServlet的用途包括：
- 提供监控信息展示的html页面
- 提供监控信息的JSON API

```java
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
    super.onStartup(servletContext);
    ServletRegistration.Dynamic registration = servletContext.addServlet("DruidStatView",com.alibaba.druid.support.http.StatViewServlet.class);
    registration.addMapping("/druid/*");
    customizeRegistration(registration);
}
```
如果想对信息展示页进行权限控制，StatViewServlet自带了两种方法，一种是用户名密码，访问页面前需要通过验证，第二种就是限制那些ip可访问，那些ip不可以访问。
下面以用户名密码举例
```java
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
    super.onStartup(servletContext);
    ServletRegistration.Dynamic registration = servletContext.addServlet("DruidStatView",com.alibaba.druid.support.http.StatViewServlet.class);
    registration.setInitParameter("param-name","abely");
    registration.setInitParameter("param-name","abely");
    Map map = new HashMap<>();
    map.put("loginUsername","abely");
    map.put("loginPassword","pass");
    registration.setInitParameters(map);
    registration.addMapping("/druid/*");
    customizeRegistration(registration);
}
```

## 结果
![](/images/11.png)

[参考链接](https://github.com/alibaba/druid/wiki/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)