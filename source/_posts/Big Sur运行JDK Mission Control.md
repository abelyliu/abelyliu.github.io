---
title: Big Sur运行JDK Mission Control
date: 2020-04-09 00:00:00
categories: 
- Java
tags:
- JDK Mission Control
---

升级到Big Sur之后，发现JDK Mission Control打不开了。

显示包内容，进入到/Applications/JDK Mission Control.app/Contents/Eclipse目录，编辑jmc.ini文件。

![image-20210409144843173](http://blog.abely.store/1617950923756-image-20210409144843173.png)

增加vm参数，修改为自己对应的JDK目录，重新打开APP即可成功运行。