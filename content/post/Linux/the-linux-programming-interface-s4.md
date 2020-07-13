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

理清这之中的关系需要了解由内核维护的3个数据结构。
- 进程级的文件描述符表
- 系统级的打开文件表
- 文件系统的i-node表

对于每个进程中的文件描述符(open file descriptor)表。该表的每一条目都记录了单个文件描述符的相关信息。
- 控制文件描述符操作的一组标志，（目前只定义了close-on-exec）
- 对打开文件句柄的引用。

内核对于所有打开的文件维护有一个系统级的描述表格，每个条目称为打开文件句柄（open file handle）。一个打开的文件句柄存储了与一个打开文件相关的全部信息：
- 当前文件偏移量
- 打开文件时的所用的状态标志
- 文件访问模式
- 与信号驱动I/O相关的设置
- 对该文件i-node对象的引用
- 文件类型和访问权限
- 一个指针，指向该文件所持有的锁的列表
- 文件的各种属性，包括文件大小以及不同类型操作相关的时间戳。


![文件描述符、打开的文件句柄和i-node之间的关系](/img/the-linux-programming-interface-s4/relationship.png)

上图展示了这三个数据结构之间的关系  
- 在进程A中，文件描述符1和20都指向同一个打开的文件句柄（23），这可能是dup()、dup2()或fcntl()而形成的。dup会创建一个文件描述符的copy,dup2功能类似，区别在于可以指定新的文件描述符而不是使用最小未用编号（之前提过，文件描述符是个小整数）。 如前文所述文件句柄中保存了当前文件偏移量，所以文件描述符指向同一文件句柄将共享文件偏移量，无论这两个文件描述符属于同一进程还是不同进程。同样的文件打开标志也保存在打开的文件句柄中，所以情况一样。（cose-on-exec标志为进程和文件描述符号私有）
- 进程A和B有文件描述符指向同一个打开的文件句柄，这种情形可能在fork之后出现（即A和B之间是父子关系）或者当某进程通过UNIX域套接字将一个打开的文件描述符传递给另一个进程时也会出现。  
- 此外不同的文件句柄也可能指向i-node表中的同一个条目，发生中情况是因为每个进程各自对统一文件发起了open()调用。同一个进程两次打开同一文件，也会发生类似情况。

