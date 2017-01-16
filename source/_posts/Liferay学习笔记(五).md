---
title: Liferay学习笔记(五)
date: 2016-09-02 13:44:12
tags: Liferay
category: Java
---
自定义查询数据库  
<!--more-->

## 基本思路
Liferay中提供了Dynamic Query API，这个个API是用来对Liferay创建的数据库进行动态的查询，包括我们用Server Builder方式创建的数据库。  

Dynamic Query API可以在任何地方使用，包括JSP，Action等。主要步骤如下：
1. 根据要查询的数据表，创建对应的DynamicQuery对象
2. 添加查询条件
3. 调用对应的XXXLocalServiceUtil的方法

比如我们根据groupid和friendlyurl的值查询对应的layout。这个查询写成sql如下：  
`select * from layout_ where groupid=12345 and friendlyurl='/test'`  
如果修改为Dynamic Query API如下：  
```java
DynamicQuery layoutQuery = DynamicQueryFactoryUtil.forClass(
					Layout.class, PortalClassLoaderUtil.getClassLoader());
layoutQuery.add(PropertyFactoryUtil.forName("groupId").eq(Long.valueOf(siteName)));
layoutQuery.add(PropertyFactoryUtil.forName("friendlyURL").eq("/user-maintenance"));
List<Layout> layouts = LayoutLocalServiceUtil.dynamicQuery(layoutQuery);
```
上面的例子比较容易理解，但有一点需要注意：
- 第一句中的`PortalClassLoaderUtil.getClassLoader()`,`DynamicQueryFactoryUtil.forClass`有若干个重载的方法，这里还可以不带`ClassLoader`或者换成`PortletClassLoaderUtil.getClassLoader()`。查资料知，选`PortalClassLoaderUtil.getClassLoader()`或`PortletClassLoaderUtil.getClassLoader()`是根据model的级别，一般内置的lifery数据表都是portal级别的，但测试时，三种皆能查询到结果，有时间再细细研究其中区别。

## 简单的示例
- `SELECT * FROM User_ WHERE lastName='Bloggs'`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("lastName").eq("Bloggs"));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT * FROM User_ WHERE lastName like 'ord%'`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("lastName").like("ord%"));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT * FROM User_ WHERE userId BETWEEN 10931 AND 10945`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("userId").between(new Long(10931), new Long(10945)));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT * FROM User_ WHERE userId < 11376`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("userId").lt(new Long(11376)));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT * FROM User_ WHERE userId < = 11376`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("userId").le(new Long(11376)));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT * FROM User_ WHERE userId  > 14015`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("userId").gt(new Long(14015)));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT * FROM User_ WHERE userId  > = 14015`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("userId").ge(new Long(14015)));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT DISTINCT firstName from User_`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
Projection projection = ProjectionFactoryUtil.distinct(ProjectionFactoryUtil.property("firstName"));
dynamicQuery.setProjection(projection);
List<Object> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT userId, firstName from User_`
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
ProjectionList projectionList = ProjectionFactoryUtil.projectionList();
projectionList.add(ProjectionFactoryUtil.property("userId"));
projectionList.add(ProjectionFactoryUtil.property("firstName"));
dynamicQuery.setProjection(projectionList);
List<Object[]> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- `SELECT * from User_  WHERE firstName = 'Test' AND userId = 10663`  
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
Criterion criterion = null;
criterion = RestrictionsFactoryUtil.eq("firstName", "Test");
criterion = RestrictionsFactoryUtil.and(criterion, RestrictionsFactoryUtil.eq("userId", new Long(10663)));
dynamicQuery.add(criterion);
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
等于如下：  
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
Junction junction = RestrictionsFactoryUtil.conjunction();
junction.add(PropertyFactoryUtil.forName("firstName").eq("Test"));
junction.add(PropertyFactoryUtil.forName("userId").eq(new Long(10663)));
dynamicQuery.add(junction);
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
RestrictionsFactoryUtil.and用来构建and查询，RestrictionsFactoryUtil.or用来构建or查询
- `SELECT * from User_  WHERE lastName = 'Bloggs' order by lastName asc `  
```java
DynamicQuery dynamicQuery = DynamicQueryFactoryUtil.forClass(User.class, PortalClassLoaderUtil.getClassLoader());
dynamicQuery.add(PropertyFactoryUtil.forName("lastName").eq("Bloggs"));
dynamicQuery.addOrder(OrderFactoryUtil.asc("lastName"));
List<User> userList = UserLocalServiceUtil.dynamicQuery(dynamicQuery);
```
- 不去分大小写查找
```java 
DynamicQuery query = DynamicQueryFactoryUtil.forClass(MessageSource.class);
query.add(RestrictionsFactoryUtil.ilike("resourceValue", "%" + resourceValue + "%"));
query.add(RestrictionsFactoryUtil.ilike("resourceBundle", resourceBundle))
```

参考链接：
1. [http://proliferay.com/liferay-dynamic-query-api/](http://proliferay.com/liferay-dynamic-query-api/)
2. [http://www.liferaysavvy.com/2014/06/liferay-dynamic-query-api.html](http://www.liferaysavvy.com/2014/06/liferay-dynamic-query-api.html)
