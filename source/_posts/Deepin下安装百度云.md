---
title: Deepin下安装百度云
date: 2016-11-24 09:56:48
tags: Deepin
category: Linux
---
本人操作系统是deepin 15.3，基于Debian定制而来，以前的版本是基于Ubuntu，大家可以根据下载对应版本的bcloud。
bcloud是百度网盘的Gtk/Linux 客户端，[下载链接](https://github.com/LiuLang/bcloud-packages)
<!--more-->
## 安装使用
1. 下载bcloud
2. 安装deb包
3. 打开，软件输入用户名密码

## 遇到的问题
当我登陆后出现网络连接错误，一开始是怀疑代理没有配置，但是我发现系统的代理已经开启，上网查询可以使用如下方法解决
```
Step1：sudo gedit /usr/lib/python3/dist-packages/bcloud/auth.py
Do：在get_bdstoken函数的if req:前一行添加"cookie.load_list(req.headers.get_all('Set-Cookie'))"

Step2：sudo gedit /usr/lib/python3/dist-packages/bcloud/pcs.py
Do：把所有cookie.sub_output()的参数添加上'SCRC','STOKEN'
例：'Cookie': cookie.sub_output('BAIDUID', 'BDUSS', 'PANWEB', 'cflag', 'SCRC', 'STOKEN'),

Step3：删除配置数据和缓存
sudo rm -rf ~/.config/bcloud/*
sudo rm -rf ~/.cache/bcloud/*

Step4：重新运行Bcloud
```

本人重启电脑后重新运行Bcloud才能正常运行。

[参考链接](https://bbs.deepin.org/forum.php?mod=viewthread&tid=42722)
