---
title: "Linux/Unix系统编程手册-笔记27. 管道"
date: 2020-11-27T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

## 管道的概念

管道一般用于在shell中将一个进程的输出作为另一个进程的输入。比如

```sh
$ ls | wc -l
```

管道是一个字节流，只能按顺序读取任意大小的数据，不能使用fseek()随机读取。  

从当前是空的管道内读取数据会阻塞，直到有数据写入管道，如果管道的写入端已关闭，读管道的进程读完管道内剩余的数据后将会读到end of file。  

管道的数据传输是单向的。  

往管道内写数据到PIPE_BUF个字节前肯定是原子操作。当写入数据大于PIPE_BUF时，内核会把数据分割成任意大小写入管道，当多个进程同时写入时，数据就可能混合。  

管道写满后（Linux 2.6.11 后管道大小为65536），`write()`会阻塞，直到`read()`从管道删除部分数据。  

管道可以让两个相关的进程进行通信，相关指的是两个进程有同一个祖先，因为管道的文件描述符是通过`pipe()`返回的，通过继承在各个进程中传递。一个普遍的场景是，父进程创建管道，之后兄弟进程利用管道通信。

## 创建和使用管道
