---
title: 远程调试Java代码
date: 2017-02-09 11:36:27
tags: Debug
category: Java
---
对于调试技术应该是每个开发人员的必会的技能，但在开发过程中会遇到服务器不在本地的情况，这时候我们就需要用到远程调试的功能。本文使用`Intellij Idea + Jrebel + Tomcat`的环境。

使用Jrebel是为了提高开发效率，Jrebel在6.x版本后推出了可以更新服务器资源的功能，使用此功能主要是减少本地修改项目然后在部署到服务器所需要的时间，当然不使用Jrebel也是可以的。

## 客户端环境

首先编辑我们的服务器配置，下图中的两个差别不是很大,选择remote

![](/images/68.png)

<!--more-->

填入Host和port，port默认5005,其它的可以不做修改，然后根据你服务器JDK版本，复制虚拟机启动参数，一般来说复制第一个

![](/images/69.png)

到此客户端的环境已经准备就绪。

## 服务器端环境

进入服务器Tomcat的bin目录,修改setenv.sh文件，如果没有可以修改catalina.sh或者新建一个setenv.sh文件。

```sh 
JAVA_OPTS="$JAVA_OPTS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005"
export JAVA_OPTS
```

只要服务器没有占用5005,重启服务器即可，服务器端的环境也配置完成。

## 使用调试

具体的调试和本地调试差别不大，也是打断点，观察堆栈信息。不过要注意保持本地代码和服务器代码一致，换句话说，当修改文件后，要重新部署到服务器，否则调试可能出现不一致的行为，一般来说写个部署脚本即可(也就是拷贝war包到服务器的webapps目录下，liferay环境下拷贝到deploy目录下)。

这种方法，每次修改代码都需要服务器重新部署项目，所以可以使用Jrebel优化这一过程，只上传改变的部分，而且不要重启项目。
 
### 下载对应的Jrebel插件

![](/images/70.png)

打开插件页面，按照提示下载`jrebel-stable-nosetup.zip`：

![](/images/71.png)

### 配置服务器

将压缩文件解压后上传到服务器端任意目录，本人上传到`/usr/local/abely/`。

修改setenv.sh文件：
```sh
export REBEL_HOME="/usr/local/abely/jrebel"
JAVA_OPTS="$JAVA_OPTS -agentpath:$REBEL_HOME/lib/libjrebel64.so -Drebel.remoting_plugin=true -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005"
```

重启服务器，服务器端的配置到此完成。

### 本地环境修改

![](/images/72.png)

选中要同步的项目，此例中的audit-report-portlet仅仅是core和web的父项目，用于管理项目，所以此处不需要勾选(服务器并没有对应的代码)。

![](/images/73.png)

第三个就是同步变更的文件。

