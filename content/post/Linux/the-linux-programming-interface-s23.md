---
title: "Linux/Unix系统编程手册-笔记23. 线程"
date: 2020-10-12T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

本章主要介绍了POSIX线程， 也就是pthread。一个进程可以包含多个线程，线程之间共享全局内存。  
多线程相比多进程有几个优点：
- 共享信息简单快速
- 线程创建速度快

线程在进程中执行时的内存布局如下图：
![线程在进程内运行时内存布局](/img/the-linux-programming-interface-s23/four_threads_in_a_process.png)

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
- 线程相关的数据
- 备用信号栈
- errno 变量
- 浮点数环境（fenv(3)）
- 实时调度策略和优先级
- CPU affinity
- capabilities
- 栈

