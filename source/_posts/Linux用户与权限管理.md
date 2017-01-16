---
title: Linux用户与权限
date: 2016-08-05 15:32:40
category: Linux
---
在使用Linux的过程中，我们经常会遇到执行或者打开某个文件权限不足的情况，今天就来讨论如何在Linux下管理用户的权限的基本知识。下面是一些常用操作，如果对某些感兴趣，可以使用man自行查询。
<!--more-->

## 用户管理
1. 增加用户  
增加用户可以使用useradd和adduser命令，就其功能上来说并没有什么不同，adduser提供了更好的交互。如果没有什么特殊要求直接输入`adduser username`即可。

2. 删除用户  
使用格式`userdel username`，值得注意的是执行命令必须要有管理员权限，而且默认不会删除用户的主目录及下面的内容。可以使用-r参数。

3. 修改密码  
命令行中直接输入passwd即可。如果你是管理员权限，可以passwd后加用户名修改其他用户的密码。

4. 修改用户  
usermod是用来修正创建用户错误的初始化参数。比如修改主文件夹，账号名称，初始用户组和次要用户组等等。我们也可以使用此命令将用户添加到一个组里面，格式为`usermod -a -G groupname username`,但因为分配用户到组一般使用gpasswd命令，所以一般很少使用此命令。

## 组管理
1. 增加组  
`groupadd groupname`

2. 修改组  
`groupmod -n new_group_name old_group_name`

3. 删除组  
`groupdel groupname`

4. 用户-组管理  
`gpasswd -a username groupname`向组中增加用户  
`gpasswd -d username groupname`移除组中的用户

## 权限管理
1. 改变文件所属组  
`chgrp [-R] groupname file`

2. 改变文件所有者  
chown [-R] username:groupname file 后面的groupname是可选的。

3. 改变文件权限  
`chmod [-R] xyz file`  
r:4 w:2 x:1，上面的xyz为三种权限的加和，还有一种方式如下：  
`chmod u=rwx,go=rx file`

## ACL
ACL是Access Control List的缩写，主要目的是提供传统的owner、group、others的read、write、execute权限之外的具体权限设置。ACL可以针对单一用户、单一文件或者目录来进行r、w、x的权限设置，对于特殊权限的使用状况非常有帮助。目前大部分文件系统都支持ACL。  

ACL主要针对如下方面控制：  
1. 用户(user)：可以针对用户来设置权限；
2. 用户组(group)：针对用户组来设置其权限；
3. 默认属性(mask)：在该目录下新建文件或目录时设置新数据的默认权限。  

ACL通过getfacl和setfacl来查看和设置。
```bash
    abely@abely-ThinkPad-T400:~$ ll pgadmin.log
    -rw-rw-r-- 1 abely abely 122  8月  4 10:53 pgadmin.log
    abely@abely-ThinkPad-T400:~$ getfacl pgadmin.log
    # file: pgadmin.log
    # owner: abely
    # group: abely
    user::rw-
    group::rw-
    other::r--

    abely@abely-ThinkPad-T400:~$ setfacl -m u:user2:rwx pgadmin.log
    abely@abely-ThinkPad-T400:~$ ll pgadmin.log
    -rw-rwxr--+ 1 abely abely 122  8月  4 10:53 pgadmin.log*
    abely@abely-ThinkPad-T400:~$ getfacl pgadmin.log
    # file: pgadmin.log
    # owner: abely
    # group: abely
    user::rw-
    user:user2:rwx
    group::rw-
    mask::rwx
    other::r--

```
上述代码中我们可以看出`ll`在显示文件属性时后面多了一个`+`，用户user2对pgadmin.log拥有读写和执行的权限。我们也可以为某一组或者other用户分配特定权限`setfacl -m g:groupname:rwx filename`,`setfacl -m o:groupname:rwx filename`。  
前面我们还提到默认属性mask，mask又是什么鬼呢？如果我们对一个文件设置了mask权限，那么在使用setfacl时，我们设置的权限也不会超出mask。简单来说如果给mask设置了rw权限，在用setfacl给某一用户或者某一组设置rwx，用户和组也只会有rw权限。mask主要用法防止分配权限时分配太大的权限，所以mask就是权限的上限。设置格式：`setfacl -m m:rw filename`。删除所有添加的权限可以使用`setfacl -b`。
