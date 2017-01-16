---
title: 局域网内通过MAC地址查找ip
date: 2016-11-24 15:57:05
tags: Shell
category: Linux
---
本人经常从同事电脑上copy文件，但是每天公司电脑的ip都会动态改变，所以经常询问，而一个电脑的MAC地址是不会经常变动的，所以就写了一个简单的脚本，通过MAC查找ip。
<!--more-->

代码实现的原理很简单，主要利用arp缓存。地址解析协议，即ARP（Address Resolution Protocol），是根据IP地址获取物理地址的一个TCP/IP协议。主机发送信息时将包含目标IP地址的ARP请求广播到网络上的所有主机，并接收返回消息，以此确定目标的物理地址；收到返回消息后将该IP地址和物理地址存入本机ARP缓存中并保留一定时间，下次请求时直接查询ARP缓存以节约资源。

在局域网内ip的数量是有限的，我们可以对所有的ip进行连接，得到它的信息然后看是否匹配MAC地址。结尾的if语句是根据输入参数判断查找谁的ip地址。
```sh
#!/bin/bash
for i in $( seq 0 9)
   do
   {
     if [[ $i == 0 ]]
then
   i=""
    ping -c 1 172.26.100.${i}1 >> /dev/null &  ping -c 1 172.26.100.${i}2 >> /dev/null & ping -c 1 172.26.100.${i}3 >> /dev/null &  ping -c 1 172.26.100.${i}4 >> /dev/null & ping -c 1 172.26.100.${i}5 >> /dev/null &  ping -c 1 172.26.100.${i}6 >> /dev/null & ping -c 1 172.26.100.${i}7 >> /dev/null &  ping -c 1 172.26.100.${i}8 >> /dev/null & ping -c 1 172.26.100.${i}9 >> /dev/null
else
	ping -c 1 172.26.100.${i}0 >> /dev/null & ping -c 1 172.26.100.${i}1 >> /dev/null &  ping -c 1 172.26.100.${i}2 >> /dev/null & ping -c 1 172.26.100.${i}3 >> /dev/null &  ping -c 1 172.26.100.${i}4 >> /dev/null & ping -c 1 172.26.100.${i}5 >> /dev/null &  ping -c 1 172.26.100.${i}6 >> /dev/null & ping -c 1 172.26.100.${i}7 >> /dev/null &  ping -c 1 172.26.100.${i}8 >> /dev/null & ping -c 1 172.26.100.${i}9 >> /dev/null
fi
 } &
done
wait
if [[ $1 == "mercy" ]]
then
   arp -a | grep -Eo "172.26.100.*00:22:68:19:50:02" | awk '{print $1}' | cut -d ")" -f1 | head -1 - > ~/doc/info/mercy_ip
elif [[ $1 == "jalen" ]]; then
	arp -a | grep -Eo "172.26.100.*74:e5:0b:f3:b1:06" | awk '{print $1}' | cut -d ")" -f1 |  head -1 - > ~/doc/info/jalen_ip
elif [[ $1 == "jermy" ]]; then
	arp -a | grep -Eo "172.26.100.*3c:97:0e:1c:79:ad" | awk '{print $1}' | cut -d ")" -f1 |  head -1 - > ~/doc/info/jermy_ip
elif [[ $1 == "abely" ]]; then
	arp -a | grep -Eo "172.26.100.*00:21:5d:1e:6f:6e" | awk '{print $1}' | cut -d ")" -f1 |  head -1 - > ~/doc/info/abely_ip
fi
```