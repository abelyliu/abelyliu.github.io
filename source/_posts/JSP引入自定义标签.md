---
title: JSP引入自定义标签
date: 2016-09-09 11:04:16
tags: Jsp
category: Java
---
在jsp中找不到标签的问题。
<!--more-->
在使用IDE时，发现jsp页面中许多变量未定义，最终发现是对应的标签未导入。
```jsp
<%@ taglib prefix="s" uri="/struts-tags" %>
<%@ taglib uri="http://java.sun.com/portlet_2_0" prefix="portlet"%>
```
上述两个标签分别代表两种情况
1. 对应的jar未被导入
2. 未的配置标签

第一种情况在pom.xml或classpath中配置对应的jar文件即可，此例中配置如下：
```xml
<dependency>
	<groupId>org.apache.struts</groupId>
	<artifactId>struts2-core</artifactId>
	<version>2.1.8.1</version>
	<scope>provided</scope>
</dependency>
```
本人因项目分为3个module，core,web,和parent，我将struts的依赖放置在core的pom.xml文件中，web依赖core，后将core中的依赖放入父module的pom.xml文件中即可。

第二种情况是对应的jar文件中没有对应的标签，则我们需要手动引入。手动引入也有两种方法：
1. 将标签放到`WEB-INF/tld`文件夹下，然后将`<%@ taglib uri="http://java.sun.com/portlet_2_0" prefix="portlet"%>`修改为`<%@ taglib uri="/WEB-INF/tld/liferay-portlet.tld" prefix="portlet" %>`
2. 修改web.xml，添加如下内容
```xml
<jsp-config>
	<taglib>
		<taglib-uri>http://java.sun.com/portlet_2_0</taglib-uri>
		<taglib-location>/WEB-INF/tld/liferay-portlet.tld</taglib-location>
	</taglib>
</jsp-config>
```
