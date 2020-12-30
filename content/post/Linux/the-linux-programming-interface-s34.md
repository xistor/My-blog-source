---
title: "Linux/Unix系统编程手册-笔记34. POSIX IPC"
date: 2020-12-31T00:05:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

POSIX 提供的三种IPC和System V类似，分别是消息队列、信号量、共享内存。  
和System V 比较优势在于：
- 接口更简单,如下图
![POSIX IPC](img/the-linux-programming-interface-s34/POSIX_IPC.png)
- 使用名字代替key,使用open、close、以及unlink函数，与传统的的UNIX文件模型更加一致。
- POSIX IPC对象是引用计数的，简化了对象删除，当所有进程都关闭该对象之后对象就会被销毁。

