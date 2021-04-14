---
title: Socket挂起异常
date: 2021-04-04 00:00:00
categories: 
- Java
tags:
- Bug排查
- network
---

最近发现一个项目很多线程停留在`java.net.SocketInputStream.socketRead0(Native Method)`中，并且永远也不会返回。

![线程栈](https://blog.abely.store/carbon.svg)

先说下结论，要想出现这个问题，需要满足下面几个条件

- 使用了Apache http 类库，这里没有确认版本情况，笔者使用的是4.5.x版本
- 使用了代理访问目标地址
- 没有配置SocketConfig
- 目标服务器断开连接时，RST包丢失

<!--more-->

## 代码分析

和服务器通讯要分为两步，第一步是建立连接，第二步才能发送HTTP请求数据。如果没有代理，那么establishRoute方法里面会进行tcp的三次握手，如果使用代理则复杂一点。

![](http://blog.abely.store/image-20210413180607724.png)我们看下建立隧道连接的过程

![](http://blog.abely.store/image-20210414142619604.png)

对http代理不太了解的可以参考前面的http 代理相关文章，我们可以看到，第一步使用了connect timeout参数，在23步的时候，这个时候并没有使用任何timeout参数，这里使用的是默认的socket配置，如果这时候连接断开，且包丢失，则会出现线程被无限挂起的问题。

在问题的排查过程中，其实先通过抓包确定了会出现RST/FIN包丢失的情况，才进行的代码分析。这里也考虑过socket keep alive机制，抓包也印证了没有开启keep alive机制。可以看到，默认的配置是不会开启keep alive选项。

![](http://blog.abely.store/image-20210414143720659.png)

解决的问题的方法很简单，只要在创建连接池的时候配置SocketConfig的超时即可。这里其实有人针对这个问题提过issue，[[HTTPCLIENT-2090\] Read timeout not applied for SSLHandshake when using proxy - ASF JIRA (apache.org)](https://issues.apache.org/jira/browse/HTTPCLIENT-2090)，不过好像是5.x的PR。

## 实验验证
Linux电脑作为客户端(`192.168.5.9`)，Mac电脑作为代理服务端(`192.168.5.3`)，请求百度服务器服务器。

请求代码如下

```java
public static void httpclient() throws IOException {
  HttpHost proxy = new HttpHost("192.168.5.3", 6152);
  HttpClient httpclient = HttpClients.custom().setProxy(proxy)
    .setDefaultRequestConfig(RequestConfig
                             .custom()
                             .setSocketTimeout(1000)
                             .setConnectTimeout(1000)
                             .build())
    .build();
  HttpGet request = new HttpGet("https://www.baidu.com");
  httpclient.execute(request);
  System.out.println("hello");
}
```

防火墙策略配置如下：

```sh
//拒绝rst包
sudo iptables -I INPUT -p tcp --tcp-flags ALL RST,ACK -j DROP
sudo iptables -I INPUT -p tcp --tcp-flags ALL RST -j DROP
//拒绝带有内容的包
sudo iptables -p tcp -s  192.168.5.3 -A INPUT -m length --length 65:65535  -j DROP
```

![](http://blog.abely.store/image-20210319230045682.png)

![](http://blog.abely.store/image-20210319231152660.png)

可以看到线程和线上服务一样，在establishRoute方法挂起。

