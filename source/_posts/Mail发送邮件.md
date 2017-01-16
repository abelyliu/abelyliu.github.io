---
title: Java发送邮件
date: 2016-10-06 12:03:32
tags: Java
category: Java
---
使用java发送邮件(使用qq邮箱)
<!--more-->
## 代码
pom.xml
```xml
<dependencies>
   <dependency>
       <groupId>com.sun.mail</groupId>
       <artifactId>javax.mail</artifactId>
       <version>1.5.6</version>
   </dependency>
</dependencies>
```
Java代码如下：
```java
import com.sun.mail.util.MailSSLSocketFactory;

import javax.mail.*;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeMessage;
import java.security.GeneralSecurityException;
import java.util.Properties;

public class Test {
    public static void main(String[] args) throws MessagingException, GeneralSecurityException {
        Properties props = new Properties();
        // 开启debug调试
        props.setProperty("mail.debug", "true");
        // 发送服务器需要身份验证
        props.setProperty("mail.smtp.auth", "true");
        // 设置邮件服务器主机名
        props.setProperty("mail.host", "smtp.qq.com");
        // 发送邮件协议名称
        props.setProperty("mail.transport.protocol", "smtp");
        MailSSLSocketFactory sf = new MailSSLSocketFactory();
        sf.setTrustAllHosts(true);
        props.put("mail.smtp.ssl.enable", "true");
        props.put("mail.smtp.ssl.socketFactory", sf);

        // 设置环境信息
        Session session = Session.getInstance(props);

        // 创建邮件对象
        Message msg = new MimeMessage(session);
        msg.setSubject("JavaMail测试");
        // 设置邮件内容
        msg.setText("这是一封由JavaMail发送的邮件！");
        // 设置发件人
        msg.setFrom(new InternetAddress("xxx@qq.com"));

        Transport transport = session.getTransport();
        // 连接邮件服务器
        transport.connect("your_email@qq.com", "yourpassword");
        // 发送邮件
        transport.sendMessage(msg, new Address[]{new InternetAddress("to_email@sina.com")});
        // 关闭连接
        transport.close();
    }
}
```
----

## 遇到的问题
在使用过程中，本人遇到了以下几个问题
- java.lang.NoClassDefFoundError: com/sun/mail/util/MailLogger
```
Exception in thread "main" java.lang.NoClassDefFoundError: com/sun/mail/util/MailLogger
	at javax.mail.Session.initLogger(Session.java:230)
	at javax.mail.Session.<init>(Session.java:214)
	at javax.mail.Session.getInstance(Session.java:268)
	at com.ouo.Test.main(Test.java:28)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)
Caused by: java.lang.ClassNotFoundException: com.sun.mail.util.MailLogger
	at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
	... 9 more
```
这个问题在于Java Mail是EE实现的一部分，如果在SE环境下使用，需要使用额外的实现包，即在pox.xml中使用
```
<dependency>
    <groupId>com.sun.mail</groupId>
    <artifactId>javax.mail</artifactId>
    <version>1.5.6</version>
</dependency>
```
而非
```
<dependency>
    <groupId>javax.mail</groupId>
    <artifactId>javax.mail-api</artifactId>
    <version>1.5.6</version>
</dependency>
```
[解决链接](http://stackoverflow.com/questions/16807758/java-lang-noclassdeffounderror-com-sun-mail-util-maillogger-for-junit-test-case)

- javax.mail.AuthenticationFailedException: 530 Error: A secure connection is requiered(such as ssl).
```
Exception in thread "main" javax.mail.AuthenticationFailedException: 530 Error: A secure connection is requiered(such as ssl). More information at http://service.mail.qq.com/cgi-bin/help?id=28

	at com.sun.mail.smtp.SMTPTransport$Authenticator.authenticate(SMTPTransport.java:932)
	at com.sun.mail.smtp.SMTPTransport.authenticate(SMTPTransport.java:843)
	at com.sun.mail.smtp.SMTPTransport.protocolConnect(SMTPTransport.java:748)
	at javax.mail.Service.connect(Service.java:366)
	at javax.mail.Service.connect(Service.java:246)
	at javax.mail.Service.connect(Service.java:267)
	at com.ouo.Test.main(Test.java:40)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)
```

经查询，解决方法是添加如下代码

```java
MailSSLSocketFactory sf = new MailSSLSocketFactory();
sf.setTrustAllHosts(true);
props.put("mail.smtp.ssl.enable", "true");
props.put("mail.smtp.ssl.socketFactory", sf);
```

- javax.mail.MessagingException: Could not connect to SMTP host: smtp.qq.com, port: 465;

```
Exception in thread "main" javax.mail.MessagingException: Could not connect to SMTP host: smtp.qq.com, port: 465;
  nested exception is:
	javax.net.ssl.SSLHandshakeException: Received fatal alert: handshake_failure
	at com.sun.mail.smtp.SMTPTransport.openServer(SMTPTransport.java:2120)
	at com.sun.mail.smtp.SMTPTransport.protocolConnect(SMTPTransport.java:712)
	at javax.mail.Service.connect(Service.java:366)
	at javax.mail.Service.connect(Service.java:246)
	at javax.mail.Service.connect(Service.java:267)
	at com.ouo.Test.main(Test.java:40)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)
Caused by: javax.net.ssl.SSLHandshakeException: Received fatal alert: handshake_failure
	at sun.security.ssl.Alerts.getSSLException(Alerts.java:192)
	at sun.security.ssl.Alerts.getSSLException(Alerts.java:154)
	at sun.security.ssl.SSLSocketImpl.recvAlert(SSLSocketImpl.java:2023)
	at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:1125)
	at sun.security.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1375)
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1403)
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1387)
	at com.sun.mail.util.SocketFetcher.configureSSLSocket(SocketFetcher.java:598)
	at com.sun.mail.util.SocketFetcher.createSocket(SocketFetcher.java:372)
	at com.sun.mail.util.SocketFetcher.getSocket(SocketFetcher.java:238)
	at com.sun.mail.smtp.SMTPTransport.openServer(SMTPTransport.java:2084)
	... 10 more
```

解决方法是将jdk8切换为jdk7,或者替换jdk对应的包，[参考链接](http://www.jianshu.com/p/5ba3bde60f21)
