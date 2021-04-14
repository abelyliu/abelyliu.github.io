---
title: HTTP代理
date: 2021-04-13 00:00:00
categories: 
- network
tags:
- proxy
---


## 反向代理

反向代理是最常见的一种代理，一般用于负载均衡，比如nginx，spring cloud gateway网关。我们不知道提供服务的具体ip，也就是对用户是透明的。

## 正向代理

正向代理和反向代理的区别在于，正向代理是有感知的。比如`curl -x  'http://127.0.0.1:8080'   'http://httpbin.org/'`，我们想要请求`httpbin.org`的数据，我们不直接和`httpbin.org`建立链接，而是和127.0.0.1建立连接，并发请求发送给它。
<!--more-->
![](https://blog.abely.store/httpproxy.svg)

客户端向代理服务器的包结构

![](https://blog.abely.store/image-20210413102449837.png)

客户端直接向目标服务器的包结构

![](https://blog.abely.store/image-20210413102907938.png)

你可以看到，在向代理服务器发送请求的时候，会把整个请求URL都带上`GET http://www.baidu.com/`，这样代理服务器才知道向哪个目标地址发送请求，如果直接向目标服务器发送请求，可以直接使用`GET /`这种方式。

## 隧道代理

上述的正向代理有个问题，无法代理https的请求，https在发送请求前，客户端和服务端需要交换加解密密钥。我们可以用http的隧道代理来实现https协议的代理。主要流程如下：

1. 和proxy服务三次握手，建立tcp连接
2. 向proxy服务发送connect请求，请求里会有目标服务器
3. proxy向目标服务器建立tcp连接
4. proxy和目标服务器成功建立连接后，返回给客户端http 200状态码
5. 客户端向proxy发送TLS握手信息，proxy将信息转发到server端
6. server给proxy返回TLS信息，proxy将信息转发到客户端
7. 等TLS信息交换完毕后，客户端向proxy发送https请求，proxy和上述过程一样，转发信息。

![](https://blog.abely.store/6.svg)



CONNECT协议格式

![](https://blog.abely.store/image-20210413094823603.png)

客户端向代理服务器的包请求

![](https://blog.abely.store/image-20210413103959645.png)

代理服务器向目标服务器的请求

![](https://blog.abely.store/image-20210413104112339.png)

可以看到，随机数，session id都是和客户端发送的信息完全一样，不仅仅是这些，proxy实际上将tcp上层数据都进行了透明的转发。这里代理服务器也无法做中间人攻击，解密中间数据，但是代理服务器只能知道你访问过哪些网站。



## 参考链接

- https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/CONNECT
