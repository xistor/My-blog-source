---
title: "Linux/Unix系统编程手册-笔记4.深入探究文件I/O"
date: 2020-07-12T16:01:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

### 原子操作与竞争条件
创建文件 
当同时指定O_EXCL和O_CREAT作为open()的标志位时，如果要打开文件已然存在，open()将返回一个错误。这提供了一种机制，保证进程是打开文件的创建者。负责当open()失败时，再次使用O_CREAT标志调用open()创建文件的过程中可能有另一个进程或线程已经创建了同名文件，因为无论文件存在与否，第二次调用open()总会成功。使用O_EXCL和O_CREAT标志来一次性的调用open()可以避免这种情况，因为其确保检查文件和创建文件属于一个单一的原子操作。

向文件尾部追加数据 
多个进程向同一个文件尾部添加数据也会出现竞争状态，要规避这一问题，需要将文件偏移量移动和数据写操作纳入同一原子操作。在打开文件时加入O_APPEND标志就可以保证这一点。

### 文件控制操作： fcntl()
fcntl()系统调用对一个打开的文件描述符执行一系列控制操作。

比如获取文件访问模式和状态标志
```
int flags, accessMode

flags = fcntl(fd, F_GETFL);
if(flags == -1) {
    errExit("fcntl");
}

if (flags & O_SYNC)
    printf("writes are synchronized\n");

```

判断文件访问模式:O_RDONLY(0),O_WRONLY(1),O_RDWR(2)

```
accessMode = flags & O_ACCMODE;
if (accessMode == O_WRONLY || accessMOde == ORDWR)
    printf("file is writable\n");

```

可以使用fcntl()的F_SETFL命令来修改打开文件的某些状态标志。允许修改的标志有O_APPEND、O_NONBLOCK、O_NOATIME、O_ASYNC和O_DIRECT。系统将忽略其他标志的修改。修改文件状态标志可以先使用fcntl的F_GETFL命令，来获取当前标志的副本，然后修改需要变更的标志位，再通过F_SETFL命令更新状态标志。

```
int flags;
flags = fcntl(fd, F_GETFL);

if (flags == -1)
    errExit("fcntl");
flags |= O_APPEND;
if (fcntl(fd, F_SETFL, flags) == -1)
    errExit("fcntl");
```


### 文件描述符和打开文件之间的关系

