---
title: OpenVPN原理介绍
tags:
  - proxy
category:
  - network
date: 2021-04-14 00:00:00
---


在家或者在公司，有时候需要连接VPN才能访问某些服务，不过有时候会出现连接VPN之后网络缓慢，或者完全打不开某些页面。这个时候就需要了解其基本原理，看看是否能改进网络体验。下面是OpenVPN连接的流程示意图。



![](https://blog.abely.store/blog/20210414162751.svg)

<!--more-->

这里以出网为例，如果请求被路由到虚拟网卡，那么open vpn处理程序会重新封装包信息，将包的目的地址和端口号修改为OpenVPN Server端的ip和port，将实际访问的地址封装到包内容中，OpenVPN Server会提取信息，然后代理请求，并返回结果。

默认情况下，vpn会代理所有流量，但是vpn server的流量带宽是有限的，所以有时候会出现网络很慢的情况。在我们公司曾经出现过连上VPN导致很多网站访问不了，这个是因为DNS的问题，首先在访问服务前需要先解析IP地址，默认的地址可能是宽带提供商的地址，vpn server在某些情况下并不能访问本地的ISP的DNS地址，所以配置一个公共的DNS就可以解决问题，如114.114.114.114。

在实际的使用过程中，我们是不需要代理所有的流量请求，我们可以配置只有必要的流量才走虚拟网卡，其它流量还是走默认直连的逻辑。这里也就是配置路由表，我们先看下连上VPN的默认路由表：

![image-20210414170557401](https://blog.abely.store/image-20210414170557401.png)

图中打马的是vpn server的地址，这个一定要走en0网卡，我们也可以看到127开头的都会走lo0本地回环网卡，0/1可以看到能匹配所有的流量，这些流量都会走utun2这个虚拟网卡。

我们当然可以手动修改路由表达到特定的流量走utun2，但是容易出错，而且成本较高，我们可以通过修改Open VPN配置的方式达到同样效果。

我们用文本编辑器打开`.ovpn`文件，加入类似下面代码

```bash
max-routes 1000
# 不使用服务端推送的路由策略
route-nopull
# 配置xxxx.info，xx.xyz域名走vpn代理，这里open vpn在启动的时候会将域名转换成ip，然后写入路由表
route xxxx.info 255.255.255.255 vpn_gateway
route xx.xyz 255.255.255.255 vpn_gateway
# 10网段走vpn代理
route 10.0.0.0 255.0.0.0 vpn_gateway
# 172.16网段走vpn代理
route 172.16.0.0 255.255.0.0 vpn_gateway
# 11.11.11.11这个ip走vpn代理
route 11.11.11.11 255.255.255.255 vpn_gateway
remote-cert-tls server
```

修改后的路由表如下所示

![image-20210414172207799](https://blog.abely.store/image-20210414172207799.png)

这里可以看到，没有`0/1`的策略，默认会走default策略，只有配置的ip才会走utun2，例如10网段。

