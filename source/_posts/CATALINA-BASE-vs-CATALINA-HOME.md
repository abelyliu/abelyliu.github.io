---
title: CATALINA_BASE vs CATALINA_HOME
tags: Tomcat
category: Server
date: 2017-01-05 09:39:06
---

在某些情况下我们需要在Tomcat启动时传入某些参数供服务器使用，这时候我们需要在启动脚本中添加我们需要的参数，还有一种比较推荐的方法是设置在setenev.(sh|bat)中，但是我们并不能在Tomcat的bin目录下找到该文件，查询得到如下信息：

>Apart from CATALINA_HOME and CATALINA_BASE, all environment variables can be specified in the "setenv" script. The script is placed either into CATALINA_BASE/bin or into CATALINA_HOME/bin directory and is named setenv.bat (on Windows) or setenv.sh (on *nix). The file has to be readable.</br>
>By default the setenv script file is absent. If the script file is present both in CATALINA_BASE and in CATALINA_HOME, the one in CATALINA_BASE is preferred 

可以看到除了`CATALINA_BASE`和`CATALINA_HOME`之外其它变量都可以在setevn中设置，`setevn`文件放置在`CATALINA_BASE/bin`或`CATALINA_HOME/bin`目录下，需要用户创建，如果两个目录下都存在setevn文件，则`CATALINA_BASE/bin`下的优先级较高。

然后问题也来了`CATALINA_BASE`和`CATALINA_HOME`分别代表什么？

<!--more-->


>Tomcat官方文档指出，CATALINA_HOME路径的路径下只需要包含bin和lib目录，这也就是支持tomcat软件运行的目录，而CATALINA_BASE设置的路径可以包括上述所有目录，不过其中bin和lib目录并不是必需的，缺省时会使用CATALINA_HOME中的bin和conf。如此，我们就可以使用一个tomcat安装目录部署多个tomcat实例，这样的好处在于方便升级，就可以在不影响tomcat实例的前提下，替换掉CATALINA_HOME指定的tomcat安装目录。

CATALINA_HOME：即指向Tomcat安装路径的系统变量  
CATALINA_BASE：即指向活跃配置路径的系统变量  
通过设置这两个变量，就可以将Tomcat的安装目录和工作目录分离，从而实现Tomcat多实例的部署。

这里又牵扯到了什么是Tomcat的多实例？

我们知道如果你有很多个web项目都部署在Tomcat的webapps下，重启服务器所有的web项目都会受到影响，有时候我们只想重启某一个web项目。简单来说就是项目和项目之间有了耦合，最简单的办法就是安装多个Tomcat，分别设置不同的端口，还有一个就是配置Tomcat的多实例，CATALINA_BASE就是指向不同的实例，但是端口不同。

## 参考链接
1. [Tomcat 7 setenv.sh is not found](http://stackoverflow.com/questions/9480210/tomcat-7-setenv-sh-is-not-found?answertab=votes#tab-top)
2. [tomcat - CATALINA_BASE and CATALINA_HOME variables](http://stackoverflow.com/questions/3090398/tomcat-catalina-base-and-catalina-home-variables)
3. [Tomcat多实例单应用部署方案](http://blog.jobbole.com/109347/)
4. [Tomcat 多实例部署脚本](http://www.linjie.org/2015/06/15/Tomcat-%E5%A4%9A%E5%AE%9E%E4%BE%8B%E9%83%A8%E7%BD%B2%E8%84%9A%E6%9C%AC/)