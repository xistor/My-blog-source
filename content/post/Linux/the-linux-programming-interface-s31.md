---
title: "Linux/Unix系统编程手册-笔记31. System V 信号量"
date: 2020-12-17T18:30:00+08:00
tags: ["Linux"]
categories: ["Linux系统编程手册阅读"]
---

信号量和其他sys v IPC不同，它不是用来传输数据的，它是用来进程间同步的。  
信号量是一个由内核维护的整数，其值被限制为大于等于0。下图是两个进程使用信号量同步的过程：

![使用信号量同步](/img/the-linux-programming-interface-s31/semaphore_sync.png)

## 使用步骤
- 使用`semget()`创建和打开一个信号量
- 使用`semctl()`带上`SETVAL`或`SETALL` flag初始化信号量的值（只需一个进程做这个）。
- 使用`semop()`操作信号量的值，加减一般用来代表释放和请求资源。
- 当所有的进程使用完信号量集，使用`semctl() IPC_RMID`删除集合。


## API

### 创建或打开一个信号量

```c
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
int semget(key_t key, int nsems, int semflg);
// Returns semaphore set identifier on success, or –1 on error
```

如果是使用`semget()`来创建一个新的信号量集合，则`nsems`指定了集合中信号量数目，若是获取一个已存在的信号量集合，则`nsems`要小于等于集合大小。不能改变一个已存在的信号量集合的大小。

### 信号量控制

```c
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
int semctl(int semid, int semnum, int cmd, ... /* union semun arg */);
// Returns nonnegative integer on success (see text); returns –1 on error
```

参数`semid`是信号量的标识符，`semmum`指定当前操作的为信号量集中的第几个信号量。`cmd`为指定的操作, `arg`是一个如下的union, 根据`cmd`的不同，用作不同用途。

```cpp
#ifndef SEMUN_H
#define SEMUN_H /* Prevent accidental double inclusion */
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
union semun { /* Used in calls to semctl() */
    int val;
    struct semid_ds * buf;
    unsigned short * array;
#if defined(__linux__)
    struct seminfo * __buf;
#endif
};
#endif
```

`cmd`可取的值：
- IRC_RMID : 删除信号量,和相关的 semid_ds数据结构（这里面记录信号量的一些权限啊、操作时间等）
- IPC_STAT : 拷贝semid_ds到arg.buf中
- IPC_SET  : 使用arg.buf中的值设置semid_ds
- GETVAL   : 返回semid指定的信号量集中的第semnum个信号量的值。
- SETVAL   : 使用arg.val初始化对应的信号量值
- GETALL   : 获取所有的信号量，放到arg.array中
- SETALL   : 使用arg.array中的值初始化所有信号量
- GETPID   : 返回最后一个操作信号量的进程的进程号
- GETNCNT  : 返回当前在等待信号量增加的进程数
- GETZCNT  : 返回当前在等待信号量变为0的进程数 ? 啥情况会去等待为0？

前面提到的semid_ds数据结构如下，每个信号量都有一个对应的此数据结构：

```c
struct semid_ds {
    struct ipc_perm sem_perm; /* Ownership and permissions */
    time_t sem_otime; /* Time of last semop() */
    time_t sem_ctime; /* Time of last change */
    unsigned long sem_nsems; /* Number of semaphores in set */
};
```

