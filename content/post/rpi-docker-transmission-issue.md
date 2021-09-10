---
title: "docker transmission 下载没速度"
date: 2021-09-11T23:03:16+08:00
tags: ["raspberrypi"]
categories: ["raspberrypi"]
---

raspberrypi下用docker部署transmission，用的是LinuxServer的image:https://hub.docker.com/r/linuxserver/transmission, 部署成功但下载一直没速度。确认所有配置都没有问题。

偶然发现transmission添加任务的时间为1970年。  

进入docker 容器内执行`date` 显示的时间也是1970年。

```
$docker exec -it 22ed19144c05 sh
root@22ed19144c05:/# date
Thu Jan  1 01:00:00 CET 1970
```

遂以此为线索，经过一番搜索，发现这是一个叫libseccomp2的库BUG， 只影响基于Debian Buster的32位系统，很不幸raspberry pi os就是。

https://docs.linuxserver.io/faq#my-host-is-incompatible-with-images-based-on-ubuntu-focal  

按照文中的命令, 在raspberry pi os中添加backports的源并安装libseccomp2

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC 648ACFD622F3D138
echo "deb http://deb.debian.org/debian buster-backports main" | sudo tee -a /etc/apt/sources.list.d/buster-backports.list
sudo apt update
sudo apt install -t buster-backports libseccomp2
```

之后，transmission可以正常下载，done.