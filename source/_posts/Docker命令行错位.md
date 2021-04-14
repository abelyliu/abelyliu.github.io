---
title: Docker命令行错位
date: 2021-04-10 00:00:00
categories: 
- 系统
tags:
- Docker
---

![image-20210409151844072](http://blog.abely.store/1617952724652-image-20210409151844072.png)

相关问题可以参考

- [Docker Exec does not resize terminal on container · Issue #314 · docker/for-linux (github.com)](https://github.com/docker/for-linux/issues/314)
- [fixes 1492: tty initial size error by lifubang · Pull Request #1529 · docker/cli (github.com)](https://github.com/docker/cli/pull/1529)

低版本的可以用下面的命令

`sudo docker exec -it -e COLUMNS="$(tput cols)" -e LINES="$(tput lines)" container_name bash`

![image-20210409151950959](http://blog.abely.store/1617952790998-image-20210409151950959.png)

