---
title: Maven的配置
date: 2016-11-22 13:43:41
tags: Maven
category: Java
---
ｍaven配置的常用修改。
<!--more-->
## 修改仓库源
#### 全局修改
```xml
<mirrors>
  <mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>        
  </mirror>
</mirrors>
```

#### 项目修改
```xml
<!--发现依赖和扩展的远程仓库列表。-->    
<repositories>
    <repository>
        <id>id</id>
        <name>name</name>
        <url>url</url>
    </repository>
</repositories>
<!--发现插件的远程仓库列表，这些插件用于构建和报表-->
<pluginRepositories>
    <pluginRepository>
        <id>id</id>
        <name>name</name>
        <url>url</url>
    </pluginRepository>
</pluginRepositories>
```

## 编译级别
#### 全局修改
```xml
<profile>
    <id>jdk-1.6</id>
    <activation>
        <activeByDefault>true</activeByDefault>
        <jdk>1.6</jdk>
    </activation>
    <properties>
        <maven.compiler.source>1.6</maven.compiler.source>
        <maven.compiler.target>1.6</maven.compiler.target>
        <maven.compiler.compilerVersion>1.6</maven.compiler.compilerVersion>
    </properties>
</profile>  
```
#### 项目修改
```xml
<build>  
    <plugins>  
      <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-compiler-plugin</artifactId>  
        <configuration>  
          <source>1.5</source>  
          <target>1.5</target>  
        </configuration>  
      </plugin>  
    </plugins>  
</build>   
```

## 代理
```xml
<proxy>
  <id>optional</id>
  <active>true</active>
  <protocol>http</protocol>
  <username>abely_liu</username>
  <password>Welcome10</password>
  <host>172.26.100.238</host>
  <port>64000</port>
  <nonProxyHosts>local.net|some.host.com</nonProxyHosts>
</proxy>
```
