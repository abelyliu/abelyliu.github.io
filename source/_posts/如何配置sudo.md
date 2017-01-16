---
title: 如何配置sudo
date: 2016-08-11 10:56:54
tags: Shell
category: Linux
---
在使用terminal的过程中，权限是绕不开的话题，sudo又是这其中一个非常常用的命令，但是你真的了解它吗？  
<!--more-->
sudo 可以针对单个命令、仅在需要时授予临时权限，减少因为执行错误命令损坏系统的可能性。sudo 也能以其他用户身份执行命令并且记录用户执行的命令，以及失败的权限申请。

## 配置

##### 查看当前设置
sudo -ll   查看当前的sudo配置
sudo -lU user  查看其他用户的配置

##### 修改配置
配置文件是/etc/sudoers，如果配置错误则sudo命令不可用，建议使用visudo进行编辑。visudo会锁住sudoers文件，保存修改到临时文件，然后检查格式，确保正确后会覆盖sudoers文件。
1. 设置用户可以执行所有的命令
```
  用户名 ALL=(ALL)ALL
```
2. 如果只想允许以某个主机名登录用户执行命令
```
  用户名   主机名=(ALL) ALL
```
3. 允许wheel用户组成员无密码使用sudo
```
  %wheel  ALL=(ALL) NOPASSWD: ALL
```
4. 要不询问某个用户的密码
```
  Defaults:USER_NAME      !authenticate
```
5. 只为某个主机名登录用户启用部分命令的执行权限，不用输入密码
```
  USER_NAME HOST_NAME= NOPASSWD: /usr/bin/halt,/usr/bin/poweroff,/usr/bin/reboot,/usr/bin/pacman -Syu
```

最后的设置会覆盖前面的设置，所以限定多的配置应该放到配置文件的后面。
##### 使用技巧
当前用户的环境变量不会应用到sudo启动的程序，除非使用-E选项
```
  sudo -E 命令 参数
```


原文出处 [wiki.archlinux.org](https://wiki.archlinux.org/index.php/Sudo_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
