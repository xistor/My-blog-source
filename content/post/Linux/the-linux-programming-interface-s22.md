---
title: "Linux/Unix系统编程手册-笔记22. 线程"
date: 2020-10-12T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

本章主要介绍了POSIX线程， 也就是pthread。一个进程可以包含多个线程，线程之间共享全局内存。  
多线程相比多进程有几个优点：
- 共享信息简单快速
- 线程创建速度快

线程之间共享的信息：

- 进程ID和父进程ID
- 进程组ID和会话ID
- 控制终端
- 用户和组ID
- 文件描述符
- 使用fcntl()创建的record locks
- 信号配置
- 文件系统相关信息：umask，当前工作目录，根目录
- 间隔计时器 (setitimer())和POSIX计时器 (timer_create())
- System V信号量
- 资源限制
- CPU 时间消耗(times())
- 资源消耗(getrusage())
- nice值

线程之间不共享的属性：

- 线程id
- 信号mask
- 线程