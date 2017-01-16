---
title: Hibernate5配置和使用
date: 2016-11-09 14:49:31
tags: Hibernate
category: Java
---
本配置是建立在spring+spring mvc的基础之上，spring+spring mvc的配置可以[参考这篇文章。]()
<!--more-->
## 添加依赖
##### Hibernate依赖
```xml
<!-- https://mvnrepository.com/artifact/org.hibernate/hibernate-core -->
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.1.2.Final</version>
</dependency>
```
##### Spring依赖
```xml
<!-- https://mvnrepository.com/artifact/org.springframework/spring-orm -->
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-orm</artifactId>
  <version>4.3.3.RELEASE</version>
</dependency>
```
spring-orm的依赖关系如下：
![](/images/8.png)
##### 数据库驱动依赖
```xml
<!-- https://mvnrepository.com/artifact/org.postgresql/postgresql -->
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <version>9.3-1104-jdbc41</version>
</dependency>
```
数据库使用的是postgresql
##### 连接池依赖 
```xml
 <!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid</artifactId>
  <version>1.0.26</version>
</dependency>
```
连接池使用的是Alibaba Druid

## Spring与Hibernate的集成
在配置搭建spring+spring mvc时我们并没有使用这个类，现在我们需要在这里配置spring与hibernate集成。
```java
package com.ouo.config;

import com.alibaba.druid.pool.DruidDataSource;
import com.ouo.domain.MesProgramInfoEntity;
import org.hibernate.SessionFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.orm.hibernate5.HibernateTransactionManager;
import org.springframework.orm.hibernate5.LocalSessionFactoryBean;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.springframework.transaction.annotation.TransactionManagementConfigurer;

import javax.sql.DataSource;
import java.io.IOException;
import java.util.Properties;

@Configuration
@ComponentScan("com.ouo")
@EnableTransactionManagement
public class RootConfig  implements TransactionManagementConfigurer {
    //配置数据源
    @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUsername("usr_mesportal");
        dataSource.setPassword("A8mes!");
        dataSource.setUrl("jdbc:postgresql://127.0.0.1:5432/mesportal");
        return dataSource;
    }
    //配置连接
    @Bean
    public SessionFactory sessionFactoryBean(DataSource dataSource) {
        LocalSessionFactoryBean sfb = new LocalSessionFactoryBean();
        sfb.setDataSource(dataSource());
        sfb.setPackagesToScan("com.ouo.dao");
        Properties props = new Properties();
        props.setProperty("dialect", "org.hibernate.dialect.PostgreSQLDialect");
        sfb.setHibernateProperties(props);
        sfb.setAnnotatedClasses(MesProgramInfoEntity.class);
        try {
            sfb.afterPropertiesSet();
        } catch (IOException e) {
            e.printStackTrace();
        }
        SessionFactory sf = sfb.getObject();
        return sf;
    }
    //配置事务
    @Autowired
    private SessionFactory sessionFactory;
    public PlatformTransactionManager annotationDrivenTransactionManager() {
        System.out.println(sessionFactory);
        HibernateTransactionManager transactionManager = new HibernateTransactionManager();
        transactionManager.setSessionFactory(sessionFactory);
        return transactionManager;
    }
}
```
我们知道使用Hibernate最关键的是获得SessionFactory，我们只需要把SessionFactory的创建工作交给spring来完成，并自动注入到需要的Dao即可。上述代码中最核心的就是第二个获得SessionFactory的方法。在配置SessionFactory时需要datasource，所以会有第一个方法，第三个方法是配置事务的,此处也是必须配置，原因会在下面解释。
下面是Dao的具体内容:
```java
package com.ouo.dao;

import com.ouo.domain.MesProgramInfoEntity;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Repository
@Transactional
public class ServerInfoDao {
    @Autowired
    SessionFactory sessionFactory;

    private Session currentSession() {
        return sessionFactory.getCurrentSession();
    }

    public MesProgramInfoEntity findOne(long program_id) {
        List<MesProgramInfoEntity> lists = (List<MesProgramInfoEntity>) this.currentSession().createCriteria(MesProgramInfoEntity.class).list();
        lists.forEach(System.out::println);
        return null;
    }

}
```
上面我们也说了一定要配置事务，原因就在于我们在Dao层使用了`sessionFactory.getCurrentSession();`而非`sessionFactory.openSession();`。
这两者的区别在于：openSession每次打开都是新的Session,所以多次获取的Session实例是不同的，并且需要人为的调用close方法进行Session关闭。getCurrentSession是从当前上下文中获取Session并且会绑定到当前线程，第一次调用时会创建一个Session实例，如果该Session未关闭，后续多次获取的是同一个Session实例；事务提交或者回滚时会自动关闭Sesison,无需人工关闭。
一开始本人并没有加上事务，得到了`org.springframework.web.util.NestedServletException: Request processing failed; nested exception is org.hibernate.HibernateException: Could not obtain transaction-synchronized Session for current thread`错误，参考了[这篇文章。](http://www.cnblogs.com/chyu/p/4817291.html)注意，不要忘记在Dao层使用@Transactional注解。
下面是实体类：
```java
package com.ouo.domain;

import javax.persistence.*;

@Entity
@Table(name = "mes_program_info", schema = "public", catalog = "mesportal")
public class MesProgramInfoEntity {
    private int programId;
    private String name;

    @Id
    @Column(name = "program_id")
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    public int getProgramId() {
        return programId;
    }

    public void setProgramId(int programId) {
        this.programId = programId;
    }

    @Basic
    @Column(name = "name")
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        MesProgramInfoEntity that = (MesProgramInfoEntity) o;

        if (programId != that.programId) return false;
        if (name != null ? !name.equals(that.name) : that.name != null) return false;

        return true;
    }

    @Override
    public int hashCode() {
        int result = programId;
        result = 31 * result + (name != null ? name.hashCode() : 0);
        return result;
    }

    @Override
    public String toString() {
        return "MesProgramInfoEntity{" +
                "programId=" + programId +
                ", name='" + name + '\'' +
                '}';
    }
}
```
controller层如下：
```java
package com.ouo.controller;

import com.ouo.dao.ServerInfoDao;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class Home {
    @Autowired
    ServerInfoDao serverInfoDao;
    @RequestMapping("/test")
    public String index(){
        serverInfoDao.findOne(1L);
        return "index";
    }
}
```
[项目地址](http://git.oschina.net/abely/s2h)
