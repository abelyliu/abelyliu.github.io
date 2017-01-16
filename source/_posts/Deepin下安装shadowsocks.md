---
title: Deepin下安装shadowsocks
date: 2016-12-13 11:07:06
tags: Deepin
category: Linux
---
Linux下安装shadowsocks。
<!--more-->

```sh
sudo apt-get install python-pip
sudo apt-get install python-setuptools m2crypto
pip install shadowsocks
sudo apt-get install shadowsocks-qt5
```
如果仓库无法搜寻到shadowsocks-qt5,可以执行如下代码：
```sh
sudo add-apt-repository ppa:hzwhuang/ss-qt5
sudo apt-get update
```
 如想了解更多细节可以参考[这里](https://aitanlu.com/ubuntu-shadowsocks-ke-hu-duan-pei-zhi.html)。
