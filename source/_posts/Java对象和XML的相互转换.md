---
title: Java对象和XML的相互转换
date: 2016-11-18 14:57:15
tags: Java
category: Java
---
在编程的过程中，处理XML是一件常见的需求。在Java中，jdk1.6及后自带了处理XML的包，如果在之前版本需要单独引入相关包。
<!--more-->

## Java转XML
#### 实体类
```java
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlRootElement;

@XmlRootElement(name="config")
public class Configuration {
    private String webProxy;
    private boolean verbose;
    private String colorName;
    private String screenName;

    public String getWebProxy() {
        return webProxy;
    }

    @XmlElement
    public void setWebProxy(String webProxy) {
        this.webProxy = webProxy;
    }

    public boolean isVerbose() {
        return verbose;
    }

    @XmlElement
    public void setVerbose(boolean verbose) {
        this.verbose = verbose;
    }

    public String getColorName() {
        return colorName;
    }

    @XmlElement
    public void setColorName(String colorName) {
        this.colorName = colorName;
    }

    public String getScreenName() {
        return screenName;
    }

    @XmlElement
    public void setScreenName(String screenName) {
        this.screenName = screenName;
    }
}
```
#### 模式文件
在Configuration.java目录下执行`schemagen  Configuration.java`，目录下会出现schema1.xsd文件，内容如下：
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<xs:schema version="1.0" xmlns:xs="http://www.w3.org/2001/XMLSchema">

  <xs:element name="config" type="configuration"/>

  <xs:complexType name="configuration">
    <xs:sequence>
      <xs:element name="colorName" type="xs:string" minOccurs="0"/>
      <xs:element name="screenName" type="xs:string" minOccurs="0"/>
      <xs:element name="verbose" type="xs:boolean"/>
      <xs:element name="webProxy" type="xs:string" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>
</xs:schema>
```
#### 生成代码
```java
import javax.xml.bind.JAXBContext;
import javax.xml.bind.JAXBException;
import javax.xml.bind.Marshaller;
import java.io.File;
import java.io.IOException;

public class App {
    public static void main(String[] args) throws IOException, JAXBException {
        //实例化一个Configuration
        Configuration configuration = new Configuration();
        configuration.setColorName("red");
        configuration.setScreenName("abely");
        configuration.setVerbose(false);
        configuration.setWebProxy("no");
        //配置
        JAXBContext jc = JAXBContext.newInstance(Configuration.class);
        Marshaller saver = jc.createMarshaller();
        //输出是否格式化，如果没有此局，将会输出一行。
        saver.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
        //写入
        File file = new File("config.xml");
        saver.marshal(configuration, file);

    }
}
```
#### 结果XML文件：
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<configuration>
    <colorName>red</colorName>
    <screenName>abely</screenName>
    <verbose>false</verbose>
    <webProxy>no</webProxy>
</configuration>
```
#### 其它方式
有时候我们需要定义标签的顺序，在定义实体类时可以如下定义：
```java
import javax.xml.bind.annotation.*;

@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = "config",propOrder = {"screenName","webProxy","verbose","colorName"})
@XmlRootElement
public class Configuration {
    private String webProxy;
    private boolean verbose;
    private String colorName;
    private String screenName;

    public String getWebProxy() {
        return webProxy;
    }

    public void setWebProxy(String webProxy) {
        this.webProxy = webProxy;
    }

    public boolean isVerbose() {
        return verbose;
    }

    public void setVerbose(boolean verbose) {
        this.verbose = verbose;
    }

    public String getColorName() {
        return colorName;
    }

    public void setColorName(String colorName) {
        this.colorName = colorName;
    }

    public String getScreenName() {
        return screenName;
    }

    public void setScreenName(String screenName) {
        this.screenName = screenName;
    }
}
```
`@XmlType(name = "config",propOrder = {"screenName","webProxy","verbose","colorName"})`定义了显示顺序，因为使用了`@XmlAccessorType(XmlAccessType.FIELD)`，所以可以不用`@XmlElement`。两者不能同时使用，如果两个都不使用，也可以正确生成文档，生成文档如下：
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<configuration>
    <screenName>abely</screenName>
    <webProxy>no</webProxy>
    <verbose>false</verbose>
    <colorName>red</colorName>
</configuration>
```

## XML转Java
我们修改一下Configuration.java，覆盖equals方法
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;

    Configuration that = (Configuration) o;

    if (verbose != that.verbose) return false;
    if (webProxy != null ? !webProxy.equals(that.webProxy) : that.webProxy != null) return false;
    if (colorName != null ? !colorName.equals(that.colorName) : that.colorName != null) return false;
    return screenName != null ? screenName.equals(that.screenName) : that.screenName == null;

}
```
在App.java的main方法加入如下语句
```java
Unmarshaller unmarshaller = jc.createUnmarshaller();
Configuration configuration1 = (Configuration) unmarshaller.unmarshal(file);
System.out.println(configuration1.equals(configuration));
```
我们发现输出true，也就是我们从file中得到了一个Configuration对象的实例configuration1。